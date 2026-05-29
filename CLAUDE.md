# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Project Overview

This is the **manada-skills** repository — the knowledge for creating, customizing, deploying and invoking Claude Code subagents ("the pack") across every scope and via the Claude Agent SDK.

**Repository**: https://github.com/milojarow/manada-skills

## Repository Structure

```
manada-skills/
├── .claude-plugin/          # marketplace.json + plugin.json
├── CLAUDE.md                # This file
├── README.md                # Project overview
├── LICENSE                  # MIT
├── evaluations/             # Trigger + findability scenarios
└── skills/
    └── manada/
        ├── SKILL.md          # Entry point (WHEN-to-use)
        └── reference/        # Depth: scopes, personalization, harness-vs-sdk, dispatching, creating-deploying
```

## The skill

### manada
Documents subagents end-to-end: the three scopes (plugin / project / user-global) and the headless Agent SDK; the two locks of scoping (location = who sees it, description = when it's chosen); per-agent customization (`model`, `tools`, `skills`, `memory`, `effort`, `color`); the plugin-agent security cap (`hooks`/`mcpServers`/`permissionMode` ignored) and how to lift it; harness (Agent tool) vs SDK (`query()`); subscription auth; and fan-out vs overhead.

## Skill Activation

Activates when the user is creating/configuring/deploying/invoking a subagent, choosing its scope, customizing it (model/tools/memory/etc.), wiring agents into a skill, working with the Claude Agent SDK, or deciding when parallel agents pay off.

## Updating this skill

After any session that discovers a new pattern, limit, or gotcha. Keep entries generic — patterns and examples, never client data, real IDs, or instance URLs. The git log of this repo is the diary.
