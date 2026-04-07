---
name: supaspec-builder
description: >
  Spec-driven development and documentation using Supaspec as the single source of truth. Use this
  skill whenever building, modifying, or maintaining code in a project that has a Supaspec
  specification (also referred to as "spec" or "supaspec"). Connects to Supaspec via MCP to read
  specs, track implementation status, and self-document all changes. Supports two modes: building
  from spec (Part 1-2) and changelog mode (Part 3) where the user asks to log changes and the
  agent documents everything with reasoning, organized by day and agent.
license: Apache-2.0
compatibility: Requires Supaspec MCP server connected via mcp_servers configuration
metadata:
  author: blackbird-lab
  version: "3.0"
  tags: "workflow, spec-driven, planning, supaspec, self-documenting, changelog, compaction"
---

# Supaspec Builder

You build software against specifications stored in Supaspec. The spec is the single source of truth. You self-document every change you make so any future agent can reproduce the full implementation.

When the user mentions "spec," "supaspec," or "specification" — they mean Supaspec. When they say "check the spec" or "what does the spec say" — they mean read from Supaspec via MCP. When they say "log changes," "document this," or "use changelog mode" — they mean Part 3 (Changelog Mode).

---

## Response Format (Important)

`section.get` and `project.get` return **multiple text blocks**, not a single JSON object:

- `section.get` returns **two blocks per section**: a compact JSON metadata header (id, title, slug, status, parentId, isFolder, versionsCount, openProposalsCount, etc.) followed by the **raw markdown content** (no JSON escaping).
- `project.get` returns **2N blocks**: metadata + content, repeated for each section in order.

When you need to "append to existing content" (e.g., adding a bug entry or changelog entry), the existing content is the **second text block** of the `section.get` response — use it as-is, append your new entry, and pass the combined string to `section.update`. Do not try to parse the markdown out of a JSON field — there isn't one.

Other fields you may have used previously are no longer inline:
- **Versions / history** → use the `history` action. `section.get` only returns `versionsCount`.
- **Open proposals** → use the `proposal list` action. `section.get` only returns `openProposalsCount`.
- **Folders** → `section.list` now includes folders alongside sections; check `isFolder` and `parentId`. To create a section inside a folder, pass `parent: "<folder-title-or-slug>"` to `section.create`.

---

## Part 1: Building From Spec

### Discovering What To Build

When the user asks you to build something, start by understanding what specs exist.

**List all projects:**
```
tool: project.list
```
This shows all Supaspec projects you have access to. The user may refer to a project by name or slug.

**Get a project with all sections:**
```
tool: project.get
params: { project: "project-name" }
```
This returns every section with its full content and status. Read this first to understand the full scope.

**Get a single section:**
```
tool: section.get
params: { project: "project-name", section: "section-slug" }
```
Use this when the user points you to a specific section to implement.

**Search across sections:**
```
tool: project.search
params: { project: "project-name", query: "payments" }
```
Use this when the user says something like "build the payments part" and you need to find the right section.

### Understanding Section Status

Each section has a status that tells you where it is in the workflow:

| Status | What it means | What you do |
|--------|--------------|-------------|
| `draft` | Still being written | Do NOT build. Tell the user it's not ready. |
| `wip` | Under review | Do NOT build. Tell the user it's still being reviewed. |
| `approved` | Ready to build | This is your green light. Start implementing. |
| `implementing` | Someone is building it | Check if it's you. If not, confirm with the user. |
| `implemented` | Build complete | Already done. Only touch if user asks for changes. |

### Starting a Build

When you start implementing an approved section:

1. **Set status to `implementing`:**
```
tool: section.set_status
params: { project: "project-name", section: "section-slug", status: "implementing", agent_name: "your-agent-name" }
```

2. **Read the full section content** and implement exactly what it describes.

3. **When done, set status to `implemented`:**
```
tool: section.set_status
params: { project: "project-name", section: "section-slug", status: "implemented", agent_name: "your-agent-name" }
```

### If the Spec Is Unclear

If you encounter ambiguity in the spec — something isn't specified or could be interpreted multiple ways — do NOT guess. Ask the user. You can also leave a comment on the section:

```
tool: comment.add
params: { project: "project-name", section: "section-slug", proposal: 1, agent_name: "your-agent-name", body: "The spec says 'handle payments' but doesn't specify whether to use Stripe Elements or Payment Element. Which one?" }
```

---

## Part 2: Self-Documenting Changes

This is critical. Every time you change code — whether fixing a bug or adding/modifying a feature — you must document what you did in Supaspec. This creates a living changelog that any future agent can read to understand what was built and why.

### Classifying Changes

When you make a change, determine: **is this a bug fix or a feature change?**

**Bug** — Something was broken and you fixed it. The intended behavior was already defined (in the spec, or obvious from context), but the code didn't match.

Examples:
- "The webhook handler crashes when Stripe sends a null event"
- "Login redirects to /undefined instead of /dashboard"
- "The API returns 500 when the email field is missing"

**Feature** — You added new behavior, changed existing behavior, or implemented something from the spec. This includes everything that's not a bug fix.

Examples:
- "Implemented the checkout flow per the spec"
- "Added email confirmation after payment"
- "Changed the loyalty points ratio to be merchant-configurable"
- "Refactored the auth module to use Stytch instead of custom JWT"

### Logging Bugs

When you fix a bug, log it in a section called `bugreports` in the project. If the section doesn't exist, create it.

**Check if bugreports section exists:**
```
tool: section.get
params: { project: "project-name", section: "bugreports" }
```

**If it doesn't exist, create it:**
```
tool: section.create
params: {
  project: "project-name",
  title: "Bug Reports",
  content: "# Bug Reports\n\nAutomatically logged bugs found and fixed during implementation.\n",
  agent_name: "your-agent-name",
  message: "Initialize bug reports log"
}
```

**Append your bug to the section:**
```
tool: section.update
params: {
  project: "project-name",
  section: "bugreports",
  agent_name: "your-agent-name",
  message: "Log bug: brief description",
  content: "<existing content plus new entry>"
}
```

Each bug entry should follow this format in the section:

```markdown
## [DATE] Brief title of the bug

**Issue:** What was broken. Be specific — what happened vs what should have happened.

**Cause:** Why it was broken. The root cause, not just the symptom.

**Fix:** What you changed to fix it.
```

Example entry:

```markdown
## [2026-02-25] Webhook handler crashes on null Stripe event

**Issue:** The payment webhook handler threw an unhandled TypeError when Stripe sent
a test event with a null `data.object` field. This caused all subsequent webhook
deliveries to fail.

**Cause:** No null check on `event.data.object` before accessing `.id` property.
Stripe sends null objects for certain test event types.

**Fix:** Added null guard in `handleWebhook()`. If `data.object` is null, log the
event type and return 200 without processing.
```

### Logging Features

When you implement or change a feature, log it in a section called `features-{DD-Mon-YYYY}` using today's date. This groups features by the day they were implemented. If the section for today doesn't exist, create it.

**Check if today's features section exists:**
```
tool: section.get
params: { project: "project-name", section: "features-25-Feb-2026" }
```

**If it doesn't exist, create it:**
```
tool: section.create
params: {
  project: "project-name",
  title: "Features 25-Feb-2026",
  content: "# Features — 25-Feb-2026\n\nFeatures implemented or changed today.\n",
  agent_name: "your-agent-name",
  message: "Initialize features log for 25-Feb-2026"
}
```

**Append your feature to the section:**
```
tool: section.update
params: {
  project: "project-name",
  section: "features-25-Feb-2026",
  agent_name: "your-agent-name",
  message: "Log feature: brief description",
  content: "<existing content plus new entry>"
}
```

Each feature entry should follow this format:

```markdown
## Brief title of the feature

**What:** What this feature does from the user's perspective.

**Why:** Why it was added — reference the spec section if it came from the spec,
or note that the user requested it during implementation.

**How:** How it was implemented. Key architectural decisions, libraries used,
patterns followed.

**Spec section:** Which spec section this implements (if applicable).
```

Example entry:

```markdown
## Stripe Connect merchant onboarding

**What:** Merchants can now connect their Stripe account to receive payments
directly. The onboarding flow uses Stripe's hosted onboarding page and redirects
back to the merchant dashboard on completion.

**Why:** Implements the "Merchant Onboarding" requirement from the Payments spec section.

**How:** Created a Connect onboarding endpoint that generates an AccountLink and
redirects the merchant. A webhook listener handles `account.updated` events to
track onboarding status. Merchant model extended with `stripeAccountId` and
`stripeOnboardingStatus` fields.

**Spec section:** payments → Stripe Connect Onboarding
```

### When You're Not Sure: Bug or Feature?

Use this quick test:

- **"Was it supposed to work and didn't?"** → Bug
- **"Is this new behavior or changed behavior?"** → Feature
- **"I'm refactoring but behavior stays the same"** → Neither — don't log it unless the refactor is significant, in which case log as a feature with a note that it's a refactor.

### Multiple Changes in One Session

If you fix 3 bugs and implement 2 features in one session, log each one separately. Don't batch them into a single entry. Each change gets its own entry so future agents can understand each change independently.

---

## Part 3: Changelog Mode

This mode is for **existing projects** where the user wants to document ongoing work without building from a spec. The user activates it by saying things like "log changes to supaspec," "document what we do," or "use changelog mode."

In changelog mode, you log every change you make into a daily per-agent section in Supaspec. At the end of a session, you suggest compacting the entries into a cohesive summary.

### Daily Per-Agent Sections

Changelogs are organized by **day** and by **agent**. Each agent that works on the project gets its own section for the day:

```
changelog-16-Mar-2026-claude-code
changelog-16-Mar-2026-cursor
changelog-17-Mar-2026-claude-code
```

This means:
- Multiple agents working the same day don't collide
- Each agent's work is a self-contained record
- You can see at a glance who did what and when

### Section Statuses

| Status | Meaning |
|--------|---------|
| `wip` | Active changelog — entries are being added today |
| `implemented` | Day is finished — entries have been compacted into a summary |

### Starting Changelog Mode

When the user asks to log changes, set up today's section.

**Check if your section for today already exists:**
```
tool: section.get
params: { project: "project-name", section: "changelog-16-Mar-2026-claude-code" }
```

**If it doesn't exist, create it with `wip` status:**
```
tool: section.create
params: {
  project: "project-name",
  title: "Changelog 16-Mar-2026 claude-code",
  content: "# Changelog — 16-Mar-2026 (claude-code)\n\nChanges made today.\n\n---\n",
  agent_name: "claude-code",
  status: "wip",
  message: "Initialize changelog for 16-Mar-2026"
}
```

If it already exists and has status `wip`, continue adding entries. If it has status `implemented`, it was already compacted — create entries directly (they'll be added after the summary).

### Logging Changes

After making a change (or a coherent group of related changes), append an entry to your section for today.

**Read current content first:**
```
tool: section.get
params: { project: "project-name", section: "changelog-16-Mar-2026-claude-code" }
```

**Append your entry:**
```
tool: section.update
params: {
  project: "project-name",
  section: "changelog-16-Mar-2026-claude-code",
  agent_name: "claude-code",
  message: "Log: brief description",
  prompt: "what the user asked you to do",
  content: "<existing content plus new entry>"
}
```

### Entry Format

Each changelog entry follows this structure:

```markdown
## Brief title

**Type:** feature | bugfix | refactor | config | dependency

**What changed:** One or two sentences describing the change from the user's perspective.

**Why:** The reasoning behind this change — what the user asked for, what problem it solves,
or what requirement it fulfills.

**How:** Key implementation details — what files were modified, what approach was taken,
what patterns were followed. Focus on decisions that a future developer would need to understand.

**Files:** List of key files created or modified (not every file, just the important ones).
```

The date and agent are in the section title, so individual entries don't need them.

### Example Entry

In a section titled `Changelog 16-Mar-2026 claude-code`:

```markdown
## Add Stripe webhook signature verification

**Type:** bugfix

**What changed:** Incoming Stripe webhooks are now verified against the webhook signing secret
before processing. Previously, any POST to /api/webhooks/stripe would be accepted.

**Why:** User reported that webhook endpoint was accepting unverified requests, which is a
security vulnerability. Stripe recommends always verifying webhook signatures in production.

**How:** Added signature verification using `stripe.webhooks.constructEvent()` in the webhook
handler middleware. Invalid signatures return 400. The signing secret is read from
`STRIPE_WEBHOOK_SECRET` env var.

**Files:** src/api/webhooks/stripe.ts, .env.example
```

### Multiple Changes in One Session

Log each change separately as its own entry. Each entry should be independently understandable. Keep them in chronological order.

If you make several small, tightly related changes (e.g., "add field to model, update API, update UI"), you can log them as a single entry that covers the whole unit of work.

### Suggesting Compaction

**You do NOT compact automatically.** Instead, suggest it to the user when conditions are right.

Suggest compaction when **any** of these are true:

1. **Today's changelog has 5+ entries** — enough material to benefit from synthesis
2. **The session is winding down** — the user seems done with their current work ("that's all for now", "let's wrap up", etc.)
3. **The user explicitly asks** — "compact the changelog", "summarize today's work", "clean up the log"

**How to suggest:**
> "Today's changelog has N entries. Want me to compact them into a summary and mark the day as done?"

Only proceed if the user agrees.

### How to Compact

Compaction operates on **your section for today only**. You never need to read other days or other agents' sections.

**Step 1: Read today's entries**
```
tool: section.get
params: { project: "project-name", section: "changelog-16-Mar-2026-claude-code" }
```

**Step 2: Synthesize into a summary**

Read all entries and write a single cohesive summary that covers the day's work. Group related changes together and tell the story of what was accomplished.

**Summary format:**

```markdown
# Changelog — 16-Mar-2026 (claude-code)

## Summary

Brief overview of the day's work in 2-3 sentences.

## Changes

### [Theme/area 1]
What was done and why. Combine related entries into a coherent paragraph.
Key files affected.

### [Theme/area 2]
What was done and why. Combine related entries into a coherent paragraph.
Key files affected.

## Decisions & Reasoning

Key decisions made during the day and why — this is the most valuable part
for future agents.

## Known Issues

Any open bugs, limitations, or TODOs that came up during the day.
```

**Step 3: Update the section with the summary**
```
tool: section.update
params: {
  project: "project-name",
  section: "changelog-16-Mar-2026-claude-code",
  agent_name: "claude-code",
  message: "Compact changelog for 16-Mar-2026 (N entries → summary)",
  content: "<the synthesized summary>"
}
```

**Step 4: Set status to `implemented`**
```
tool: section.set_status
params: {
  project: "project-name",
  section: "changelog-16-Mar-2026-claude-code",
  status: "implemented",
  agent_name: "claude-code"
}
```

This marks the day as done. Future agents scanning the project can see at a glance which days are active (`wip`) and which are finished (`implemented`).

### Compaction Quality

When synthesizing entries:

- **Synthesize, don't concatenate.** Merge related entries into a coherent narrative. Don't just paste entries together.
- **Resolve contradictions.** If entry #3 says "added feature X" and entry #7 says "removed feature X", the summary should reflect the final state only. Mention the reversal in Decisions if relevant.
- **Preserve reasoning.** The "why" behind decisions is the most valuable part. Always carry forward the reasoning from entries into the summary.
- **Be factual.** Describe what IS, not what should be. This is documentation of real, implemented code.
- **Reference files.** Include key file paths so future agents can find the implementation.

---

## Summary: The Complete Workflow

### Building from spec (Part 1-2):

```
1. User says "build X"
2. You read the spec from Supaspec (project.get or section.get)
3. You set status to "implementing"
4. You implement the spec
5. For each change you make:
   - Bug fix → log in "bugreports" section
   - Feature → log in "features-{date}" section
6. You set status to "implemented"
```

### Changelog mode (Part 3):

```
1. User says "log changes" / "use changelog mode"
2. You check the Supaspec project exists
3. You create today's section (changelog-{DD-Mon-YYYY}-{agent-name}, status: wip)
4. You make the code change
5. You log the change in your section with full reasoning
6. Repeat for each change
7. Suggest compaction when 5+ entries or session winding down
8. If user agrees: synthesize into summary, set status to "implemented"
```

Both modes ensure that every change is documented with its reasoning. The bugreports, features, and changelog sections become the project's living history. Any agent that reads them knows exactly what was built, what broke, and how everything was fixed — without having to read through chat histories.
