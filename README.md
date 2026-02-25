# Supaspec Builder Skill

   An [Agent Skill](https://agentskills.io) for spec-driven development with [Supaspec](https://supaspec.dev).

   Teaches AI coding agents to build from Supaspec specs and self-document every change as bugs or features — so any future agent can reproduce the full implementation.

   ## Install
```bash
   npx skills add super-full-stack/supaspec-builder
```

   Works with Claude Code, Cursor, Codex, GitHub Copilot, and any agent that supports the Agent Skills standard.

   ## Requires

   [Supaspec MCP server](https://supaspec.dev) connected to your agent.

   ## What it does

   - Reads specs from Supaspec before coding
   - Tracks implementation status (approved → implementing → implemented)
   - Self-documents bug fixes in a `bugreports` section
   - Self-documents features in daily `features-{date}` sections
   - Creates a living changelog any agent can read
```

4. **Submit to skills.sh** — go to https://skills.sh and follow their submission flow (it indexes from GitHub, so once the repo is public with a valid SKILL.md it should get picked up, or you can submit manually).

5. **Add to Supaspec docs** — wherever you document MCP setup, add:
```
   ## Recommended: Install the builder skill
   npx skills add super-full-stack/supaspec-builder
