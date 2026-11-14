---
name: kitty-image-preview
description: Use when showing screenshots or images in a terminal, especially Rex, or when a terminal preview is blank, blurry, pixelated, or rendered as Unicode blocks.
---

# Kitty image preview

Show the original image using native terminal graphics. Prefer installed Showme with an explicit Kitty backend in Rex. Kitty's graphics protocol does not require the Kitty application to be installed.

## Choose and verify the surface

1. Locate the image and renderer: `command -v showme`, `showme --version`, and `showme --help`. Preserve the source file. Use an absolute image path; quote paths with spaces.
2. For Rex, read the `rex` skill or `rex skill show`. Check `rex whoami --json` and inspect the current session. Inherited block IDs can be stale. Keep a session name that already describes the task.
3. Use a dedicated preview pane. Never type a rendering command into a pane running an agent or another foreground application. Create a pane with `rex run`; do not assume an earlier preview pane still exists.

## Measure the destination, then render

Always measure the **destination preview pane after it exists**, not the agent pane or the pre-split window. For a new Rex preview, start an idle shell so its actual layout can be measured before rendering:

```sh
rex run -s "$REX_SESSION" -b "$REX_BLOCK" \
  --split right --ratio 40 --label image-preview --keep-open --json -- /bin/zsh -f
```

Save the returned block ID as `preview_block`. Inspect the existing pane instead if one is suitable and idle:

```sh
rex block call -s "$REX_SESSION" -b "$preview_block" size '{}'
rex block call -s "$REX_SESSION" -b "$preview_block" process '{}'
```

`size` reports `columns`, `rows`, `cell_width_px`, and `cell_height_px`. If the pane is still acquiring its size, retry a few times with a short bounded delay. Never accept zero dimensions or substitute the parent pane's dimensions. If there are multiple clients, use the dimensions of the client the user is viewing; the response's owner/desired fields help identify mismatches.

Read image dimensions without modifying the original. For example, macOS provides:

```sh
sips -g pixelWidth -g pixelHeight '/absolute/path/image.png'
```

For images with EXIF rotation, use orientation-corrected dimensions (swap width/height for a 90/270 degree orientation); Pillow's `ImageOps.exif_transpose` can determine them in memory. PNG screenshots normally need no rotation adjustment.

Calculate the fit from measured values:

```python
# W, H: oriented image pixels; C, R: destination columns and rows
# cw, ch: measured cell width and height in pixels
import math
available_cols, available_rows = C - 2, R - 3
if min(W, H, cw, ch, available_cols, available_rows) <= 0:
    raise ValueError("Image or pane dimensions are unavailable or too small")
scale = min(available_cols * cw / W, available_rows * ch / H, 1.0)
cols = max(1, min(available_cols, math.floor(W * scale / cw)))
rows = max(1, min(available_rows, math.floor(H * scale / ch)))
geometry = f"{cols}x{rows}"
```

This reserves two columns and three rows, fits both axes, and avoids enlarging small source images. Cell rounding introduces a small aspect-ratio error; tiny images may be limited to one cell. Both image and terminal pixels must use consistent units. If measurements are reported in different coordinate units, fit is still possible using the cell aspect ratio, but do not claim exact physical 1:1 scale.

Render using the **calculated** geometry:

```sh
showme --backend kitty --geometry "$geometry" --clear --center --quiet '/absolute/path/image.png'
```

In Rex, send the fully shell-quoted command to the confirmed idle `preview_block` with `rex send`, followed by `rex send-key ... Enter`. Use safe shell quoting such as Python `shlex.join` when constructing arguments; pass the resulting command as a single subprocess argument, never interpolate paths into shell source. Do not send it into the agent pane. Resolve the absolute Showme executable if the preview shell has a different PATH.

Outside Rex, obtain the **target terminal's** rows, columns, and pixel dimensions through `TIOCGWINSZ` on its `/dev/tty`; run the query inside that terminal. If pixel dimensions are missing, use a supported cell-size query or clearly label an estimated cell aspect ratio. Do not query the agent's unrelated tool PTY or silently assume 80x24.

`--clear` is appropriate in the dedicated preview pane. Avoid `--alternate-screen`, which restores the previous screen on exit. Do not pass `--fit-width` or `--fit-height` for an all-visible preview.

### Readability and resizing

A whole image fitting on screen does not guarantee legible UI text. Report severe downscaling; for a detailed screenshot, prefer a larger preview window when it materially increases the available image area. A wider pane will not help when height is the limiting dimension.

To maximize a Rex preview, use `rex block zoom -s "$REX_SESSION" -b "$preview_block" --on`. Then re-query its size, recompute geometry, and rerender. Use `--off` to restore the split and measure/rerender again. Creating a separate window is another option (omit the split flags from `rex run`). Keep changes scoped to the preview.

Repeat measurement after every pane resize, font-size change, or zoom. Showme renders static images once and exits; this skill does not provide a continuous resize watcher, drag-to-pan, or interactive image zoom. Do not promise those controls. If the full image remains unreadable, use an interactive viewer or an explicitly requested detail view.

## Why force the backend?

Showme 0.1.1 supports `--backend kitty` and `--backend iterm2`; Sixel requires a build with that feature. Its automatic selection uses terminal environment hints and can fall back to Unicode. The inspected build does not recognize Rex by name. Unicode half/quarter blocks are unsuitable for reading fine UI text. Do not try to fix that by adjusting `--pixelation`.

The inspected 0.1.1 Kitty backend assumes a 0.5 cell aspect ratio when automatically fitting; with both geometry dimensions supplied it uses them directly. Therefore calculate proportional cell dimensions yourself instead of passing the whole pane as geometry. Check the installed version before assuming its detection or options remain the same. Do not globally spoof TERM or set KITTY_WINDOW_ID to influence detection.

## Verify honestly

- When rendering inside the persistent preview shell, append an explicit status marker to the safely quoted command: `; preview_status=$?; printf '\nPreview renderer exit: %s\n' "$preview_status"`. Read that marker from `rex capture` after rendering. The block's `process` result describes the shell and does not by itself establish Showme's exit status. For a block running Showme directly, its process exit result is sufficient. Report errors before claiming success.
- Rex text capture does not include image pixels. An empty text capture does not establish that the image is blank.
- Showme's Kitty output uses `q=2`, suppressing protocol replies. Exit code zero establishes that the renderer finished, not that the user saw correct pixels. Use a screenshot when available, or the user's feedback, for visual confirmation.
- To diagnose the selected encoding without dumping binary data into chat, use `showme --backend kitty --output /tmp/preview-output.ansi IMAGE`. Inspect the file for `ESC_G` packets and `f=100` (PNG); do not print the base64 body. This verifies encoding, not display.

## Fallbacks

If Showme is missing but `kitten` is installed, inspect `kitten icat --help` and render in the same dedicated terminal. Do not install the full Kitty app just to use the protocol. If neither renderer is available, use an available native image viewer or explain the missing renderer before installing anything.

If a terminal rejects Kitty graphics, use a backend it actually supports. Do not silently substitute Unicode for a requested sharp screenshot preview. Preserve the original image and report the display limitation.
