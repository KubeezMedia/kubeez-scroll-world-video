# 🍔 Kubeez Scroll-World-Video — *Smash & Co.* demo

> A scroll‑scrubbed **"fly through the world"** landing page. As you scroll, a pre‑rendered
> camera glides forward through a miniature clay **burger world** — market → grill → counter →
> the burger — as one continuous shot, no cuts. **Every frame was generated with
> [Kubeez](https://kubeez.com).**

### ▶️ **[Live demo → kubeezmedia.github.io/kubeez-scroll-world-video](https://kubeezmedia.github.io/kubeez-scroll-world-video/)**

![Smash & Co. — scroll-world-video showcase](img/showcase.gif)

<sub>▶ **[Watch the full 20-second video](https://kubeezmedia.github.io/kubeez-scroll-world-video/img/showcase.mp4)** · or scroll the **[live demo](https://kubeezmedia.github.io/kubeez-scroll-world-video/)** yourself.</sub>

---

## What it is

Scroll doesn't move a page here — it **drives a camera**. Each scene is a short pre‑rendered
video; the engine maps your scroll position to the video's `currentTime`, so scrolling scrubs
the camera forward through the world. Between scenes, dedicated **transition clips** glide the
camera from one place to the next, so the whole journey reads as a single continuous flight.

| The world | Landing (Market) | The Counter | The Burger |
|---|---|---|---|
| ![](img/world.png) | ![](img/landing.png) | ![](img/counter.png) | ![](img/finale.png) |

---

## How it was built (the tutorial)

The entire thing — stills, motion, everything — was generated on the **Kubeez** platform. No
external tools. Here's the pipeline:

### 1. Stills — one coherent world
- A **master world still** (`gpt-image-2`, text‑to‑image) sets the palette, angle and light.
- Each **scene still** (Market, Grill, Counter, Hero Burger) is derived from the master with
  **image‑to‑image** so they all share one look.

### 2. Dive clips — fly into each scene
- Each scene still is animated with **Seedance 2 Fast (720p)**, `image-to-video`: the camera
  begins high and wide, then glides forward and descends into the scene.

### 3. Transitions — the part that makes it feel continuous ✨
This is the technique that matters. For each pair of consecutive dives:

```
transition.firstFrame = the ACTUAL last frame of dive N   (extracted from the rendered mp4)
transition.lastFrame  = the ACTUAL first frame of dive N+1 (extracted from the rendered mp4)
```

Feed those two real frames to **Seedance 2** as keyframes with a **gentle forward‑glide**
prompt, and it fills the in‑between. Because the transition *starts* on dive N's real last
frame and *ends* on dive N+1's real first frame, **the seams are frame‑exact** — the engine
plays `dive1 → transition1 → dive2 …`, so scrolling off the end of a dive kicks off its
transition with no visible cut.

> **The one rule:** keep the transition a **low, continuous forward glide.** Do *not* prompt
> it to "pull up / rise into the sky / fly to a map view" — that camera reversal reads as a
> jarring *jump* when scrubbed, even with perfect seams. If two scenes are far apart, give the
> transition a **longer duration** (7–10s) so the move stays slow and smooth.

### 4. Encode for smooth scrubbing
- Native resolution, `crf 20`, small GOP (`-g 8`), light sharpen, no audio, faststart.
- The engine loads each clip as a **Blob** and scrubs `currentTime`, so smooth seeking works
  even on static hosts (like GitHub Pages) that don't serve HTTP byte ranges.

### 5. Wire the engine
The whole page is one config object passed to a portable, dependency‑free vanilla‑JS engine:

```html
<script src="assets/scrub-engine.js"></script>
<script>
  mountKubeezWorld(document.getElementById('world'), {
    brand: { name: 'Smash & Co.' },
    sections: [ /* market, grill, counter, burger — copy + accent per scene */ ],
    connectors: [ 'assets/vid/conn1.mp4', 'assets/vid/conn2.mp4', 'assets/vid/conn3.mp4' ],
  });
</script>
```

The engine handles scroll→time mapping, blob loading, lazy prefetch, frame‑matched crossfades,
pinned per‑section copy, the route rail, `prefers-reduced-motion`, and phone hardening.

---

## The clips

| Section | Model | Notes |
|---|---|---|
| Market / Grill / Counter / Burger dives | Seedance 2 Fast · 720p | `image-to-video` from each scene still |
| Transitions (×3) | Seedance 2 Fast · 720p | keyframed from real boundary frames; conn3 is 7s |
| Stills | gpt‑image‑2 | master + image‑to‑image scenes |

Roughly ~1,150 Kubeez credits for the full 720p build.

---

## Build your own — install the skill

This demo was produced by the **`scroll-world-video`** skill, which interviews you for your
topic + brand, generates the world, and wires this exact engine — for *any* business.

**Claude Code** (installs the plugin and auto-wires the Kubeez MCP):
```
/plugin marketplace add KubeezMedia/kubeez-scroll-world-video
/plugin install scroll-world-video@kubeez
/scroll-world-video
```

**Codex / Cursor / other agents:** point your agent at
[`skills/scroll-world-video/SKILL.md`](skills/scroll-world-video/SKILL.md) and connect the
Kubeez MCP — see [`AGENTS.md`](AGENTS.md).

> ⚡ **Powered by Kubeez.** The scroll engine and method are open, but the skill generates
> *only* through the [Kubeez](https://kubeez.com) platform (`mcp.kubeez.com/mcp`) — that's what
> turns your idea into a world. Sign in with a Kubeez account to run it.

## Run it locally

The engine fetches clips as blobs, so serve over HTTP (not `file://`):

```bash
python -m http.server 8080
# open http://localhost:8080
```

---

<sub>Made with [Kubeez](https://kubeez.com). All imagery and motion generated on the Kubeez platform.</sub>
