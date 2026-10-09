# wals.pro Ai 4 weclapp

Installable, auto-updating Agent Skills that teach your AI assistant proven weclapp ERP workflows. This repository is a plugin marketplace in both the Claude (`.claude-plugin/`) and the Codex (`.agents/plugins/`) catalog format.

## Install in Claude Code

```
/plugin marketplace add Wals-pro/weclapp-skills
/plugin install walspro-ai-4-weclapp@weclapp-skills
```

## Install in Claude (web), Claude Desktop, or Cowork

Open **Customize → Plugins → Add marketplace**, paste this repository's URL, then install **wals.pro Ai 4 weclapp**. Turn on auto-update to receive new skill versions automatically.

## Install in Codex (CLI or app)

```
codex plugin marketplace add Wals-pro/weclapp-skills
codex plugin add walspro-ai-4-weclapp@weclapp-skills
```

## Install in ChatGPT (Business and Enterprise)

A workspace admin opens **Workspace settings → Plugins → Add → Import marketplace**, enters this repository's URL as Source (leave Path empty) and sets the installation policy. ChatGPT syncs the marketplace daily; **Sync now** pulls updates immediately.

Ships one plugin, **walspro-ai-4-weclapp**, bundling 19 skills (version 0.7.1).

The skills drive the wals.pro AI MCP server at `https://mcp.ai.wals.pro/v1/mcp`. The plugin bundles this remote MCP connection. Install, authorize the connection in the client and select your weclapp tenant. No API key is shipped or requested by the skills.

## Existing installations

The former `weclapp@weclapp-skills` identifier is now `walspro-ai-4-weclapp@weclapp-skills`. Install the new plugin and check the connection before disabling the old plugin yourself. Nothing is automatically uninstalled. The repository subdirectory `plugins/weclapp` remains stable; it is a path, not the plugin name.

Clients with an existing connection on the same MCP URL may report a duplicate connection during installation. Review the client connection settings and keep the authorized connection you intend to use; never remove a working connection automatically.

> Generated from the private weclapp-mcp repository. Do not edit here — changes are overwritten on the next release.
