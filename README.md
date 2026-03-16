# Supaspec Builder Skill

An [Agent Skill](https://agentskills.io) for spec-driven development and documentation with [Supaspec](https://supaspec.dev).

Teaches AI coding agents to build from Supaspec specs, self-document every change, and log ongoing work as daily changelogs — so any future agent can reproduce the full implementation.

## Install

```bash
npx skills add super-full-stack/supaspec-builder
```

Works with Claude Code, Cursor, Codex, GitHub Copilot, and any agent that supports the Agent Skills standard.

## Requires

[Supaspec MCP server](https://supaspec.dev) connected to your agent.

## What it does

**Building from spec (Part 1-2):**
- Reads specs from Supaspec before coding
- Tracks implementation status (approved → implementing → implemented)
- Self-documents bug fixes in a `bugreports` section
- Self-documents features in daily `features-{date}` sections

**Changelog mode (Part 3):**
- Activated when user says "log changes" or "use changelog mode"
- Logs every code change to daily per-agent `changelog-{date}-{agent}` sections
- Captures what changed, why, how, and which files were affected
- Uses statuses: `wip` for active logs, `implemented` for compacted summaries
- Suggests compacting the day's entries into a cohesive summary at end of session
