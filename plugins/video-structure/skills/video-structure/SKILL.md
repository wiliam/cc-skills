---
name: video-structure
description: Extract structured notes with key theses, tools, people, and resources from video or audio content. MUST use this skill whenever the user shares a YouTube link (youtube.com, youtu.be) — regardless of how they phrase the request (e.g. "summarize", "what's in this video", "make notes", "pull out key points", "конспект", "тезисы", "разбери видео"). Also trigger for requests to structure existing transcripts, meeting recordings, subtitle files (.srt), or any "расшифровка"/"transcript" that needs theses and resource extraction. This skill adds verification, Q&A extraction, shared resources separation, and uncertainty marking that basic summarization misses.
---

# Video Structure Extraction

Extract structure and specifics from video subtitles or a transcript, produce a markdown document.

## Execution model

This skill has two stages: **download** (needs Bash for yt-dlp) and **process** (reading + writing).

**Recommended approach:** handle Phase 1 (download) yourself, then spawn Phase 2–5 (process) as a subagent on **sonnet**. Sonnet handles the skill well — comparable quality to opus at lower cost. It finds the same (or more) details, follows the template, and produces the verification section.

```
1. Download subtitles yourself (Phase 1) — you need Bash access
2. Spawn a general-purpose subagent with model=sonnet:
   - Pass: skill file path, transcript path, video metadata (title, URL, date, transcript filename)
   - Instruct: "Read the skill, skip Phase 1, follow Phases 2–5, save to _inbox/"
```

If the user explicitly asks for maximum depth or the video is unusually complex (3+ speakers, heavy technical content), use opus instead.

## Input

$ARGUMENTS

User may provide:
- A YouTube URL (with optional context: speaker, timestamp, focus)
- A path to an existing transcript/subtitle file (.srt, .txt, .md)
- A description of where to find the transcript (e.g. "it's in _inbox")

Parse $ARGUMENTS yourself — no formal flags.

---

## Phase 1: Get the text

### Path A: User provided a transcript file
If the input is a file path or the user says they already have a transcript/subtitles:
1. Find and read the file
2. Note the source format (SRT with timestamps / plain text / etc.)
3. Set subtitle type to "provided transcript" in metadata
4. Skip to Phase 2

### Path B: YouTube URL — download subtitles

The key principle: **minimize yt-dlp calls**. Each call is a separate YouTube request. 3+ calls in quick succession → 429 rate limit.

1. Check `yt-dlp`: `which yt-dlp` (install via `brew install yt-dlp` if missing)
2. **Single call — download everything at once:**
   ```
   yt-dlp --skip-download --write-sub --write-auto-sub --sub-lang "ru-orig,ru,en-orig,en" --sub-format srt --print title --print upload_date -o "_inbox/transcripts/%(title)s" <URL>
   ```
   This downloads subs, prints the title and upload date (YYYYMMDD). The `-orig` variants are auto-generated captions in the original language — some videos only have these. After download, rename the .srt file to: `YYYY-MM-DD-youtube-short-name.lang.srt` (e.g. `2026-02-03-youtube-demchog-balance.ru.srt`). Short name: 2-4 English words, lowercase, hyphens.
3. Check what files appeared in `_inbox/transcripts/`. If both `ru` and `en` downloaded, prefer `ru` for Russian-language videos
4. If no srt appeared: retry without language filter: `yt-dlp --skip-download --write-auto-sub --sub-format srt --print title -o "_inbox/transcripts/%(title)s" <URL>` — this grabs whatever is available
5. If still nothing: `sleep 30` then `yt-dlp --list-subs <URL>` to see what exists, retry with correct language code
6. If 429: **wait 30 seconds** and retry once. If persistent, ask user to provide a cookies file
7. Get upload date: `yt-dlp --print upload_date <URL>` (returns YYYYMMDD, convert to YYYY-MM-DD for frontmatter)
8. Note subtitle type (manual/auto), language, and transcript filename — all go into document frontmatter

### Troubleshooting yt-dlp
- **429 Too Many Requests**: caused by too many calls. Always try the single-call approach first. If you need a second call, add `sleep 30` before it
- **"No subtitles for requested languages"**: drop `--sub-lang` entirely to get whatever is available, or run `--list-subs` to see what exists
- **ffmpeg not found warning**: `--sub-format srt` requests SRT directly from YouTube (no conversion needed). This is fine
- **Last resort for persistent 429**: ask the user to export cookies: `yt-dlp --cookies-from-browser chrome` (will prompt for Keychain password)

## Phase 2: Full read

Goal: understand structure. Do NOT write the document yet.

1. If user specified a timestamp — find the range via grep `HH:MM:` in SRT
2. Read in 500-line chunks — the ENTIRE segment end to end
3. **Read everything first, then outline.** Writing while reading biases toward early content and loses late details
4. Draft an outline: topic blocks, transitions, approximate timestamps

## Phase 3: Targeted extraction

First-pass reading catches the gist but misses specific names, tools, books. Run grep across the **ENTIRE subtitle file** (not just the segment — speakers may reference things elsewhere):

- Each proper name from Phase 2 — as a separate query
- Books/sources: `книг|book|читал|paper|study`
- Theories: `теори|метод|фреймворк|approach|framework`
- Tools: specific names from Phase 2 + `github|figma|cursor|mcp|crm|api`
- Numbers: `процент|%|доллар|dollar|month|рубл`
- Recommendations: `попробуй|обязательно|must|should|never`

For each match — read ±20 lines of context. Record: what, why, how it's used.

## Phase 4: Assemble document

Save to `_inbox/` (or user-specified path). Use the output language matching the subtitle language.

### Template

```markdown
---
title: "[Original video/meeting title as-is]"
speakers: [comma-separated names]
host: [name, if applicable — omit field if no host]
source: [YouTube URL or other source]
date: [YYYY-MM-DD, get via: yt-dlp --print upload_date <URL>, convert YYYYMMDD → YYYY-MM-DD]
tags: [3-8 topic tags, comma-separated, lowercase]
transcript: "[[YYYY-MM-DD-source-short-name.lang.srt]]"
---

# [Topic — your editorial title, can differ from original]
**Timestamps:** XX:XX – YY:YY | **Subtitles:** manual/auto

## 1. [Block]
- Thesis with **specifics**
> Direct quote from subtitles when the phrasing is particularly precise

### Q&A highlights
- [Key question] → [Key answer/insight from the speaker]

...

---

## Shared resources
Resources that speakers explicitly shared or recommended — things the viewer can go use.

| Resource | Who shared | What it is | How to get it |
|----------|-----------|------------|---------------|

## All mentions

### Tools
| Tool | What it is | Context |
|------|-----------|---------|

### Books and theories
| Source | What it is | How it's used |
|--------|-----------|---------------|

### People
| Who | Context |
|-----|---------|

Only include people who are speakers, named experts, or referenced authors.
Do NOT list audience members who asked questions — their contributions belong in Q&A highlights.

---

## Verification

**Blocks:** N | **Tools:** N | **People:** N | **Books/theories:** N | **Shared resources:** N

### Found during verification pass
- [what was missed in first pass]

### Unclear terms
- [terms auto-subtitles may have distorted]

### Confidence level
[auto/manual subs, which names need manual verification]
```

The "Verification" section is mandatory. The document is incomplete without it.

The "Shared resources" table is for things viewers can actually use: GitHub repos, frameworks, courses, tools with install instructions. This separates actionable resources from passing mentions.

## Phase 5: Pre-save check

After assembling — one more grep round for non-obvious patterns:

```
[A-Z]{2,}                              # abbreviations
скормил|загрузил|закинул|залил          # what was fed to the system
стоит|нужно|важно|главн|ключев         # speaker emphasis
```

Any new match not in the document — add it. Update counters in Verification.

Check:
- Every block has specifics (not just abstractions)
- No orphan claims without context
- No duplicates across sections

---

## Principles

- **Don't fabricate.** If unclear — write `[unclear: ...]`
- **Quote.** Use `>` blockquotes for sharp speaker formulations
- **Q&A is valuable.** Questions often reveal practical details the talk itself missed. Summarize key Q&A exchanges after each speaker section
- **Separate shared vs mentioned.** A tool the speaker built and published on GitHub ≠ a tool they mentioned in passing. The "Shared resources" table is for things viewers can go use right now
- **Auto-subs lie:** names distorted, English terms transcribed phonetically, `>>` speaker changes approximate
- **Target:** 200–500 lines markdown. Specifics > completeness
- **Long videos (>2h):** split into parts, process each separately
