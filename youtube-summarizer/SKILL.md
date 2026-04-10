---
name: youtube-summarizer
description: Use when the user shares a YouTube URL or asks to summarize, extract key points from, or explain the content of a YouTube video, talk, tutorial, interview, podcast, or commentary.
---

# YouTube Summarizer

## Overview

Turn a YouTube video's spoken content into a terse, high-coverage text summary the user can read instead of watch. Prefer YouTube-provided subtitles or captions. Do not download the full media. Preserve chronology and substance; if captions are partial, low-quality, translated, or miss visual-only information, say so.

## When to Use

- User pastes a YouTube URL
- User asks for a summary, key points, workflow extraction, or "what does this video say?" for YouTube content
- Best for talks, tutorials, commentary, interviews, and video podcasts where spoken content carries most of the value

Do not use when:

- The user wants a verbatim transcript rather than a summary
- The content is primarily visual and captions alone cannot carry the meaning
- The URL is not YouTube

## Workflow

1. Extract and validate the YouTube URL.
2. Create an isolated temp directory for this run.

```bash
tmpdir="$(mktemp -d /tmp/youtube-summarizer.XXXXXX)"
trap 'rm -rf "$tmpdir"' EXIT
```

3. Fetch metadata in a structured form. Prefer structured output over scraping human-readable logs.

```bash
yt-dlp -J --skip-download "$url" > "$tmpdir/metadata.json"
```

Use this metadata for at least `title`, `duration` or `duration_string`, plus `subtitles` and `automatic_captions`. Read only the fields needed for the summary and subtitle selection; do not stream the full metadata blob into the model.

4. Download captions into `tmpdir`. Prefer uploaded subtitles and fall back to auto-generated captions. Prefer English when available. Request VTT output so parsing uses one format.

```bash
yt-dlp --skip-download --write-subs --write-auto-subs --sub-langs "en.*,en" --convert-subs vtt --output "$tmpdir/%(id)s.%(ext)s" "$url"
```

5. Select the best subtitle file. Do not assume yt-dlp will emit only one subtitle file.

- Use `metadata.json` to distinguish `subtitles` from `automatic_captions`
- Prefer uploaded subtitles from `subtitles` over auto-generated subtitles from `automatic_captions`
- Prefer English over other languages
- Ignore live chat and unrelated sidecar files
- If multiple candidates remain, choose the most complete VTT track for the selected subtitle source and language, usually the one with the most substantive text

6. If no English subtitles exist but another subtitle language does, inspect the metadata for available subtitle languages, choose the best fallback language, and run a second subtitle download pass for that language.

```bash
yt-dlp --skip-download --write-subs --write-auto-subs --sub-langs "$fallback_lang" --convert-subs vtt --output "$tmpdir/%(id)s.%(ext)s" "$url"
```

Summarize that fallback-language transcript in the chat language and state which subtitle language was used.
7. Read the full subtitle file. Never summarize only the first chunk by accident.

- If the file is larger than one tool read, read it in slices until the whole transcript has been processed
- It is acceptable to preprocess the VTT into a cleaned text file first, then summarize that cleaned text

8. Normalize the transcript before summarizing. See `Transcript Normalization`.
9. If the cleaned transcript is too large for one-pass summarization, summarize it chunk by chunk in chronological order, then compress those chunk summaries into the final answer. Never skip later sections.
10. Produce the final answer using `Report Structure`.
11. Delete `tmpdir` on every exit path. Register cleanup immediately after `mktemp`; do not rely only on a final best-effort cleanup command.

```bash
rm -rf "$tmpdir"
```

## Transcript Normalization

For WebVTT input:

- Ignore `WEBVTT`, blank lines, and non-content blocks such as `NOTE`, `STYLE`, and `REGION`
- Strip timestamps and cue settings
- Strip inline markup such as `<c>`, `<i>`, `<b>`, and similar tags
- Decode HTML entities if present
- Merge multiline cues into continuous spoken text
- De-duplicate rolling captions and overlapping repeated lines; keep the fuller later text when one cue extends the previous one
- Preserve chronological order

If yt-dlp produces a non-VTT subtitle file anyway, either convert it to VTT first or parse its real structure correctly. Do not assume SRT-style numbered blocks unless the file actually uses them.

## Report Structure

Always use this shape:

```text
[Video Title]
[Duration]

[1-2 sentence synopsis of the full arc of the video]

Key points:
- [point 1]
- [point 2]
- [point N]
```

## Summary Rules

- The synopsis is not a topic gloss; it should capture the full arc of the video in compressed form
- The bullet list should cover substantially all substantive information from the video in chronological order
- Each bullet should carry one discrete claim, instruction, step, argument, example, number, or decision
- Include concrete details: names, metrics, commands, package names, feature names, trade-offs, and notable examples
- Compress repetition aggressively, but do not drop novel information
- If the video is a tutorial, preserve the steps and prerequisites
- If the video is commentary or an interview, preserve the main arguments and attribute notable views to the speaker when clear
- For long videos, it is fine to insert short plain-text section labels above related bullets
- Use timestamps only when a visual detail is essential to understanding and the captions alone are not enough
- If visuals appear to matter but captions do not fully describe them, say so briefly instead of inventing details
- If captions are partial, auto-generated, translated, or noisy, disclose that in one short note before `Key points:`

Quality bar: the user should be able to skip watching the video and still grasp the substance.

## Error Handling

| Condition | Response |
|-----------|----------|
| Invalid URL | "That doesn't look like a YouTube URL." |
| `yt-dlp` missing | "This skill requires `yt-dlp` to be installed and available on PATH." |
| Video unavailable or private | "Couldn't access this video (unavailable or private)." |
| Age-restricted or auth-restricted video | "This video needs authenticated access or browser cookies before captions can be fetched." |
| Network or rate-limit failure | "Couldn't fetch video metadata or captions due to a network or rate-limit issue." |
| No subtitles or captions in any language | "No subtitles or captions are available for this video." |
| Live stream or premiere without usable captions | "This video does not have usable finalized captions yet." |
| Partial caption parse | Summarize only the successfully extracted text and say that some details may be missing |

If parsing fails completely, stop with a clear failure message instead of improvising.

## Out of Scope

- Whisper or audio transcription fallback
- Saving summaries to files by default
- Batch processing multiple videos
- User-configurable summary length
