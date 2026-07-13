---
name: scroll-world-video
description: >
  Kubeez's own skill for building an immersive scroll-scrubbed "fly through the world"
  landing page for any industry or brand, generated end-to-end on the Kubeez platform. As
  the visitor scrolls, a pre-rendered camera flies forward from one scene into the next
  with NO cuts, one continuous connected flight (Emons-style isometric diorama world, or
  any art direction you pick). The skill interviews the user for the topic, story
  beats/sections, and brand kit, then generates cohesive scenes plus seamless forward-glide
  transitions with the Kubeez MCP (get_models, generate_media) and wires a portable,
  framework-agnostic scroll-scrub engine. Use when the user wants a "3D world" /
  "browse-through-the-industry" hero, a scroll cinematic, a diorama landing, or to turn a
  business into a scrollable Kubeez world.
allowed-tools: Bash, Read, Write, Edit, AskUserQuestion, Skill, mcp__kubeez__get_models, mcp__kubeez__generate_media, mcp__kubeez__get_skill, mcp__kubeez__estimate, mcp__kubeez__get_balance, mcp__kubeez__get_generation, mcp__kubeez__get_upload_url, mcp__kubeez__get_upload_session
---

# scroll-world-video (a Kubeez skill)

> **Requires the Kubeez platform.** This skill generates *exclusively* through the Kubeez
> MCP (`mcp__kubeez__*`) — stills, video, everything. It cannot build a world without it.
> Add the Kubeez MCP server (`https://mcp.kubeez.com/mcp`) and sign in with your Kubeez
> account ([kubeez.com](https://kubeez.com)). The scroll engine and method are open; the
> generation is Kubeez-powered by design.

Produces a landing page where **scroll drives a camera**: it dives from outside a scene
into its interior, then flies out and into the next scene, continuously, with no visible
cuts. The visuals are AI-generated with the **Kubeez MCP** (Kubeez is the user's own AI
media platform, 40+ models, so dogfood it); the page just scrubs pre-rendered video by
scroll position. This is the same technique behind Apple's scroll-through product pages:
the camera genuinely moves, scroll only drives time.

**What you generate:** 1 master world still → N scene stills (derived from the master for a
shared look) → N "dive-in" camera clips → N-1 "connector" clips that join consecutive
scenes seamlessly → a portable scrub engine that plays the whole chain as one flight.

**The one rule that makes or breaks it:** seams must be *frame-identical*. Read
[Connectors](#step-5--connectors-architecture-b-only) before generating any connector.
Getting this wrong is the single most common failure and produces a visible "pop" between
scenes.

Do not assume a frontend framework. The scrub engine in `references/scrub-engine.js` is
self-contained vanilla JS (it builds its own DOM + injects its own CSS into a container
you give it), so it drops into plain HTML, Next.js, Vue, a Python-served page, anything.
The value of this skill is the Kubeez generation pipeline, the prompts, and the seam
method, not the framework.

**Kubeez MCP rules apply.** Before each generation, fetch the relevant Kubeez skill with
`mcp__kubeez__get_skill` (**`model-selection`** always, **`billing-confirmation`** before
any credit-costing video, **`polling`** for async jobs, **`media-upload`** for the frame
handoff). Pick models with `mcp__kubeez__get_models` and use only the `model_id` values it
returns. Video generations cost credits and Kubeez has **NO refunds**, so `estimate` +
`get_balance` and confirm before the video batch.

---

## Step 0 — Bootstrap

1. **Kubeez MCP.** The `mcp__kubeez__*` tools are how you generate. Auth is the MCP
   connection itself (no CLI login). Confirm there are enough credits with
   `mcp__kubeez__get_balance`: a full run is roughly `N+1` image gens (master + N scenes)
   plus `(2N-1)` video gens. Video is credit-costing and **non-refundable**, so preview
   with `mcp__kubeez__estimate` and confirm before the paid batch (Step 4/5).
2. **ffmpeg / ffprobe** on `$PATH` (frame extraction + encoding). You still download the
   generated media and process frames locally; only the generation calls are remote.
3. **An image tool** for background knockout if you want floating scenes: PIL
   (`python3 -c "import PIL"`), or `cwebp`/`sips`. Optional, see Step 3.
4. Caveats:
   - macOS ships **bash 3.2** (no `declare -A`); don't use associative arrays in scripts.
   - Kubeez generations are **async**: `generate_media` returns a `generation_id`, then you
     poll `mcp__kubeez__get_generation(id)` (first-poll ~10s for an image, ~30–60s for
     video, then every ~5s) until `status == "completed"` and the output URLs are non-null.
     Fire the N scene/leg jobs concurrently (one call each) and poll the set. Never assume a
     job is done without polling.
   - **`source_media_urls` takes HTTPS URLs, not local paths.** Generation *outputs* are
     already URLs, so still→dive and master→scene chain directly. A frame you extract with
     ffmpeg is a *local* file, so it must be uploaded first (Step 5 upload flow) before it
     can seed the next clip.
   - Video models differ in accepted params and in whether they support first/last-frame
     conditioning at all. Before batching, confirm the chosen model's `capabilities`
     (`generation_types`, `max_input_images`, `duration_options`, `resolution`, `video_audio`)
     with `mcp__kubeez__get_models` and see the Step 4 model table.

---

## Step 1 — Interview the user

The **subject is the user's to state — ask it as an open question in plain prose**, never a
fabricated multiple-choice. A made-up list of industries biases them and reads as you
deciding their business for them; let them answer in their own words (their real business,
a client's, or any idea). Reserve structured multiple-choice (`AskUserQuestion` in Claude
Code; a plain either/or question elsewhere) for the genuinely
enumerable, lower-stakes choices below — art direction and brand-kit approach — and even
there, signal they can go their own way ("Other"). Ask only what you can't sensibly
default. Cover:

1. **Subject** (ask openly, not multiple-choice) — "What should this world be about? Your
   business, a client's, or any idea — a word or a sentence is fine." Capture the
   industry/product + a one-line pitch (e.g. "a bubble tea company, from leaf to last
   sip"), and a brand name if they have one; otherwise you'll propose one below.
2. **Brand kit** — offer two paths, pick one:
   - The user hands you palette + name + tone directly.
   - You propose a palette + name and let them approve.
   Capture **4–6 named hex values**, a display name, and a tone word or two. (If the user
   only has a website, read it yourself with your browsing tools to infer palette/tone;
   there is no Kubeez brand-kit fetch tool.)
3. **Art direction** — default is "soft matte low-poly **clay diorama**, isometric,
   tilt-shift miniature, warm light." Offer alternatives (flat papercraft, glossy toy,
   claymation, neon night). Whatever is chosen becomes the shared **style preamble**
   reused verbatim in every scene prompt (this is what makes the world cohesive).
4. **The journey (sections)** — the ordered scenes the camera flies through. Propose a
   set derived from the subject's own value chain and let the user edit. 5–7 works well.
   Boba example: farms → pearl kitchen → flagship shop → delivery → community plaza →
   the hero product. Each section needs: a short subject description (what's IN the
   diorama), an eyebrow, a headline, one line of body, and 0–3 tag pills. The last
   section is usually the hero product + the CTA.
5. **Mobile version (beta) — ALWAYS ask this; never silently generate both.** Ask as a
   two-option choice (`AskUserQuestion` in Claude Code; a plain question elsewhere):
   *"Want a mobile-optimized version too? Mobile support is in
   **beta** — the scroll-scrub mechanic is desktop-native; on phones you get lighter
   encodes and engine hardening, but portrait crops the 16:9 frame and low-end devices
   may still stutter."* Options: "Desktop only" / "Desktop + mobile (beta)". The beta
   disclaimer must be stated to the user, not just implied. What the answer gates:
   - **Yes** → produce the `-m.mp4` mobile encodes (Step 6) and wire
     `clipMobile`/`connectorsMobile` (Step 7); run the full mobile QA (Step 8). If any
     scene's focal subject sits off-centre, offer the 9:16 hero-variant escape hatch
     (extra Kubeez credits, say so).
   - **No** → skip the mobile encodes and wiring entirely. The engine's phone hardening
     (seek-coalescing, iOS priming, safe-area CSS) is always on regardless — that's not
     a "mobile version," it's just the page not breaking when a phone visits — so a
     desktop-only build still degrades gracefully.

Video model is **not** an interview question. Default to a keyframe-capable Kubeez video
model silently (confirm the exact `model_id` with `get_models` at run time, e.g.
`seedance-2-720p` or `kling-3-0-pro`, see Step 4). If the user names a preference, honor it
**only if it can frame-lock seams** (accepts a first/last frame via `source_media_urls`).
This skill only ships seamless output, so a model that can't frame-lock is declined with a
one-line why, not substituted in; use a roster model instead.

Keep the scroll mechanic fixed (continuous fly-through) — that's the point of the skill.
See `references/prompts.md` for the intake checklist and copy structure.

---

## Step 2 — Generate the stills (master first, then the scenes)

First fetch `get_skill(name="model-selection")`, then `get_models(model_type='image')` and
pick an image `model_id` (default **`gpt-image-2`**: sharpest, best value, great at isometric
illustration, and returns a solid/white background perfect for floating diorama "islands").
Use only ids `get_models` returns. `nano-banana-2` is the one escape hatch, and only for
aspect `4:5`, which `gpt-image-2` cannot render.

Generate at **`aspect_ratio='16:9'`** so each still equals the clip frame (this makes the
first-frame handoff into the dive clip cleaner). `gpt-image-2` supports 1:1/9:16/16:9/4:3/3:4
(no 3:2). Note: at `resolution` 2K/4K `gpt-image-2` **requires** an explicit non-square
aspect, and 16:9 qualifies; 1K (`resolution='standard'`) is fine for web.

**1. Master world still (cohesion anchor).** Generate ONE master isometric still of the whole
world using the shared style preamble, so every scene inherits one coherent look:

```
mcp__kubeez__generate_media(
  model='gpt-image-2', generation_type='text-to-image',
  prompt='<STYLE PREAMBLE> ... a single cohesive miniature world overview',
  aspect_ratio='16:9', resolution='2k', quality='high')
```

Poll `get_generation(id)` until complete; keep the output URL as `MASTER_URL`.

**2. Per-scene stills, derived from the master.** One image per section, **all reusing the
same style preamble** AND seeded from the master for a shared palette/angle/light. Derive each
with an image-to-image reference pass on the master URL (confirm the model's accepted
`generation_types` + `max_input_images` in `get_models`):

```
mcp__kubeez__generate_media(
  model='gpt-image-2', generation_type='image-to-image',
  source_media_urls=[MASTER_URL],
  prompt='<STYLE PREAMBLE>. Subject: <what is in THIS diorama>. No text, centered.',
  aspect_ratio='16:9', resolution='2k', quality='high')
```

- Fire all N concurrently (one `generate_media` call each), then poll the set with
  `get_generation`. Output URLs are already HTTPS, so they feed straight into Step 4 as
  dive/leg start frames, no upload needed.
- A generation may fail transiently; re-roll that one individually, don't restart the batch.
- Download each scene's output URL (`curl`) for the local webp poster + fallback (Step 3).
- **Review the stills before continuing.** They must read as one cohesive world (same angle,
  palette, light). If one is off-style, regenerate it (the master reference already locks the
  look; tighten the prompt or re-derive from the master).

See `references/pipeline.md` for the exact per-step flow.

---

## Step 3 — (Optional) Float the scenes

If you want the dioramas to float over an atmospheric background instead of sitting in a
solid box, knock out the flat background to transparency with
`references/knockout.py` (border-connected flood fill — preserves interior colour that
matches the bg, e.g. cream walls). Then encode to webp. If you'd rather keep it simple,
just make the page background the same colour as the scene background and skip this.

These stills double as **video posters and lazy-load fallbacks**, so keep them.

---

## Step 4 — Camera architecture (pick one — this makes or breaks the feel)

How the camera moves *between* scenes is the single biggest quality lever. Two shapes;
pick by aesthetic.

### Video model — pick ONE for the whole chain

**This skill only ships seamless output**, so the only usable models are ones that can
frame-lock a seam: every chained clip must accept a **first frame** in `source_media_urls`,
and connectors also need a **last frame** (a 2nd image the model interpolates toward). That
capability, not preference, is the selection rule. Fetch `get_skill(name="model-selection")`,
then run `get_models(model_type='video')` and read each row's `capabilities`
(`generation_types` must include `image-to-video`; `max_input_images >= 2` for connectors).
**Skip anything whose image input is reference-only** (can't set the first frame): it can
only *condition* a generation, not *continue* a shot, so it physically can't hold a seam.
Candidate keyframe models (confirm the exact `model_id` + limits live in `get_models`):

| Model (confirm via get_models) | first / last frame | Notes |
|---|---|---|
| `seedance-2-720p` (default) | ✓ / ✓ | Full chain (legs + connectors); image-to-video takes a first frame and an optional 2nd (last) frame. Per-second billing, audio **free**. Its content filter is the touchy one (see Gotchas). Resolution/duration per `get_models`. |
| `kling-3-0-pro` | ✓ / ✓ | **Content-filter fallback ONLY, not for quality.** Field testing found Seedance 2 has better, steadier camera control for these gliding transitions; Kling's motion drifts more. Reach for it solely when Seedance's INPUT filter rejects a frame (e.g. prominent clay figures flagged as "real person"). Per-second billing; audio is a surcharge, so pass `sound=false`. |
| `seedance-2-mini` | ✓ / ✓ | Cheapest tier (~50% under Fast, 480p/720p) that keeps frame-locking. The **previz** tier: run the whole chain here first to validate the journey + seams, then re-render final legs on the full model. Still seamless, so it translates directly. |

Those are the roster; all do both architectures. (`kling-3-0-turbo` also frame-locks via a
single first frame but takes only ONE image, so it's architecture-A-only and can't make
connectors. It's not in the default roster; only reach for it, and wire it by hand, if
architecture A's sequential render time is a proven bottleneck.)

Rules:
- **One model for all chained clips.** Each renderer has its own motion/color/grain
  character; mixing models mid-chain keeps *position* continuity (frames still hand off) but
  the render-character shift reads as a subtle pop. The one sanctioned exception is the
  content-filter fallback for a single stubborn clip (Gotchas): a slight character shift on
  one 5s connector beats a missing connector.
- **Prefer Seedance 2 for the whole chain** — `seedance-2-fast` (best quality/value;
  `-720p` for a presentation-grade build, `-480p` for a cheap previz) or `seedance-2-720p`
  standard. Field-proven to have the best camera control for these forward glides. Use Kling
  ONLY as a content-filter fallback for a single stubborn clip, never for quality. Honor a
  user's stated model **only if it qualifies** (frame-locking, `image-to-video`); if it
  doesn't, say so and use a Seedance 2 model. Never ship a non-seamless build to satisfy a
  model request.
- **Preview cost before rendering.** Video is per-second and Kubeez has NO refunds. Run
  `get_skill(name="billing-confirmation")`, then `estimate(model, generation_type='image-to-video', duration, sound=false)`
  and `get_balance` before the paid batch, and confirm with the user.

### A) Continuous forward take — RECOMMENDED for grounded / realistic / walkthrough
One camera that only ever glides **forward**, first scene through last, as a single take.
Generate the legs **sequentially**: leg 0 from scene-0's still URL (glide forward into it);
then each leg's **first frame** = the **previous leg's ACTUAL last frame** (extract with
ffmpeg, then upload it via Step 5's flow to get a URL for `source_media_urls`), prompt
*"continue gliding smoothly FORWARD into [scene i], never pulling back"* (or an expressive
mid-leg move under the motion-handoff contract, see **Camera grammar** below), and **pass
only that ONE image, no last frame**. A last frame of a wide establishing shot forces the
camera to pull back, which is the #1 cause of stutter. Extract each leg's last frame to feed
the next. Result: every seam is frame-identical **and** the camera never reverses. There are
**no connectors** (skip Step 5); the legs ARE the journey. Wire each leg as a section clip
with `connectors: []` and a small `crossfade` (~0.08). Even without a last frame the legs
still arrive at distinct rooms (the prompt steers the content). Cost: strictly **sequential**
(can't parallelize) and slower; interiors trip the content filter, so build in re-rolls
(3 attempts/leg).

### B) Dive-in + forward-glide connector — RECOMMENDED, works for any world (field-proven)
A "dive into each scene" clip (video 1, 2, 3, 4…) + a connector between each pair, built
from the REAL boundary frames: connector's first frame = video N's actual LAST frame,
connector's last frame = video N+1's actual FIRST frame (Step 5). The engine plays
video1 → connector1 → video2 …, so scrolling off the end of a dive kicks its transition.
**Keep the connector a low, continuous FORWARD glide** (see the Step 5 prompt) — this is the
version that scrubs as one smooth shot.

> Legacy warning: older versions prompted the connector to "pull up and out / rise into the
> sky / fly to a map view." That **reverses camera direction at the seam** (forward dive →
> backward pull-out) and reads as a jarring rewind/jump when scrubbed, even with frame-perfect
> seams. Do NOT do that. A gentle forward glide between the two real frames is smooth; if the
> two scenes are far apart, give the connector a longer `duration` (7–10s) rather than a
> faster/bigger move.

Architecture A (continuous forward take) is the alternative when you want the dives themselves
to chain with no separate connectors; B is the default for distinct, pre-approved scenes.

### Camera grammar — the move should fit the concept (A is NOT "forward only")

"Forward only" is the *seam* rule, not the *leg* rule. The physics of the chain:

- **Position continuity** at a seam comes from the frame handoff (next leg starts from the
  previous leg's actual last frame).
- **Velocity continuity** at a seam means the camera must never *reverse across a seam* —
  that's the rewind stutter.
- **Inside a single leg the camera is free.** One leg is one continuous render — there is
  no seam to break mid-leg, so orbits, crane-ups, lateral tracking, even a push-in that
  eases back out are all safe *within* the clip. Reversals are only fatal *across* seams.

So give each leg an expressive move chosen from the scene's own logic, under a **motion
handoff contract**: every leg **ends by settling into a slow, steady forward drift** toward
the next destination (final ~1 s), and every leg **begins by continuing that same drift**.
Keep both clauses in the prompts verbatim (templates in `references/prompts.md`).

Pick the grammar from the concept:

| Concept / tone | Mid-leg move |
|---|---|
| Product / luxury retail | slow half-orbit around the hero object, then continue past it |
| Real estate / hospitality | steadicam glide through doorways; gentle crane-up in atria |
| Industrial / process / logistics | low lateral track alongside the line, foreground parallax |
| Travel / outdoors / campus | drone-style rise-and-reveal, then a descending swoop |
| Food / craft / detail-driven | push in close to the craft moment, ease back, carry on |
| Playful miniature (arch. B) | dives + aerial hops — the connector IS the grammar |

Honest costs: expressive mid-leg moves raise re-roll odds — the model can end a fancy move
in a state that isn't a clean forward drift. Mitigations: keep the final-second settle
clause verbatim; **eyeball each leg's last frame before chaining the next** (it should look
like a frame from a gentle forward glide — if not, re-roll before wasting the next leg);
budget ~1 extra re-roll per expressive leg. A plain forward glide stays the zero-risk
default — use it for legs where the scene itself is the show.

Two related pacing knobs live in the engine (Step 7): per-section `scroll` (more scroll
distance = longer dwell in that scene) and `linger` (the camera settles mid-scene exactly
while the copy peaks, then picks up speed toward the seam). Prefer expressive motion in the
*clip* and restraint in the *scrub mapping* — they compound.

And remember scroll is a scrubber: visitors can scroll **up**, so every move also plays in
reverse. That's free and expected — no extra work — but it's another reason seam velocity
must be consistent in both directions (a seam that reads fine forward reads as a stutter
backward too if velocity flips).

**For B**, one camera flight per scene: starts high/outside, descends into the interior,
structure opens. Model: the chain model you picked above (default **`seedance-2-720p`**),
`generation_type='image-to-video'`, `source_media_urls=[<the scene still URL>]` (a single
first frame, no last frame for the dive).

- Use the **solid-background still** (not the knocked-out transparent one) as the first
  frame, so the video has a full frame.
- Prompt: "Single continuous cinematic camera move, no cuts. Begin high and far looking
  at the whole <scene> from outside … descend and fly inside toward <focal point> … the
  roof/walls gently open to reveal the interior. <style>, smooth graceful slow motion.
  No text." (Template in `references/prompts.md`.)
- Params: `aspect_ratio='16:9'`, `duration=8`, `sound=false` (you mute anyway, and it
  avoids Kling's audio surcharge), plus `resolution` per `get_models`. The default
  `seedance-2-720p` renders 720p; 1080p needs the standard tier's 1080p resolution. Read the
  model's real `duration_options` and `resolution` from `get_models` rather than assuming.
- Fire the N dives concurrently (one `generate_media` call each), poll with
  `get_generation`, then download each output URL. Re-roll individual failures. Keep the raw
  sources; you need their frames next.

---

## Step 5 — Connectors (architecture B only)

Skip this whole step for architecture **A** — the forward take has no connectors; its legs
already chain seamlessly. This step applies to **B** (diorama/miniature), and note the
reversal caveat from Step 4.

The connector clips are what make the world feel *connected* instead of cut. A connector
flies from the end of scene i out and into the start of scene i+1. **Both of its
endpoints must be the ACTUAL RENDERED FRAMES of the neighbouring clips — never the
original diorama still.**

Why: every generation renders slightly differently. If a connector *ends* on a fresh render
of "the kitchen diorama," but the next dive clip *starts* on its own different render of that
same diorama, the two won't match and you get a pop at the seam. The fix is to hand off the
exact pixels as the connector's two keyframes:

```
For each connector between dive_i and dive_{i+1}:
  first frame (source_media_urls[0]) = the LAST frame extracted from dive_i's rendered video
  last  frame (source_media_urls[1]) = the FIRST frame extracted from dive_{i+1}'s rendered video
```

The keyframe model interpolates from the first to the last frame, so every seam is
frame-identical on *both* sides: `dive_i.end == connector.start` and
`connector.end == dive_{i+1}.start`.

Extract the boundary frames from the rendered dives (not the stills):

```bash
ffmpeg -sseof -0.15 -i dive_i.mp4   -frames:v 1 -q:v 2 dive_i_last.png    # interior of i
ffmpeg -ss 0      -i dive_{i+1}.mp4 -frames:v 1 -q:v 2 dive_next_first.png # establishing of i+1
```

Those PNGs are **local files**, and `source_media_urls` needs HTTPS URLs, so upload them
first (order matters, first frame then last). Fetch `get_skill(name="media-upload")`, then:

```
get_upload_url(model_id='<video model>')                       # returns direct_upload_url + token
# POST both PNGs to that session in ONE curl (order = first, last):
curl -X POST "<direct_upload_url>" -F "token=<token>" \
  -F "file=@C:/work/dive_i_last.png;type=image/png" \
  -F "file=@C:/work/dive_next_first.png;type=image/png"       # Windows path form on Git Bash
get_upload_session(token)                                       # returns media_urls in upload order
```

Then generate the connector (`duration=5` is plenty). Connectors need a **last frame**, so
`max_input_images >= 2` (any roster model qualifies: `seedance-2-720p`, `seedance-2-mini`,
`kling-3-0-pro`):

```
mcp__kubeez__generate_media(
  model='<video model>', generation_type='image-to-video',
  source_media_urls=[<uploaded dive_i_last URL>, <uploaded dive_next_first URL>],
  prompt='<connector prompt>',
  aspect_ratio='16:9', duration=5, sound=false)   # resolution per get_models
```

Connector prompt: "Single continuous smooth camera movement, no cuts, no pulling back.
Glide gently and steadily FORWARD through the connected miniature world, flowing from
<scene i> across to <scene i+1>, one seamless continuous motion. <style>. No text."
**Never** tell the connector to pull up / rise into the sky / arrive above / go to a map
view — that reversal is the #1 cause of a scrubbed "jump"; keep it a low forward glide.
Give a long-distance transition a longer `duration` (7–10s) so the move stays slow and
smooth. (Template + rationale in `references/prompts.md`.)

Insurance: the model lands *close* to the last frame but not always pixel-perfect, so the
engine still applies a **short crossfade** (a few frames) at each seam. Frame-matched
endpoints + a small crossfade = no visible cut. Never skip the actual-frame handoff and
rely on the crossfade alone; a big content jump can't be hidden by a crossfade. Where a
true frame match is impossible (a model that only takes a first frame, or a clip you
couldn't re-roll to match), hide that one join with a fast push/whip transition at assembly
time instead.

---

## Step 6 — Encode for smooth scrubbing

Scrubbing = setting `video.currentTime` from scroll. Two things matter, and they are
often gotten wrong:

1. **Seekability, not keyframe density, is what makes scrubbing work.** Many static
   hosts (and `python -m http.server`) don't serve HTTP byte-range requests, which pins
   `video.seekable` to `[0,0]` and clamps *every* seek to frame 0 — the video looks
   frozen. The robust fix is to **fetch each clip as a `Blob` and play it from an
   in-memory object URL** (blobs are always fully seekable). The engine does this.
   Because of it, you do **not** need all-intra video.
2. **Don't shrink quality to get smooth seeks.** Encode at the **native resolution**
   (whatever ffprobe reports for your model, e.g. 720p or 1080p; don't upscale and don't
   downscale), `crf ~20`, a **small GOP** (`-g 8`) rather
   than all-intra (all-intra bloats an 8s clip to ~25 MB; GOP 8 is ~8 MB and scrubs
   fine via blob). Strip audio, add faststart, and a light `unsharp` counters video
   softness:

```bash
ffmpeg -i src.mp4 -an -vf "unsharp=5:5:0.8:5:5:0.0" \
  -c:v libx264 -preset slow -crf 20 -pix_fmt yuv420p \
  -g 8 -keyint_min 8 -sc_threshold 0 -movflags +faststart out.mp4
```

Encode all 2N-1 clips (dives + connectors) with the same settings for uniform quality.

**Mobile encodes (beta — only if the user opted in at Step 1.5).** Phone video decoders seek
far slower than a laptop's, and seek cost scales with GOP length, so the native-resolution
`-g 8` master that scrubs smoothly on desktop can stutter on a phone. Produce a lighter `-m.mp4` sibling for
every clip — **720p, `-g 4`** (more keyframes = cheaper seeks), crf 23 — and wire them as
`clipMobile` / `connectorsMobile` (Step 7). The engine serves them automatically on phones and
falls back to the desktop clip when absent. The exact `encm()` script is in
`references/pipeline.md` §6. If the user chose desktop-only, skip this — the engine still
hardens phone scrubbing regardless (seek-coalescing, iOS priming), so the page degrades
gracefully rather than breaking.

---

## Step 7 — Assemble the page

Copy `references/scrub-engine.js` (and, if you want a fully standalone page, the tiny
`references/index-template.html`) into the user's project — or adapt into their
framework. It's config-driven and self-contained:

```js
mountKubeezWorld(document.getElementById('world'), {
  brand: { name: 'Pearl & Co.' },
  diveScroll: 1.3, connScroll: 0.9,          // viewport-heights of scroll per clip
  sections: [
    { id:'farm', label:'The Farms', still:'assets/farm.webp',
      clip:'assets/vid/farm.mp4', clipMobile:'assets/vid/farm-m.mp4',   // mobile beta only
      scroll: 1.6, linger: 0.45,   // optional pacing: longer dwell + camera settles mid-scene
      accent:'#8FB98A', eyebrow:'From leaf to last sip', title:'It starts in the hills.',
      body:'…', tags:['Single-origin','Hand-picked'] },
    // …one per section; last may carry a `cta`
  ],
  connectors:       ['assets/vid/conn1.mp4','assets/vid/conn2.mp4',   /* … length = sections-1 */],
  connectorsMobile: ['assets/vid/conn1-m.mp4','assets/vid/conn2-m.mp4' /* … same length; mobile beta only */],
});
```

The engine handles: the ordered dive/connector chain, scroll→currentTime with rAF
smoothing, blob loading, lazy prefetch of nearby clips, frame-matched crossfades, pinned
per-section copy (first section greets on landing, last holds its CTA), a route rail,
`prefers-reduced-motion`, and mobile. **Pacing per section:** `scroll` overrides
`diveScroll` for that scene (more scroll = longer dwell) and `linger` (0–1, keep ≤ 0.6)
remaps time so the camera settles mid-scene — exactly while the copy peaks — then speeds
up toward the seam; seam frames are untouched (f(0)=0, f(1)=1). Give the hero and finale
scenes a higher `scroll` + some `linger`; keep transit scenes brisk. Theme it with CSS variables (`--accent`,
`--sw-bg`, `--sw-ink`, …) — the visual identity comes from the generated clips, so the
chrome stays quiet. See the header of `scrub-engine.js` for the full config + CSS vars.

**On phones the engine adapts automatically** (coarse pointer or ≤860px): it serves
`clipMobile` / `connectorsMobile` when present, **coalesces seeks** (never queues a new
`currentTime` while the decoder is still seeking — this is what stops a fast flick from
freezing the clip), **keeps the still as a poster until the clip paints its first frame**
and **primes each video on first touch** (fixes iOS's blank-until-played video), drops the
drifting particles, ignores URL-bar-only resizes (no scroll jump), and uses safe-area
insets so copy clears the notch/home indicator. All of this hardening is on by default —
no config needed. The `clipMobile`/`connectorsMobile` encodes are the opt-in **mobile
beta** part (Step 1.5): only wire them when the user asked for the mobile version.

For non-JS backends (Python/Rails/etc.): serve the assets and drop the engine `<script>`
into the rendered HTML; nothing about it is framework-specific.

---

## Step 8 — QA the seams (don't skip)

Drive the page in a headless browser and **verify frame continuity at the seams**, which
is the thing most likely to be wrong:

- Screenshot at scroll positions just before and just after each seam. The two frames
  must be near-identical (the dive's last frame == the connector's first frame). If they
  pop, you used the diorama still instead of the actual rendered frame (redo Step 5), or
  the crossfade band is too short.
- Check the console for errors, confirm `video.seekable.end(0) > 0` (blob working), and
  that `currentTime` tracks scroll across each clip's band.
- **Mobile — full checklist only if the user opted into the mobile beta (Step 1.5).**
  For a desktop-only build, just sanity-check a phone viewport once: page loads, still
  posters show, nothing overlaps — the engine's hardening covers graceful degradation.
  For the beta (do this on a real phone or an emulated one, portrait + landscape):
  - Emulate a phone viewport **with CPU throttled 4–6×** and scroll fast — the clip should
    track without freezing (the seek-coalescing + `-m.mp4` encodes are what make this hold).
  - Confirm the first scene shows immediately (its still is the poster) and the video takes
    over the instant you scroll — no blank/black scene (the iOS priming fix). Test iOS Safari
    specifically; it's the one that goes blank if this regresses.
  - Verify the `-m.mp4` variant is actually served on mobile (Network panel), and the
    heavier full-resolution master on desktop.
  - Slowly scroll so the URL bar collapses — the page must **not jump** (height-only resizes
    are ignored on touch). Rotate the device — layout should recompose cleanly.
  - Portrait crops a 16:9 clip to its centre; confirm the focal subject still reads. If a
    hero scene's subject sits off-centre and gets cut, recompose it (prompts.md) or generate
    a 9:16 variant for that scene.
- Check reduced-motion (should fall back to the stills, no video, no particles).

---

## Gotchas (hard-won)

- **Seam pop** → connector endpoints were the diorama stills, not the neighbouring
  clips' actual frames. Always extract real frames (Step 5).
- **Seam stutter / camera "jumps backward"** → even with frame-matched seams, if the
  camera *velocity reverses* (forward dive, then a connector that pulls back out) it
  reads as a rewind. This is inherent to architecture B. For any grounded walkthrough use
  architecture A (one continuous forward take: legs chained from actual last frames, no
  pull-back, only a first frame passed, no last frame); see Step 4.
- **Frozen video / stuck at frame 0** → `seekable=[0,0]`; the host isn't serving byte
  ranges. Use blob URLs (engine does).
- **Huge files** → you used all-intra. Use `-g 8` + blob instead.
- **Soft / low quality** → you downscaled or over-compressed. Encode at the model's native
  resolution (whatever ffprobe reports; never upscale), crf ≤ 20, add `unsharp`. Video is
  inherently softer than the stills, so keep the stills as the lite fallback for max fidelity.
- **Concurrent gens fail / "insufficient credits" race** → transient when many launch at
  once; re-roll the individual failure, it's not really out of credits (verify with
  `mcp__kubeez__get_balance`).
- **Content-filter false-positives (video `status: "failed"` on a moderation flag)** → the
  video content filter flags perfectly innocuous clips, especially **bedroom, pool,
  spa/wellness** contexts and trigger words like "bed", "pool", "waterfall", "wine", "swim".
  It's partly the prompt wording and partly the reference frames. Fixes, in order: (1)
  re-roll, it's often non-deterministic and passes on the 2nd–3rd try (a clean moderation
  failure is auto-refunded, so re-running is a normal prepaid call); (2) strip trigger words
  and add "empty, unoccupied, no people, no figures, architectural, tasteful"; (3) regenerate
  just that clip on **`kling-3-0-pro`** with the same first/last frames, a different
  provider's filter often passes what Seedance blocks. Expect a slight render-character
  shift on that one clip (each model has its own grain/motion feel); for a 5s connector
  behind a crossfade that usually beats option (4): set the connector slot to `null` —
  the engine crossfades that seam directly (optional connectors), so the page still
  completes. Budget extra credits/time for these re-rolls on interiors/real-estate content.
- **Dark / custom theme** → the engine wraps its default tokens in `@layer sw`, so a
  page-level `:root` / `.sw-root { --sw-bg; --sw-ink; --sw-accent; --sw-font-* }` block
  wins cleanly (no specificity hacks). `--sw-ink` is your primary **text/heading** colour;
  the **accent** fills the primary button and active nav. For a dark theme, set `--sw-bg`
  dark and `--sw-ink` light — the copy scrim and title shadow follow `--sw-bg` automatically.
- **Phone scrub stutters / freezes on a fast flick** → the full-resolution master is too
  heavy for a phone decoder and seeks pile up. Ship the `-m.mp4` mobile encodes (720p, `-g 4`) and wire
  `clipMobile`/`connectorsMobile` (Step 6/7). The engine already coalesces seeks; the lighter
  encode is the other half. Still choppy on a low-end device? Tighten GOP (`-g 2` / all-intra).
- **Blank / black scene on iOS (desktop was fine)** → an iOS Safari quirk: a muted video that
  was never played won't paint a seeked frame. The engine fixes this by keeping the still as a
  poster until the clip paints and priming each video on first touch — so **don't** hide the
  still on `loadedmetadata` or strip the `playsinline`/`muted` attributes if you adapt the
  engine into a framework.
- **Page jumps while scrolling on mobile** → something is re-running layout on the URL-bar
  show/hide `resize`. The engine ignores height-only resizes on touch; if you ported it, gate
  your resize handler on a width change (keep the `orientationchange` path for rotation).
- **Copy hidden behind the URL bar / notch on mobile** → use the engine's safe-area-aware
  bottom offset (`env(safe-area-inset-bottom)` + `dvh`); make sure the page's
  `<meta viewport>` includes `viewport-fit=cover` (the template does).
- **Portrait crops the scene** → a 16:9 clip on a tall phone shows only its centre. Keep each
  scene's focal subject centred with a little headroom (prompts.md), or generate a 9:16 hero
  for the scenes that matter most. The engine centre-crops (`object-fit:cover`); it can't
  un-crop a widescreen composition.
- **Audio you didn't want** → `sound` defaults **on** for video models. Pass `sound=false`;
  you mute in HTML and `-an` on encode anyway, and on Kling 3.0 it drops the audio surcharge.
- **Wrong resolution / duration** → don't assume; read each model's real `resolution`,
  `duration_options`, and `video_audio` from `get_models`. Encode at whatever native
  resolution ffprobe reports, never upscale.
- **Seam pop only where you "saved credits"** → you swapped models mid-chain, or used a
  first-frame-only model where a connector needs a last frame (`max_input_images >= 2`).
  One model for the whole chain; the only cheap tier is `seedance-2-mini`, which keeps
  frame-locking so it stays seamless. (Any model with reference-only image input can't hold
  a seam at all, see Step 4.)
- **White-box scenes** → `gpt-image-2` returns a solid bg; either match the page bg to it
  or knock it out (Step 3).
- **bash 3.2** on macOS → no associative arrays in scripts.

## References

- `references/prompts.md` — the intake checklist, style-preamble pattern, and every
  prompt template (scene still, dive, connector) with fill-in slots.
- `references/pipeline.md` — the run flow: Kubeez MCP generation/upload/poll calls
  interleaved with copy-paste bash for download, extract frames, encode, and mobile encode
  (bash-3.2-safe).
- `references/scrub-engine.js` — the portable, config-driven scrub engine (builds DOM +
  injects CSS; blob-seek, lazy load, seam crossfade, copy, route rail, reduced-motion, and
  phone hardening: mobile encodes, seek-coalescing, iOS priming, safe-area, no-jump resize).
- `references/index-template.html` — a minimal standalone page that mounts the engine.
- `references/knockout.py` — border-connected background knockout for floating scenes.
