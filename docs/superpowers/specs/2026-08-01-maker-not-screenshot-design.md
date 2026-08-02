# Maker Evidence → Not Screenshot (v8)

## Problem

Organized names like `screenshot_20231029_150334_sony_171c.jpg` are self-contradictory: the pipeline classified the file as a screenshot, then embedded EXIF Make (`sony`) into the filename. Real camera photos with Make but no GPS were misrouted to `screenshots/`.

## Decision

**Approach B:** Treat EXIF Make **or** a whitelisted source token in an archived filename as proof the file is a normal photo, not a screenshot. Fix forward classification and migrate existing misclassified files under `screenshots/`.

## Classification (images, v8)

```
0. Camera filename whitelist → normal
1. Maker evidence (EXIF Make OR archived filename source ∈ SOURCE_WHITELIST) → normal
2. Filename contains "screenshot" → screenshot
3. No EXIF GPS → screenshot
4. Else → normal
```

Videos unchanged.

## Stock cleanup

CLI: `rename_organize.py --work … --fix-maker-screenshots [--dry-run]`

- Scan `screenshots/` media files
- If maker evidence → `reclassify_paths(..., 'to_normal')`
- Print summary (moved / skipped / errors)

## Non-goals

- No change to video recording rules
- No automatic run during inbox organize (explicit flag only)
- No commit of design/plan required before implementation
