# Marketplaces (Claude Code & Cursor)

Skills are the primary plugin. Source bundles live in [../skills/](../skills/); a **full copy** is kept under [../plugins/ai-dlc-skills/skills/](../plugins/ai-dlc-skills/skills/) so each plugin bundle is self-contained. After editing `skills/`, run `./scripts/sync-plugin-skills.sh` before commit.

## Cursor team marketplace

**Cursor** indexes a **`.cursor-plugin/marketplace.json`** at the repository root (not `.claude-plugin/`). Use the multi-plugin layout Cursor expects: **`metadata.pluginRoot`** points at the `plugins/` directory, and each plugin uses a **short** `source` name (e.g. `ai-dlc-skills`, not `./plugins/...`).

| File | Role |
|------|------|
| [`.cursor-plugin/marketplace.json`](../.cursor-plugin/marketplace.json) | Team marketplace catalog |
| [`plugins/ai-dlc-skills/.cursor-plugin/plugin.json`](../plugins/ai-dlc-skills/.cursor-plugin/plugin.json) | Per-plugin manifest for Cursor |

**Claude Code** keeps using [`.claude-plugin/marketplace.json`](../.claude-plugin/marketplace.json) and [`plugins/ai-dlc-skills/.claude-plugin/plugin.json`](../plugins/ai-dlc-skills/.claude-plugin/plugin.json) — same `skills/` tree under `plugins/ai-dlc-skills/skills/`.

If the dashboard shows **“No plugins found”** or a misleading connection error, confirm these paths exist on **`main`** and use **Refresh** on the marketplace (or disconnect/re-add the repo). The issue is often missing manifests, not VPN.

## Claude Code marketplace

Claude Code copies plugins to a cache and **does not support** content outside the plugin directory (see [Discover plugins — Troubleshooting](https://code.claude.com/docs/en/discover-plugins#troubleshooting)).

## Use from your local repo

```bash
/plugin marketplace add /path/to/AI-DLC
/plugin install ai-dlc-skills@ai-dlc
```

Replace `/path/to/AI-DLC` with your clone path (e.g. `~/GitHub/AI-DLC`).

## Share with others

Push the repo to GitHub. Others can run:

```bash
/plugin marketplace add queen-of-code/AI-DLC
/plugin install ai-dlc-skills@ai-dlc
```

Nothing is submitted to a central Anthropic store; your repo is the marketplace.

## Catalog and plugin

- **Claude catalog:** [../.claude-plugin/marketplace.json](../.claude-plugin/marketplace.json)
- **Cursor catalog:** [../.cursor-plugin/marketplace.json](../.cursor-plugin/marketplace.json)
- **Plugin folder:** [../plugins/ai-dlc-skills/](../plugins/ai-dlc-skills/) — `skills/` copy plus `.claude-plugin/` and `.cursor-plugin/` manifests

After installing, skills appear as `ai-dlc-skills:skill-name` (e.g. `/ai-dlc-skills:architecture`).
