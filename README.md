# weclapp Skills for Claude, Codex and ChatGPT

Installable, auto-updating Agent Skills that teach your AI assistant proven weclapp ERP workflows. This repository is a plugin marketplace in both the Claude (`.claude-plugin/`) and the Codex (`.agents/plugins/`) catalog format.

## Install in Claude Code

```
/plugin marketplace add Wals-pro/weclapp-skills
/plugin install weclapp@weclapp-skills
```

## Install in Claude (web), Claude Desktop, or Cowork

Open **Customize → Plugins → Add marketplace**, paste this repository's URL, then install the **weclapp** plugin. Turn on auto-update to receive new skill versions automatically.

## Install in Codex (CLI or app)

```
codex plugin marketplace add Wals-pro/weclapp-skills
codex plugin add weclapp@weclapp-skills
```

## Install in ChatGPT (Business and Enterprise)

A workspace admin opens **Workspace settings → Plugins → Add → Import marketplace**, enters this repository's URL as Source (leave Path empty) and sets the installation policy. ChatGPT syncs the marketplace daily; **Sync now** pulls updates immediately.

Ships one plugin, **weclapp**, bundling 18 skills (version 0.6.5).

The skills drive the wals.pro AI MCP server at `https://mcp.ai.wals.pro/v1/mcp`. Add that connection first (Claude: Connectors Directory or Customize → Connectors; Codex and ChatGPT: MCP settings), sign in and pick your weclapp tenant, then install this plugin.

> Generated from the private weclapp-mcp repository. Do not edit here — changes are overwritten on the next release.
