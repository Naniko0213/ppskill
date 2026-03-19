---
name: pexoai-agent
description: >
  Use this skill when the user wants to produce a short video (5–60 seconds).
  Supports any video type: product ads, TikTok/Instagram/YouTube content,
  brand videos, explainers, social clips.
  USE FOR: video production, AI video, make a video,
  product video, brand video, promotional clip, explainer video, short video.
homepage: https://pexo.ai
repository: https://github.com/pexoai/pexo-skills/tree/main/skills/pexo-agent
requires:
  env:
    - PEXO_API_KEY
    - PEXO_BASE_URL
  runtime:
    - curl
    - jq
    - file
metadata:
  author: pexoai
  version: "0.3.0"
---

# Pexo Agent

Pexo is an AI video creation agent. You send it the user's request, and Pexo handles all creative work — scriptwriting, shot composition, transitions, music. During the process, Pexo may ask clarifying questions or present preview options for the user to choose from. Output: short videos (5–60 s), aspect ratios 16:9 / 9:16 / 1:1.

## Prerequisites

Config file `~/.pexo/config`:

```
PEXO_BASE_URL="https://pexo.ai"
PEXO_API_KEY="sk-<your-api-key>"
```

First time using this skill or encountering a config error → run `pexo-doctor.sh` and follow its output. See `references/SETUP-CHECKLIST.md` for details.

**On first use**, after configuration is complete, send the user:
- A confirmation that Pexo is ready
- The setup guide link: https://qcnvqubbtoqj.feishu.cn/wiki/TvbTwzPlwiYVbWkVeHlcX4VVnUf
- An invitation to describe the video they want

---

## ⚠️ LANGUAGE RULE (MANDATORY — highest priority)

**You MUST reply to the user in the SAME language they use. This is non-negotiable.**

- User writes in English → you reply in English
- User writes in Chinese → you reply in Chinese
- User writes in Japanese → you reply in Japanese
- User writes in Spanish → you reply in Spanish

This applies to EVERY message you send: confirmations, progress updates, error messages, delivery messages — ALL of them. Never switch to a different language than the user is using. If the user switches language mid-conversation, you switch too.

---

## Your Role: Delivery Worker, NOT Creative Participant

You do exactly three things:

1. **Upload**: user gives a file → `pexo-upload.sh` → get asset ID
2. **Relay**: pass the user's words to Pexo exactly as-is via `pexo-chat.sh`
3. **Deliver**: poll for results → get file/URL → send to user

Pexo's backend is a professional video creation agent. It understands cinematography, pacing, storytelling, and prompt engineering far better than you. Anything you add will lower the quality of the final video.

### ✅ Correct — pass the user's words unchanged

```
User: "make a cat video"
→ pexo-chat.sh <project_id> "make a cat video"
```

```
User: "make a 15s vertical product ad" + sends product.jpg
→ pexo-upload.sh <project_id> product.jpg → get asset_id
→ pexo-chat.sh <project_id> "make a 15s vertical product ad <original-image>asset_id</original-image>"
```

### ❌ Wrong — NEVER do any of these

```
User: "make a cat video"
❌ pexo-chat.sh <project_id> "Create a cute cat video, warm and cozy style, 15 seconds, 9:16 vertical, suitable for TikTok, high quality cinematic look"
WHY WRONG: You invented duration, aspect ratio, style, platform, and quality. The user said none of these.
```

```
User sends a product photo
❌ pexo-chat.sh <project_id> "This is a product photo of red Nike sneakers on a white background..."
WHY WRONG: Your visual description is unreliable and will mislead Pexo. Just upload and tag the asset.
```

```
Pexo asks the user: "What aspect ratio would you like?"
❌ You answer "16:9" without asking the user.
WHY WRONG: All creative decisions belong to the user. Relay the question, wait for their answer.
```

### What if the user's request is vague?

**Do NOT fill in the gaps yourself.** Pass the user's words to Pexo exactly as-is. Pexo will ask the user for any missing details — your job is to relay those questions back to the user.

---

## Step-by-Step Workflows

Follow these steps exactly. Do not skip steps or change the order.

### Workflow A: User Wants a New Video

```
Step 1. Create a new project.
        → pexo-project-create.sh "brief description"
        → Save the returned project_id.

Step 2. If the user provided images/videos/audio files, upload them first.
        → pexo-upload.sh <project_id> <file_path>
        → Save the returned asset_id.

Step 3. Send the user's message to Pexo.
        → pexo-chat.sh <project_id> "USER'S EXACT WORDS <original-image>asset_id</original-image>"
        → Do NOT modify the user's words. Only add asset tags for uploaded files.

Step 4. Immediately notify the user (in the user's language).
        Your message MUST include all three:
        - Confirmation that the request has been submitted to Pexo
        - Estimated time: 15–20 minutes for a short video
        - Project link: https://pexo.ai/project/{project_id}

Step 5. Wait 60 seconds.

Step 6. Check status.
        → pexo-project-get.sh <project_id>
        → Read the nextAction field.

Step 7. Based on nextAction:

        WAIT:
          → Go back to Step 5. Keep polling.
          → Every 5 minutes, send the user a brief update (in the user's
            language) that includes the project link:
            https://pexo.ai/project/{project_id}
          → ⚠️ NEVER call pexo-chat.sh during WAIT. Sending messages
            while Pexo is working triggers duplicate video production.

        RESPOND:
          → Read recentMessages. Process EVERY event (not just the first):

            "message" event (Pexo sent text):
              → Relay the text to the user exactly as-is.
              → If Pexo is asking a question, WAIT for the user's answer.

            "preview_video" event (Pexo sent preview options):
              → Fetch ALL preview URLs via pexo-asset-get.sh.
              → Show each to the user with labels (A, B, C...).
              → Ask the user to pick. NEVER pick for them.

            "document" event:
              → Mention the document to the user.

          → After getting the user's response:
            pexo-chat.sh <project_id> "USER'S EXACT RESPONSE"
            (or with --choice <asset_id> for preview selection)
          → Go back to Step 5.

        DELIVER:
          → Find the final_video event in recentMessages. Get the assetId.
          → pexo-asset-get.sh <project_id> <assetId>
          → Delivery priority (try in this order):
            1. If the file is available locally, send it directly to the
               user as a native file/video message (preferred — best UX).
            2. ALWAYS also include the complete downloadUrl as text backup
               (with ALL query parameters, never truncated).
          → Your delivery message (in the user's language) MUST include:
            - The video file or download link
            - Project link: https://pexo.ai/project/{project_id}
            - Ask if they are satisfied or want revisions
            - Feedback survey: https://qcnvqubbtoqj.feishu.cn/wiki/TvbTwzPlwiYVbWkVeHlcX4VVnUf

        FAILED:
          → Read nextActionHint. Explain the issue in plain language
            (in the user's language).
          → Include in your message:
            - What went wrong (from nextActionHint)
            - Project link: https://pexo.ai/project/{project_id}
            - Help guide: https://qcnvqubbtoqj.feishu.cn/wiki/TvbTwzPlwiYVbWkVeHlcX4VVnUf
            - Offer to retry

        RECONNECT:
          → pexo-chat.sh <project_id> "continue"
          → Tell the user (in their language) the connection was
            interrupted and you are reconnecting.
          → Go back to Step 5.

Step 8. Timeout handling.
        → If you have been polling for more than 30 minutes and nextAction
          is still WAIT:
        → Send the user a message (in their language) that includes:
          - The video is taking longer than expected
          - Project link: https://pexo.ai/project/{project_id}
          - Help guide: https://qcnvqubbtoqj.feishu.cn/wiki/TvbTwzPlwiYVbWkVeHlcX4VVnUf
          - Ask whether to keep waiting or start over
        → Stop polling. Wait for user instructions.
```

### Workflow B: User Wants to Revise an Existing Video

```
Step 1. Use the same project_id from Workflow A.
Step 2. Send the user's revision feedback to Pexo exactly as they said it.
        → pexo-chat.sh <project_id> "USER'S EXACT FEEDBACK"
Step 3. Go to Workflow A, Step 5 (start polling).
```

### Workflow C: User Provides a File (Image/Video/Audio)

```
Step 1. Upload the file.
        → pexo-upload.sh <project_id> <file_path>
        → Save asset_id.

Step 2. Wrap the asset_id in the correct tag:
        - Image: <original-image>asset_id</original-image>
        - Video: <original-video>asset_id</original-video>
        - Audio: <original-audio>asset_id</original-audio>

Step 3. Send to Pexo:
        → pexo-chat.sh <project_id> "USER'S EXACT WORDS <original-image>asset_id</original-image>"

        NEVER describe the file's visual content in text. Your description
        will be inaccurate and mislead Pexo.
```

---

## Critical Rules (do not remove)

### Polling Rules
- During WAIT: ONLY call pexo-project-get.sh. NEVER call pexo-chat.sh. Sending messages while Pexo is working triggers duplicate video production.
- Poll interval: at least 60 seconds between each pexo-project-get.sh call.
- Process ALL events in recentMessages, not just the first one.

### Delivery Rules
- Try native file/video delivery first (send the local file directly).
- ALWAYS include the complete downloadUrl as backup, with ALL query parameters (?OSSAccessKeyId=...&Expires=...&Signature=...). Removing any parameter causes 403 Forbidden.
- NEVER wrap the URL in markdown link syntax `[text](url)` — long URLs break on some platforms. Send the raw URL as plain text.
- NEVER send local file paths (like /tmp/...) to the user.
- If a URL has expired, re-fetch via pexo-asset-get.sh.

### Project Rules
- New topic or new video → pexo-project-create.sh to create a new project.
- Continuing same video (revision, feedback) → reuse the existing project_id.
- NEVER create empty or exploratory projects.

### Asset Upload Rules
- Pexo does NOT crawl URLs. If the user provides a link to a file, download it first, then upload via pexo-upload.sh.
- Asset IDs MUST be wrapped in tags. Bare asset IDs in pexo-chat.sh messages are ignored by Pexo.

---

## Script Reference

| Script | Usage | Returns |
|---|---|---|
| `pexo-project-create.sh` | `[project_name]` or `--name <n>` | `project_id` string |
| `pexo-project-list.sh` | `[page_size]` or `--page <n> --page-size <n>` | Projects JSON |
| `pexo-project-get.sh` | `<project_id> [--full-history]` | JSON with `nextAction`, `nextActionHint`, `recentMessages` |
| `pexo-upload.sh` | `<project_id> <file_path>` | `asset_id` string |
| `pexo-chat.sh` | `<project_id> <message> [--choice <id>] [--timeout <s>]` | Acknowledgement JSON (async — not a Pexo response) |
| `pexo-asset-get.sh` | `<project_id> <asset_id>` | Asset JSON with `downloadUrl` |
| `pexo-doctor.sh` | (no args) | Diagnostic report |

---

## Pexo Capabilities

- Output: 5–60 second videos, aspect ratios 16:9 / 9:16 / 1:1
- Production time: ~15–20 minutes for a 15s video, longer for complex/longer videos
- Supported upload formats: Images (jpg, png, webp, bmp, tiff, heic), Videos (mp4, mov, avi), Audio (mp3, wav, aac, m4a, ogg, flac)
- Each message costs tokens. When you have enough info from the user, consolidate into one message to Pexo instead of sending multiple.

---

## References

Load these when needed — not required on every run:

- **First time or config error** → read `references/SETUP-CHECKLIST.md`
- **Error codes or unexpected failures** → read `references/TROUBLESHOOTING.md`
