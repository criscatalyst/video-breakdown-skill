---
name: video-breakdown
description: Break a video down into its VISUALS and its AUDIO so you can study exactly what happens on screen and what is said. Downloads the video with yt-dlp, splits the audio track, samples timestamped frames + contact-sheet grids with ffmpeg, and transcribes the audio with local Whisper. Trigger when the user pastes a video URL (Instagram reel, TikTok, YouTube / YT Short, X/Twitter, Vimeo, …) and asks to "analyze the visuals", "break this down", "what happens in this video", "shot-by-shot", "frame by frame", "study this reel", or wants both what is shown and what is said.
---

# Video Breakdown

Reverse-engineer any video — both the **visual** track (what's on screen, shot by shot) and the **audio** track (the transcript) — using only local tools. No API key, nothing uploaded.

## When to use

- User pastes a video URL and wants to know **what happens on screen** (not just the words).
- Studying a competitor reel/short: hook visuals, cut pacing, text overlays, b-roll, CTA frame.
- Mapping a video shot-by-shot to recreate or repurpose it.
- Any time the user wants **visuals + transcript together**, aligned by timestamp.

> If the user only wants the spoken words, the lighter `transcribe` skill is enough. Use this one when the **visuals matter**.

## Setup (run once, only if needed)

The script needs three tools. Check first:

```bash
command -v yt-dlp ffmpeg whisper
```

If any is missing, tell the user and offer to install (this changes their system — get a yes first):

```bash
brew install yt-dlp ffmpeg openai-whisper
```

(Requires [Homebrew](https://brew.sh).) Make the script executable on first use:

```bash
chmod +x ~/.claude/skills/video-breakdown/breakdown.sh
```

## How to run

```bash
~/.claude/skills/video-breakdown/breakdown.sh "<URL>" [fps] [whisper_model] [language] [outdir]
```

| Arg | Default | Notes |
|---|---|---|
| `fps` | `1` | Frames sampled per second. Use `0.5` for long videos, `2` for fast-cut reels. |
| `whisper_model` | `small` | `medium` for accents/noise, `tiny`/`base` for speed, `large-v3` for best quality. |
| `language` | auto | Pass `en` / `it` / … if auto-detect mis-fires on short clips. |
| `outdir` | `./video-breakdown-<timestamp>` | Where everything lands. |

**Pick `fps` before running.** Estimate the video length:
- Short reel / TikTok (≤60s): `1` (default) is good — ~30–60 frames.
- Fast-cut / heavy text-overlay reel: `2`.
- Long video (several minutes): `0.5` or lower, or you'll generate hundreds of frames. Warn the user about the wait first.

The script prints a summary to stdout when done.

## How to read the output and analyze

After the script finishes, work through the output dir in this order:

1. **`sheets/sheet_*.png` first — the visual overview.** Each sheet is a 5×6 grid of frames with the timestamp burned into each thumbnail. Read these images to get the whole flow of the video at a glance: scene changes, on-screen text, b-roll vs talking head, where the cuts land.
2. **`frames/frame_*.jpg` for detail.** When a specific moment matters (the hook frame, a text overlay, the CTA), read the individual frame — it's higher resolution. Each frame has its timestamp top-left, so you can name the exact second.
3. **`transcript.srt` to align audio to visuals.** The `.srt` has timestamped segments. Match each spoken line to what's on screen at that timestamp. `transcript.txt` is the plain version if you just want the words.

Then synthesize for the user. Don't dump raw frames — deliver a **breakdown**:
- **Shot-by-shot timeline**: `0:00–0:02 hook frame (text: "…", talking head)`, `0:02–0:05 b-roll of …`, etc.
- **Hook analysis**: what's on screen + said in the first 3 seconds, and why it stops the scroll.
- **Visual devices**: text overlays, captions style, cut rhythm, zoom/punch-ins, b-roll ratio.
- **Structure beats**: hook → build-up → value → CTA, with the timestamp of each.
- **CTA frame**: what the final frame shows and says.

Ask the user what they're after (recreate it? steal the hook? study pacing?) and tailor the depth.

## Failure modes

- **Private / login-walled** (IG, some TikTok) → yt-dlp fails at step 1. Offer to retry by editing the script's yt-dlp call to add `--cookies-from-browser safari` (or `chrome`).
- **No audio track** → step 2 reports it; transcription is skipped, visual analysis still works.
- **Wrong language in transcript** → re-run with an explicit `language` arg.
- **Hundreds of frames / huge sheets** → fps was too high for a long video. Re-run with a lower `fps`.
- **No burned-in timestamp on frames** → no TTF font was found on the system; frame N is at ≈ N/fps seconds. Note this when citing times.

## Stack

- `yt-dlp` — downloads the video from almost any site
- `ffmpeg` — splits the audio, samples frames, builds contact sheets, burns timestamps
- `openai-whisper` — local speech-to-text (timestamped `.srt`)

Everything runs locally. Zero cost. First Whisper run downloads model weights (~500MB `small`), cached in `~/.cache/whisper/`.
