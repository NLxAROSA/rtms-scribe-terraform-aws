# Real-Time Scribe Pipeline (design)

An additive extension to [this repo](https://github.com/zoom/rtms-terraform-aws) that replaces the worker's default `on_transcript_data` path with **rolling audio segments routed through [Zoom Scribe Fast Mode](https://developers.zoom.us/docs/ai-services/scribe/fast-mode/)**. Transcripts arrive continuously during the meeting (~one every few seconds) instead of only at meeting end.

This document describes the architecture only — see the main [README](https://github.com/zoom/rtms-terraform-aws/README.md) for the base infrastructure.

---

## Architecture

```
                            Zoom Cloud
                                │
              ┌─────────────────┼─────────────────────┐
              │ webhooks        │ RTMS audio          │ Scribe Fast Mode
              │ (HTTPS)         │ (continuous PCM)    │ (per-segment POST)
              ▼                 ▼                     ▲
           ALB :443  ────►   ECS worker               │
                                 │                    │
                                 │ on_audio_data      │
                                 │ buffer per         │
                                 │ meeting_uuid       │
                                 ▼                    │
                          ┌──────────────┐            │
                          │ rotate every │            │
                          │ N seconds    │ ── async ──┘
                          │ (e.g. 5s)    │   per segment
                          └──────┬───────┘
                                 │
                                 ▼  WAV
                  S3 recordings/<uuid>/<seq>.wav
                                 │  presigned GET URL
                                 │  (Scribe fetches it)
                                 ▼
                       Scribe POST /transcribe
                                 │  JSON
                                 ▼
                  S3 transcripts/<uuid>/<seq>.json
                       (one object per segment;
                        full transcript = concat seq → seq+1 → …)
```

## Timeline

```
  audio in   ─────────────────────────────────────────►  meeting ends

  buffer     [ seg 1 ][ seg 2 ][ seg 3 ][ seg 4 ][ seg 5 ]
                │       │       │       │       │
                ▼       ▼       ▼       ▼       ▼     ← fire async, in parallel,
              POST    POST    POST    POST    POST       via the executor pool
                │       │       │       │       │
                ▼       ▼       ▼       ▼       ▼
              JSON    JSON    JSON    JSON    JSON    ← land in S3 a few seconds
                                                         after each segment closes
```

---

## What changes vs. the base repo

| Layer | Change |
|---|---|
| **Terraform** | One new variable (`zm_scribe_s2s_oauth_secret_arn`) + one IAM expansion (`s3:GetObject` on `recordings/*` so the worker can sign presigned URLs). **No new AWS resources.** |
| **Secrets Manager** | One additional entry: Server-to-Server OAuth credentials as JSON `{account_id, client_id, client_secret}`. The Zoom Marketplace app needs the `aiservices:scribe` scope. |
| **S3 layout** | Existing bucket gains two prefixes: `recordings/<uuid>/<seq>.wav` and `transcripts/<uuid>/<seq>.json`. |
| **Worker code** | Subscribes to `on_audio_data` instead of `on_transcript_data`. Per-meeting `threading.Timer` rotates the buffer every `SCRIBE_SEGMENT_SECONDS`, encodes WAV, mints a presigned URL, and submits a Scribe call to the existing `ThreadPoolExecutor`. On `rtms_stopped`/`on_leave`, flushes the partial tail as the final segment. |
| **Outbound HTTPS** | Token mint at `zoom.us/oauth/token` (cached ~55 min) + Scribe POST per segment. Both exit via the existing IGW. |

What stays unchanged: VPC, ALB, ECS, ACM, Route 53, CloudWatch dashboards/alarms, autoscaling.

## Trade-offs

- **Time-to-first-transcript** drops from "meeting length" → "~segment length + Scribe latency" (roughly 5–10 seconds at 5 s segments).
- **Word boundaries at segment edges** can split a word or sentence — Scribe has no cross-segment context. Two mitigations: (a) accept it (typical for live captioning); (b) overlap segments by ~0.5 s and dedupe downstream. (a) is the simpler default.
- **Fixed-window segmentation** is dumb but simple. No VAD; segments may start mid-word but at 5 s windows this is rare (most words are <0.5 s).
- **Spot reclaim drops only the in-flight segment's audio** (≤ N seconds), not the whole meeting.
- **S3 PUT volume.** A 1 hr meeting at 5 s segments produces ~720 WAVs + ~720 JSONs. Storage is trivial (~115 MB of audio); PUT cost is ~$0.004 per meeting.
- **Scribe API call volume.** ~720 calls per 1 hr meeting. Worth checking against per-account quotas before going wide.

## Configuration

New env vars (defaults shown):

| Var | Default | Purpose |
|---|---|---|
| `SCRIBE_ENABLED` | `false` | Master switch. When false, worker uses the original `on_transcript_data` path. |
| `ZM_SCRIBE_OAUTH_JSON` | — | JSON blob from Secrets Manager: `{"account_id":"...","client_id":"...","client_secret":"..."}`. |
| `SCRIBE_SEGMENT_SECONDS` | `5` | Segment length. 5 s = snappy captions; 15 s = fewer PUTs / Scribe calls. |
| `SCRIBE_BASE_URL` | `https://api.zoom.us` | Scribe API base. |
| `SCRIBE_LANGUAGE` | `en-US` | Per-segment language hint. |
| `SCRIBE_SAMPLE_RATE` | `16000` | WAV sample rate; matches RTMS default 16 kHz mono. |

Recommended tuning when enabled:

- Bump `CALLBACK_EXECUTOR_WORKERS` from 16 → 32 so Scribe calls don't queue under load.
- Set `spot_weight = 0` (or mix with on-demand) — Spot reclaim risk grows with continuous in-flight Scribe calls.
- Add a lifecycle rule on `recordings/*` (e.g. 3-day expiry) if the audio archive isn't needed.

## Stitching transcripts

Each segment lands as its own S3 object. To reconstruct the meeting transcript:

```bash
aws s3 cp s3://<bucket>/transcripts/<uuid>/ ./out --recursive
# Filenames are zero-padded seq numbers, so a lexical sort = chronological order:
ls out/*.json | sort | xargs jq -r '.result.text_display' | tr '\n' ' '
```

For consumers that want a single rolled-up artifact, add a small "stitcher" — either a Lambda triggered by `meeting.rtms_stopped` that reads all segments and writes `transcripts/<uuid>.jsonl`, or an S3 Event → Lambda that appends each segment to a running JSONL as it arrives.

---

## Status

Design only — not implemented in this repo yet. The hooks (one variable, one IAM expansion, one task-definition conditional, ~150 lines of worker code) are scoped but unmerged. The default `worker_image` is the pre-Scribe build; adopting this design requires rebuilding via `scripts/build-and-push-worker.sh` and pointing `worker_image` at the new ECR tag.
