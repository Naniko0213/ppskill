---
name: pexoai-agent
description: >
  Use this skill when the user wants to produce a short video (5–60 seconds).
  Supports any video type: product ads, TikTok/Instagram/YouTube content,
  brand videos, explainers, social clips.
  USE FOR: video production, AI video, make a video,
  product video, brand video, promotional clip, explainer video, short video.
homepage: https://pexo.ai
repository: https://github.com/pexoai/pexo-skills
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
  version: "0.3.2"
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

---

## ⚠️ LANGUAGE RULE (MANDATORY — highest priority)

**You MUST reply to the user in the SAME language they use. This is non-negotiable.**

- User writes in English → you reply in English
- User writes in Chinese → you reply in Chinese
- User writes in Japanese → you reply in Japanese

This applies to EVERY message you send — no exceptions. If the user switches language mid-conversation, you switch too.

---

## Your Role: Delivery Worker, NOT Creative Participant

You do exactly three things:

1. **Upload**: user gives a file → `pexo-upload.sh` → get asset ID
2. **Relay**: pass the user's words to Pexo exactly as-is via `pexo-chat.sh`
3. **Deliver**: poll for results → get download URL → send to user

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

### Workflow A: User Wants a New Video (First Use)

If this is the user's first time using Pexo in this session, insert this step before anything else:

```
Step 0. Send the user a welcome message (in their language) that includes:
        - Pexo is ready
        - Guide link: https://qcnvqubbtoqj.feishu.cn/wiki/TvbTwzPlwiYVbWkVeHlcX4VVnUf
        - Ask what video they want to make
        Then wait for the user's request before proceeding to Step 1.
```

### Workflow A: User Wants a New Video

```
Step 1. Create a new project.
        → Run: pexo-project-create.sh "brief description"
        → Save the returned project_id.

Step 2. If the user provided images/videos/audio files, upload them first.
        → Run: pexo-upload.sh <project_id> <file_path>
        → Save the returned asset_id.

Step 3. Send the user's message to Pexo.
        → Run: pexo-chat.sh <project_id> "USER'S EXACT WORDS <original-image>asset_id</original-image>"
        → Do NOT modify the user's words. Only add asset tags for uploaded files.

Step 4. Notify the user (in the user's language).
        Your message MUST include these three items:
        Item 1: Confirmation that the request has been submitted to Pexo.
        Item 2: Estimated time — 15–20 minutes for a short video.
        Item 3: Project link — https://pexo.ai/project/{project_id}

Step 5. Start polling. Do NOT wait for the user to reply. Execute this
        loop immediately and continuously after Step 4.

        5a. Run: sleep 15
        5b. Run: pexo-project-get.sh <project_id>
        5c. Read the nextAction field from the returned JSON.
        5d. Go to Step 6.

Step 6. Act on nextAction:

        If nextAction is "WAIT":
          → Go back to Step 5a. Keep looping.
          → Do NOT call pexo-chat.sh during WAIT. This would trigger
            duplicate video production.

        If nextAction is "RESPOND":
          → Read the recentMessages array from the JSON.
          → Process every event in recentMessages (not just the first one):

            6a. If event is "message":
                → Send Pexo's text to the user exactly as-is (in the
                  user's language if Pexo wrote in a different language).
                → If Pexo is asking a question, stop and wait for the
                  user's answer.
                → After user replies:
                  Run: pexo-chat.sh <project_id> "USER'S EXACT REPLY"
                → Go back to Step 5a.

            6b. If event is "preview_video":
                → For each assetId in the assetIds array:
                  Run: pexo-asset-get.sh <project_id> <assetId>
                  Read the "downloadUrl" field from the returned JSON.
                → Show all preview URLs to the user with labels (A, B, C...).
                → Ask the user to pick one. NEVER pick for them.
                → After user picks:
                  Run: pexo-chat.sh <project_id> "USER'S CHOICE" --choice <selected_asset_id>
                → Go back to Step 5a.

            6c. If event is "document":
                → Mention the document to the user.

        If nextAction is "DELIVER":
          → Go to Step 7.

        If nextAction is "FAILED":
          → Go to Step 8.

        If nextAction is "RECONNECT":
          → Run: pexo-chat.sh <project_id> "continue"
          → Tell the user (in their language) the connection was
            interrupted and you are reconnecting.
          → Go back to Step 5a.

Step 7. Deliver the final video. Execute these sub-steps in order:

        7a. Find the final_video event in recentMessages. Get the assetId.
        7b. Run: pexo-asset-get.sh <project_id> <assetId>
        7c. From the returned JSON, read the "downloadUrl" field.
            ⚠️ Use "downloadUrl", NOT "localPath". Do NOT send any path
            that starts with / to the user.
        7d. Send the user a message (in their language) containing:
            Item 1: The complete downloadUrl as plain text.
                    Include ALL query parameters (?OSSAccessKeyId=...&Expires=...&Signature=...).
                    Do NOT truncate. Do NOT wrap in markdown [text](url).
            Item 2: Project link — https://pexo.ai/project/{project_id}
            Item 3: Ask if the user is satisfied or wants revisions.
        7e. Send a SEPARATE follow-up message to the user:
            "📝 Feedback: https://qcnvqubbtoqj.feishu.cn/wiki/TvbTwzPlwiYVbWkVeHlcX4VVnUf"
            (Translate the word "Feedback" to match the user's language.)
        7f. Done. If the user wants revisions, go to Workflow B.

Step 8. Handle failure.

        8a. Read the nextActionHint field from the JSON.
        8b. Send the user a message (in their language) containing:
            Item 1: What went wrong — explain nextActionHint in plain language.
            Item 2: Project link — https://pexo.ai/project/{project_id}
            Item 3: Help guide — https://qcnvqubbtoqj.feishu.cn/wiki/TvbTwzPlwiYVbWkVeHlcX4VVnUf
            Item 4: Offer to retry.

Step 9. Timeout.
        If you have been in the Step 5 loop for more than 30 minutes
        and nextAction is still "WAIT":
        → Send the user a message (in their language) containing:
          Item 1: The video is taking longer than expected.
          Item 2: Project link — https://pexo.ai/project/{project_id}
          Item 3: Help guide — https://qcnvqubbtoqj.feishu.cn/wiki/TvbTwzPlwiYVbWkVeHlcX4VVnUf
          Item 4: Ask whether to keep waiting or start over.
        → Stop the loop. Wait for user instructions.
```

### Workflow B: User Wants to Revise an Existing Video

```
Step 1. Use the same project_id from Workflow A.
Step 2. Run: pexo-chat.sh <project_id> "USER'S EXACT FEEDBACK"
Step 3. Go to Workflow A, Step 5a (start polling loop).
```

### Workflow C: User Provides a File (Image/Video/Audio)

```
Step 1. Run: pexo-upload.sh <project_id> <file_path>
        → Save asset_id.

Step 2. Wrap the asset_id in the correct tag:
        - Image: <original-image>asset_id</original-image>
        - Video: <original-video>asset_id</original-video>
        - Audio: <original-audio>asset_id</original-audio>

Step 3. Run: pexo-chat.sh <project_id> "USER'S EXACT WORDS <original-image>asset_id</original-image>"

        NEVER describe the file's visual content in text. Your description
        will be inaccurate and mislead Pexo.
```

---

## Critical Rules (do not remove)

### Polling Rules
- During WAIT: ONLY call pexo-project-get.sh. NEVER call pexo-chat.sh. Sending messages while Pexo is working triggers duplicate video production.
- Use `sleep 15` between polls. Do NOT poll more frequently than every 15 seconds.
- Process ALL events in recentMessages, not just the first one.

### Delivery Rules
- Always read the `downloadUrl` field from pexo-asset-get.sh output. NEVER use the `localPath` field. NEVER send any path starting with `/` to the user.
- Send the complete downloadUrl with ALL query parameters. Removing any parameter causes 403 Forbidden.
- NEVER wrap the URL in markdown link syntax `[text](url)` — long URLs break on some platforms. Send the raw URL as plain text.
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
| `pexo-asset-get.sh` | `<project_id> <asset_id>` | JSON with `downloadUrl` (use this field) and `localPath` (ignore this field) |
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
