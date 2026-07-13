# Prompt templates & intake

Everything here is fill-in-the-slots. Keep the **style preamble** byte-for-byte identical
across all scene stills — that identical text is what makes the world feel like one place.

## Intake checklist (Step 1)

Collect and write down:

- `SUBJECT` — the business + one-line pitch.
- `BRAND_NAME` — display name.
- `PALETTE` — 4–6 named hexes, e.g. `taro #9B7EBD, cream #F5EDE0, caramel #C88A5A, matcha #8FB98A, plum #3A2E48`. Pick ONE as the scene **background** colour (usually the lightest) and one as the primary **accent**.
- `TONE` — a word or two (cozy/premium, playful, industrial…).
- `STYLE` — the art direction (default below).
- `SECTIONS[]` — ordered list; for each: `id`, `label`, `subject` (what's in the diorama), `eyebrow`, `title`, `body` (≤ 1 sentence), `tags[]` (0–3). Last section = hero product + CTA.
- `MOBILE` — yes/no. **Always asked** (SKILL Step 1.5), presented to the user as **beta**. Gates the `-m.mp4` encodes (pipeline §6) + `clipMobile`/`connectorsMobile` wiring + the full mobile QA.

## Style preamble (default: clay diorama)

Reuse verbatim in every scene prompt. Swap the bracketed bits for the brand's palette/bg.

```
Isometric low-poly 3D diorama floating as a small rounded island on a plain solid
[BG_HEX] background with a soft contact shadow beneath it. Soft matte clay 3D render,
rounded toy-model shapes, gentle warm studio lighting, soft long shadows, tilt-shift
miniature look. Cohesive color palette of [PALETTE]. Highly detailed, centered
composition, absolutely no text, no letters, no numbers, no logos.
```

Alternate directions (swap the first two sentences, keep the palette/no-text tail):
- **Flat papercraft:** "Isometric layered paper-craft diorama, matte cardstock, clean die-cut edges, subtle drop shadows between layers."
- **Glossy toy:** "Isometric glossy vinyl-toy diorama, smooth plastic shading, soft rim light, collectible figurine look."
- **Claymation:** "Isometric stop-motion clay set, visible thumbprints, handmade plasticine texture, soft studio softbox light."
- **Neon night:** "Isometric miniature at night, warm interior glow and neon signage, moody rim light, wet reflective ground."
- **Photoreal architectural** (real estate, hospitality, premium/luxury): "Ultra-photorealistic architectural photography of a single cohesive [subject], cinematic wide-angle, warm golden-hour light, natural materials, restrained designer furnishings, a breathtaking view, editorial magazine quality (Architectural Digest), shallow depth of field, no people." For photoreal, drop the floating-island framing and the knockout (Step 3) — the scenes are **full-bleed** (a dark page background reads premium), the "dive" glides *through doorways/glass* rather than opening a roof, and cohesion comes entirely from the identical preamble (for photoreal, generate each scene `text-to-image` and do NOT derive from the master via `image-to-image`, since a real-room reference clones the same room). Interiors trip the video content filter often; see SKILL Gotchas.

## Master world still prompt (Step 2, generated FIRST)

One overview of the whole world, `generation_type='text-to-image'`. Its output URL becomes
the reference every scene still is derived from (`image-to-image`), which is what locks the
shared palette/angle/light. (Photoreal is the exception: skip the master-derive, see the
photoreal note above.)

```
[STYLE PREAMBLE]
Subject: a single cohesive miniature world overview showing [the whole business at a glance:
the main districts/stages arranged as one connected island cluster]. One unified palette and
light. Centered, no text.
```

## Scene still prompt (Step 2)

`generation_type='image-to-image'`, `source_media_urls=[MASTER_URL]` so the scene inherits
the master's look.

```
[STYLE PREAMBLE]
Subject: [SECTION.subject — describe the miniature scene: the building/space, a few
characters doing the work, the props that signal this stage of the business].
```

Tips:
- Name concrete props (they anchor the scene): tanks, cauldrons, conveyor, crates, awning, string lights, benches, scooters, map pins.
- For the final "hero product" section, drop the diorama-island framing and prompt a
  single oversized product centerpiece floating on the same background with a few small
  orbiting props.
- **Compose for the centre.** The page renders every clip `object-fit:cover`, and a portrait
  phone crops a 16:9 frame to roughly its centre half. Keep the focal subject horizontally
  centred with a little headroom, and don't park anything essential at the far left/right
  edges — it will be cut off on phones. This also keeps the dive's focal point (which the
  camera flies toward) inside the mobile crop. For a scene that absolutely must show its full
  width on mobile, generate a separate 9:16 variant for it.
- Aspect `16:9` (matches the clip frame for a clean handoff), `resolution='2k'`,
  `quality='high'` (or `resolution='standard'` for 1K web). `gpt-image-2` at 2K/4K requires
  an explicit non-square aspect, which 16:9 satisfies.

## Leg prompt — architecture A, continuous forward take (Step 4)

`generation_type='image-to-video'`, `source_media_urls=[<previous leg's ACTUAL last frame,
uploaded>]` (leg 0: the first scene's still URL). **Pass only that one frame, no last
frame** (a last frame forces a pull-back). The bolded clauses are the motion-handoff
contract; keep them verbatim, the mid-leg move is where the expression goes.

```
Single continuous cinematic camera move, no cuts. **Continue the same slow, steady
forward glide.** [MID-LEG MOVE — optional, from the library below.] The camera moves
into [SCENE i] toward [FOCAL POINT]. **In the final second, settle back into a slow,
steady forward glide toward [the doorway / opening / direction of the next scene].**
[STYLE tail + PALETTE]. Smooth, graceful, slow motion, subtle parallax. No text, no captions.
```

### Mid-leg move library (pick by concept; omit for a plain glide)

Reversals are safe *inside* a leg (it's one continuous render) — only a seam may never
reverse. That's why "ease back out" is fine mid-leg.

- **Half-orbit** (product, luxury): "sweeping in a slow half-orbit around [the hero
  object], keeping it centered, then continuing past it"
- **Crane-up reveal** (scale, atriums, campuses): "rising smoothly as the full scale of
  [the space] reveals below"
- **Low lateral track** (production lines, counters, shelves): "tracking low and level
  alongside [the line], foreground objects sliding past in parallax"
- **Push-in + ease back** (craft, detail): "pushing in close to [the craft moment] until
  it nearly fills the frame, then easing gently back out"
- **Rise-and-swoop** (travel, outdoors): "climbing in a gentle arc over [the terrain],
  then swooping down toward [the next focal point]"

After rendering each leg, **check its last frame** before generating the next: it should
read as a frame from a calm forward glide (no motion blur sideways, no half-finished
orbit). If it doesn't, re-roll this leg; a bad handoff frame poisons every leg after it.

## Dive-in clip prompt (Step 4)

`generation_type='image-to-video'`, `source_media_urls=[<the scene still URL>]` (the
solid-bg version), a single first frame with no last frame.

```
Single continuous cinematic camera move, no cuts. Begin high and far, looking down at the
whole [SECTION.subject] from outside like a tiny model. The camera slowly glides forward
and descends toward it, sweeping in toward [FOCAL POINT — the counter/the cauldrons/the
people], as if flying inside. As the camera pushes in, the roof and upper structure
gently lift and open away to reveal the warm interior. [STYLE tail: soft matte clay
diorama, tilt-shift miniature, warm light, [PALETTE]]. Smooth, graceful, slow motion,
subtle parallax. No text, no captions.
```

For scenes with no building to open (a field, a plaza, a road), replace the roof clause
with "the camera flies low across [the scene] toward [focal point]."

Params: `aspect_ratio='16:9'`, `duration=8` (legs can go longer, e.g. 10), `sound=false`
(you mute anyway; on Kling it also drops the audio surcharge), plus `resolution` per
`get_models` (the default `seedance-2-720p` renders 720p; 1080p needs the standard tier's
1080p resolution). Read the model's real `duration_options` + `resolution` from `get_models`
rather than assuming. Same params for architecture-A legs.

## Connector clip prompt (Step 5)

`generation_type='image-to-video'`, `source_media_urls=[<dive_i LAST frame>, <dive_{i+1}
FIRST frame>]` (order = first then last). Both extracted from the RENDERED videos (not the
stills) and **uploaded** first (SKILL Step 5 / pipeline §4), since `source_media_urls` needs
URLs.

```
Single continuous smooth camera movement, no cuts, no pulling back. Glide gently and
steadily FORWARD through the connected miniature world, flowing from [SCENE i] across to
[SCENE i+1], one seamless continuous motion. [STYLE tail + PALETTE]. Smooth graceful slow
motion. No text, no captions.
```

**Do NOT prompt the connector to "pull up", "rise into the sky", "arrive above", or fly to
a map view.** That camera *reversal* (dive pushes IN, connector yanks UP and OUT) is the #1
cause of a visible jump when the page is scrubbed, even with frame-perfect seams. Keep every
connector a low, continuous FORWARD glide between the two real boundary frames — the seams
already guarantee continuity, the prompt just has to not reverse. (Field-proven: the gentle
glide reads as one shot; the aerial-hop wording reads as a rewind.)

For the last connector into a hero-product finale: "…keep gliding forward and the world
gently resolves toward a single giant [PRODUCT] floating in soft [BG] space, arriving in
front of it."

Params: `aspect_ratio='16:9'`, `sound=false`, plus `resolution` per `get_models`. Duration:
5s is the floor — **use a LONGER duration (7–10s) when the transition travels a long distance
or needs to expand the scene**, so the move stays slow and continuous; a big move rushed into
5s reads as a jump. Connectors need a **last frame**, so the model must accept two input
images (`max_input_images >= 2`); use a Seedance 2 roster model (Step 4).

## Copy per section (for the engine config)

- `eyebrow` — 2–4 words, uppercase feel (a value-prop label).
- `title` — 3–6 words, the beat's headline. First section = the site's hero line; last =
  the payoff + it carries the CTA.
- `body` — one sentence, plain-spoken, from the visitor's side.
- `tags` — 0–3 short proof chips (e.g. "Fresh-cooked", "30-min delivery").
