# scroll-world-video — for any AI coding agent

This repo is a portable skill for building a **scroll-scrubbed "fly through the world"**
landing page (see the [live demo](https://kubeezmedia.github.io/kubeez-scroll-world-video/)).
It works in **Claude Code** as an installable plugin, and in **Codex / Cursor / any agent**
as a plain skill you point the model at.

## Requirement (the important part)

This skill generates **exclusively through the Kubeez platform**. It has no other image/video
backend by design. To use it you must have the **Kubeez MCP server** connected:

- URL: `https://mcp.kubeez.com/mcp` (HTTP MCP, sign in with a Kubeez account at
  [kubeez.com](https://kubeez.com))
- The skill calls `mcp__kubeez__generate_media`, `get_models`, `estimate`, etc. Without the
  Kubeez MCP, it cannot generate a world.

The scroll engine (`assets/scrub-engine.js`, `mountKubeezWorld(...)`) and the whole method are
open — the **generation** is what runs on Kubeez.

## Use it

### Claude Code (plugin)
```
/plugin marketplace add KubeezMedia/kubeez-scroll-world-video
/plugin install scroll-world-video@kubeez
/scroll-world-video
```
The plugin auto-wires the Kubeez MCP server for you.

### Codex / Cursor / other agents
1. Make sure the Kubeez MCP (`https://mcp.kubeez.com/mcp`) is available to your agent.
2. Point the agent at [`skills/scroll-world-video/SKILL.md`](skills/scroll-world-video/SKILL.md)
   and tell it to follow that skill. The `references/` folder has the prompt templates,
   pipeline, and the scrub engine.

## What's in here

| Path | What |
|---|---|
| `skills/scroll-world-video/SKILL.md` | The skill — full instructions |
| `skills/scroll-world-video/references/` | Prompt templates, pipeline, scrub engine, knockout util |
| `.claude-plugin/` | Claude Code plugin + marketplace manifests |
| `index.html`, `assets/` | The live *Smash & Co.* burger demo (GitHub Pages) |

Built and maintained by [Kubeez](https://kubeez.com).
