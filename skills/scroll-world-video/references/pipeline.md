# Pipeline: the run flow (Kubeez MCP + bash-3.2-safe local processing)

Generation happens through the **Kubeez MCP** tools (`mcp__kubeez__*`), which you call
directly. The MCP API only ingests **HTTPS URLs**, so it can't read your disk: generation
outputs are already URLs (they chain straight through), but any frame you extract with
ffmpeg is a local file that must be **uploaded** before it can seed the next clip. The bash
below does only what bash does (download, extract frames, encode); it does **not** call the
MCP tools inside a shell script.

Set these once. `NAMES` is the ordered section ids; the last is the hero/finale.

```bash
WORK=/tmp/scroll-world-video     # scratch dir for prompts, sources, frames
ASSETS=./assets                  # where the site reads stills (webp) + clips (mp4)
mkdir -p "$WORK" "$ASSETS/vid"
NAMES="farm kitchen shop delivery plaza finale"   # <-- your section ids, in order
```

Model ids are **not** hardcoded here. Before generating, fetch the Kubeez skills
(`get_skill`: `model-selection` always, `billing-confirmation` before video, `polling`,
`media-upload`) and call `get_models` to pick live `model_id`s. This doc uses
`gpt-image-2` (image) and a keyframe video model (default `seedance-2-720p`, peer
`kling-3-0-pro`) as the running examples; confirm the exact ids + `capabilities`
(`generation_types`, `max_input_images`, `duration_options`, `resolution`, `video_audio`)
at run time.

Kubeez jobs are **async**: `generate_media` returns a `generation_id`; poll
`get_generation(id)` (first poll ~10s image / ~30–60s video, then every ~5s) until
`status == "completed"` with non-null output URLs. Fire a batch as concurrent
`generate_media` calls, then poll the set.

## 1. Stills: master first, then scenes (Step 2)

Write one prompt file per section to `$WORK/still_<name>.txt` (see prompts.md), plus a
master-world prompt at `$WORK/still_master.txt`.

**MCP calls (you run these, not bash):**

1. Master still (cohesion anchor):
   ```
   generate_media(model='gpt-image-2', generation_type='text-to-image',
     prompt=<contents of still_master.txt>,
     aspect_ratio='16:9', resolution='2k', quality='high')
   ```
   Poll `get_generation` → keep the output URL as `MASTER_URL`.

2. Per-scene stills, derived from the master so they share palette/angle/light (fire all
   N concurrently, then poll the set):
   ```
   for each name in NAMES:
     generate_media(model='gpt-image-2', generation_type='image-to-image',
       source_media_urls=[MASTER_URL],
       prompt=<contents of still_$name.txt>,
       aspect_ratio='16:9', resolution='2k', quality='high')
   ```
   (`gpt-image-2` at 2K/4K requires an explicit non-square aspect; 16:9 qualifies. Confirm
   the model's accepted `generation_types` + `max_input_images` in `get_models`.)

**Download the outputs** (bash) so you have local PNGs for the webp posters and, later, the
dive first frames. Save each scene still URL to `$WORK/still_<name>.url` from the MCP
result, then:

```bash
for n in $NAMES; do curl -fsSL "$(cat "$WORK/still_$n.url")" -o "$WORK/still_$n.png"; done
```

Convert to webp for the site (and optionally run knockout.py first for transparency):

```bash
for n in $NAMES; do cwebp -quiet -q 84 -resize 1800 0 "$WORK/still_$n.png" -o "$ASSETS/$n.webp"; done
```

Review the stills for cohesion before continuing. Re-roll any off-style one (the master
reference already locks the look; tighten its prompt or re-derive from `MASTER_URL`).

## 2. Dive-in clips (Step 4)

Prompt files at `$WORK/dive_<name>.txt`. First frame = the scene still's **output URL**
(already HTTPS, no upload). **Preview cost first** (`get_skill('billing-confirmation')` →
`estimate` → `get_balance` → confirm), then fire the N dives concurrently:

```
for each name in NAMES:
  generate_media(model='seedance-2-720p', generation_type='image-to-video',
    source_media_urls=[<still_$name output URL>],
    prompt=<contents of dive_$name.txt>,
    aspect_ratio='16:9', duration=8, sound=false)   # resolution/duration per get_models
```

Poll each with `get_generation`, then download the rendered mp4s (bash). Save each output
URL to `$WORK/dive_<name>.url`:

```bash
for n in $NAMES; do curl -fsSL "$(cat "$WORK/dive_$n.url")" -o "$WORK/dive_$n.mp4"; done
```

Re-roll individual failures (transient errors and clean moderation flags are auto-refunded;
re-running is a normal prepaid call).

**Architecture A (forward take) instead of dives+connectors:** legs are **sequential**, not
concurrent. Leg 0's first frame is scene-0's still URL. Each later leg's first frame is the
**previous leg's actual last frame**, which you extract (§3) then upload (§4) to get a URL,
and you pass **only that one frame** (no last frame) so the camera never pulls back. Skip
the connector step (§4 second half) entirely.

## 3. Extract boundary frames, the seam handoff (Step 5)

For each adjacent pair, the connector's first frame = dive_i's LAST frame, last frame =
dive_{i+1}'s FIRST frame, extracted from the **rendered videos**, never the stills.

```bash
for n in $NAMES; do
  ffmpeg -v error -ss 0 -i "$WORK/dive_$n.mp4" -frames:v 1 -q:v 2 "$WORK/first_$n.png"       # establishing
  ffmpeg -v error -sseof -0.15 -i "$WORK/dive_$n.mp4" -frames:v 1 -q:v 2 "$WORK/last_$n.png" # interior
done
```

## 4. Upload frames, then generate connectors (Step 5)

These extracted PNGs are local files, so upload them to get URLs before they can seed a
connector. Fetch `get_skill('media-upload')`. For each adjacent pair (i = 1..N-1, between
`prev` and `n`), the connector needs TWO frames in order (first = `last_$prev.png`, last =
`first_$n.png`):

**MCP + curl per connector:**

1. `get_upload_url(model_id='seedance-2-720p')` → returns `direct_upload_url` + `token`
   (the model_id enforces `max_input_images >= 2`).
2. Upload both PNGs in ONE curl, **first frame then last frame** (order is preserved in the
   session). On Git Bash / Windows use the `C:/Users/...` path form, and always append
   `;type=image/png` (without it the endpoint 400s on `application/octet-stream`):
   ```bash
   curl -X POST "<direct_upload_url>" -F "token=<token>" \
     -F "file=@$WORK/last_$prev.png;type=image/png" \
     -F "file=@$WORK/first_$n.png;type=image/png"
   ```
3. `get_upload_session(token)` → `media_urls` (in upload order = [first, last]).
4. Generate the connector:
   ```
   generate_media(model='seedance-2-720p', generation_type='image-to-video',
     source_media_urls=[<media_urls[0]>, <media_urls[1]>],
     prompt=<contents of conn_$i.txt>,
     aspect_ratio='16:9', duration=5, sound=false)
   ```

Poll with `get_generation`, download to `$WORK/conn_<i>.mp4`. Connectors are independent, so
you can fire the upload+generate for all pairs concurrently. If one seam can't be
frame-matched (e.g. a first-frame-only model), hide that join with a fast push/whip at
assembly instead (SKILL Step 5).

## 5. Encode everything for scrubbing (Step 6)

Native resolution (encode whatever ffprobe reports; never upscale), crf 20, GOP 8, light
sharpen, no audio, faststart. Same for dives + connectors.

```bash
enc() { ffmpeg -v error -y -i "$1" -an -vf "unsharp=5:5:0.8:5:5:0.0" \
  -c:v libx264 -preset slow -crf 20 -pix_fmt yuv420p \
  -g 8 -keyint_min 8 -sc_threshold 0 -movflags +faststart "$2"; echo "enc $2 $(du -h "$2"|cut -f1)"; }

for n in $NAMES; do enc "$WORK/dive_$n.mp4" "$ASSETS/vid/$n.mp4"; done
i=0; for f in "$WORK"/conn_*.mp4; do i=$((i+1)); enc "$f" "$ASSETS/vid/conn$i.mp4"; done
```

Now the engine config's `sections[k].clip = assets/vid/<name>.mp4` and
`connectors = [assets/vid/conn1.mp4, …]` (length N-1, in order).

## 6. Mobile encodes (Step 6): mobile beta, only if the user opted in

**Skip this section unless the user chose the mobile (beta) version in the Step 1
interview.** Scrubbing sets `currentTime` every frame, and a phone decoder's **seek cost
scales with how many frames it must decode from the nearest keyframe**, so a full-resolution
`-g 8` master that scrubs fine on a laptop stutters on a phone. A **smaller frame + tighter GOP**
fixes that (and halves the bytes on cellular). Produce a `-m.mp4` sibling for every clip:

```bash
# 720p, GOP 4 (twice the keyframes = ~half the seek-decode work), crf 23, same sharpen/faststart.
encm() { ffmpeg -v error -y -i "$1" -an -vf "scale=-2:720,unsharp=5:5:0.6:5:5:0.0" \
  -c:v libx264 -preset slow -crf 23 -pix_fmt yuv420p \
  -g 4 -keyint_min 4 -sc_threshold 0 -movflags +faststart "$2"; echo "encm $2 $(du -h "$2"|cut -f1)"; }

for n in $NAMES; do encm "$WORK/dive_$n.mp4" "$ASSETS/vid/$n-m.mp4"; done
i=0; for f in "$WORK"/conn_*.mp4; do i=$((i+1)); encm "$f" "$ASSETS/vid/conn$i-m.mp4"; done
```

Wire the variants in the engine config; the engine serves them automatically on phones,
falling back to the desktop `clip` when a mobile one is absent:

```js
sections[k].clipMobile = 'assets/vid/<name>-m.mp4';
connectorsMobile = ['assets/vid/conn1-m.mp4', …];   // length N-1, in order
```

If phone scrubbing still stutters, tighten the GOP further (`-g 2`, or `-g 1` for all-intra
= instant seeks at the cost of larger files); if cellular weight is the bigger worry, raise
`crf` (24–26) or drop to `scale=-2:600`. If the master is already 720p, the mobile encode
still pays off (the tighter GOP is what makes phone seeks cheap). All mobile encodes stay
16:9; the engine centre-crops them; see the portrait note in SKILL Step 8 / prompts.md.

## Notes

- **Polling shortcut:** if your runtime exposes `wait_for_generation`, it's strictly better
  than a manual poll loop (returns the instant a job is done). Otherwise poll
  `get_generation(id)` on the cadence above. A job counts as done only when
  `status == "completed"` AND the output URLs are non-null (CDN/R2, never vendor temp links).
- **Content-filter fallback across models**: if one clip keeps getting flagged after
  re-rolls + prompt scrubbing, regenerate just that clip on `kling-3-0-pro` with the SAME
  first/last frames (upload them once, reuse the URLs), then restore your chain model. See
  SKILL Gotchas for the trade-off. Never fall over to an uncensored model (e.g. Wan) to
  dodge a moderation block; it won't bypass copyright/IP filters.
- **Previz on the cheap**: run the whole chain once with `model='seedance-2-mini'`
  (frame-locking intact, 480p/720p) to validate the journey and seams before spending
  full-model credits; because it's still seamless, the previz translates directly to the
  final render. Don't reach for reference-only image models here: without a settable first
  frame they can't hold a seam, so their output can't be chained (Step 4 rule).
- **Cost is per-second and non-refundable**: always `estimate` + `get_balance` and confirm
  before the video batch. Stills are flat-rate, so a preview is optional.
- Concurrency: firing ~5–6 `generate_media` calls at once is fine (Kubeez infra accepts
  concurrent jobs); if a batch stalls, check `get_generation` `error_message` and
  `get_balance` for credits.
