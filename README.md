# Martech Agent Workspace (Claude Code CLI)

A delegation workspace for a martech manager: one repo you `cd` into, where Claude Code acts as the
orchestrator, specialist subagents own each system (Marketo, Salesforce, Databricks, Jira/Confluence,
Slack/Google Workspace, browser QA), skills encode your repeatable procedures, hooks enforce safety,
and scheduled jobs run the recurring work while you're in meetings.

```
                  you (terminal: `claude`, `claude agents`)
                                   │
          Claude Code main session = orchestrator (CLAUDE.md + AGENTS.md)
      ┌───────────┬────────────┬──────────┼───────────┬───────────┬───────────┐
 marketo-ops  sfdc-analyst  databricks-  data-      jira-pm   comms-      qa-browser
   (MCP)        (MCP)       analyst     reconciler   (MCP)    drafter    (Playwright)
                             (MCP)       (3 MCPs)              (MCP)
      └───────────── skills: /campaign-launch /weekly-funnel-report /daily-brief ... ─────┘
   hooks: ask-before-write · block destructive/bulk · SELECT-only SQL · audit log · Mac notifications
   schedules: /loop (in-session) · agent view (background) · launchd headless jobs · CI (optional)
```

## File map

| Path | What it is |
|------|------------|
| `AGENTS.md` | Tool-agnostic context and guardrails (also readable by Gemini CLI / Codex) |
| `CLAUDE.md` | Imports AGENTS.md; adds the delegation map, skill list, and safety notes |
| `.mcp.json` | Gateway MCP servers (slack, atlassian, gworkspace, marketo, salesforce, databricks) |
| `.claude/settings.json` | Permissions, hooks, env (committed; team-shareable) |
| `.claude/agents/*.md` | 7 specialist subagents (Markdown + YAML frontmatter) |
| `.claude/skills/*/SKILL.md` | 9 skills: 8 workflows + 1 background-knowledge skill |
| `.claude/hooks/*.sh` | Write guard, read-only SQL guard, audit log, macOS notifications |
| `.claude/rules/` | Data-handling rule (always) and SQL style rule (path-scoped) |
| `.claude/loop.md` | Default prompt for a bare `/loop` (ops sweep) |
| `automations/` | Headless job runner, launchd schedules, container runner, CI example, shell aliases |
| `interop/gemini/` | Example Gemini CLI command (TOML) that reuses AGENTS.md |

## Setup (about 30 minutes)

1. **Prereqs:** `brew install jq node` (node provides `npx` for Playwright MCP). Update Claude Code:
   `claude update`. Several features used here (agent view, `--permission-prompts`, AGENTS.md
   loading) need a recent 2.1.2xx build.
2. **Place the repo:** `~/work/martech-agent-workspace`, `git init`, commit.
3. **Gateway endpoints:** `cp .env.example .env`, set `MCP_GATEWAY_URL`, then `export` it in your
   shell profile too (`.mcp.json` expands `${MCP_GATEWAY_URL}`). If your gateway gives per-server
   URLs that don't follow `/<name>/mcp`, edit `.mcp.json` directly. If it needs a header token, add
   `"headers": {"Authorization": "Bearer ${MCP_GATEWAY_TOKEN}"}` to each server.
4. **Authenticate:** start `claude` in the repo, accept workspace trust, run `/mcp`, and sign in to
   each server (or `claude mcp login <server>` from the shell).
5. **Match server names:** the whole kit assumes the names in `.mcp.json`. If your admins
   pre-configured servers with different names, rename them there *or* find-and-replace
   `mcp__marketo` etc. across `.claude/`.
6. **Tune the write guard:** in `/mcp`, look at each server's tool names. Adjust the verb lists in
   `.claude/hooks/_classify.sh` so reads pass, drafts pass, and writes ask. Test:
   `echo '{"tool_name":"mcp__marketo__cloneProgram","tool_input":{}}' | .claude/hooks/guard-writes.sh`
7. **Fill in your org's facts:** every `TODO` in `.claude/skills/martech-context/references/` and
   the "Org conventions" section of `AGENTS.md`. This is the single biggest quality lever.
8. **Shell aliases:** add `source ~/work/martech-agent-workspace/automations/aliases.zsh` to `~/.zshrc`.
9. **Smoke test:** `mops` then ask: "Use sfdc-analyst to count leads created yesterday" and
   "/program-health-check". Confirm the audit log with `maudit`.
10. **Schedules (optional, after a week of interactive use):** `automations/launchd/install.sh`.

## How to work day to day

**Home base.** Keep one terminal tab on `magents` (agent view). It lists every background session
with its state (working / needs input / done). Keep a second tab for a foreground `mops` session.
Press `←` on an empty prompt in any session to jump to agent view; `Space` on a row to peek and reply.

**Morning.** `/daily-brief` (or let launchd produce it at 7:40 and read it with `mlogs 1`). The brief
ends with ready-to-run delegation prompts; paste each into agent view to dispatch it.

**During the day.** Start a sweep in your foreground session: `/loop 45m` (uses `.claude/loop.md`), so
health checks and new requests surface between meetings. It stays quiet when nothing changed.

**Delegating.** Three levels, from lightest to heaviest:
- Inside a session: "Have marketo-ops and sfdc-analyst check X in parallel" (subagents, results return to you).
- Background session: `mbg "reconcile September MQLs across Marketo, SFDC and Databricks"` or type it
  in agent view. It runs without a terminal and pings you (macOS notification) when it needs input.
- Specialist as its own session: `mspec marketo-ops "clone the webinar template for MOPS-412"`, or in
  agent view type `@marketo-ops clone the webinar template for MOPS-412`.

**Approvals.** Every write asks you, with the reason from the hook. External-facing actions are
labeled `EXTERNAL-FACING`. Destructive or bulk (>50 items) Marketo/SFDC operations are denied outright
and turned into a change plan in `workspace/`.

## Ten workflows to start with

1. **Campaign launch from a ticket:** `/campaign-launch MOPS-412`. Five phases with three approval
   checkpoints: brief → build in test → QA (LP, form, emails) → go-live prep → 24h post-launch check.
2. **Weekly funnel report:** `/weekly-funnel-report` Monday mornings. Parallel pulls from Databricks,
   SFDC, and Marketo, auto-reconciliation if MQLs disagree by >2%, Google Doc, Slack summary draft.
3. **Intake triage twice a day:** `/intake-triage` classifies Jira + Slack asks, sizes them, drafts
   "missing info" replies, and offers to ticket Slack-only requests.
4. **Continuous health monitoring:** `/loop 1h /program-health-check` (in session) or the launchd job.
   Runs inside a forked marketo-ops context so your main conversation stays clean.
5. **Form end-to-end test:** `/form-qa https://go.example.com/ai-ops-webinar "2026-10_Webinar_NA_AI-Ops"`
   verifies page → form → Marketo activity → program status → SFDC sync.
6. **Reconciliation on demand:** "Why does the dashboard show 1,240 MQLs but SFDC shows 1,105 for
   September?" The orchestrator routes to data-reconciler, which classifies the gap with record IDs.
7. **Routing/SLA audit:** `/lead-routing-audit 30` before a sales-marketing SLA review.
8. **Meeting → tickets:** `/meeting-to-actions <Google Doc URL>` right after a planning meeting.
9. **Stakeholder update:** "Draft my monthly martech update for the CMO from this month's closed MOPS
   tickets and the last 4 weekly funnel reports." comms-drafter writes a Gmail draft.
10. **Parallel research:** dispatch three background sessions from agent view, e.g. audit unused
    Marketo programs older than 18 months, list SFDC fields with <5% fill rate on Lead, and find
    Databricks tables with no updates in 30 days. Each writes a report to `workspace/`.

## Long-running automation options

| Option | Runs where | Survives closing the terminal | Best for |
|--------|-----------|-------------------------------|----------|
| `/loop 45m ...` | Your open session | No (restored on `--resume`; recurring tasks expire after 7 days) | Sweeps while you work |
| `/goal <condition>` | Your open session | No | "Keep going until the build passes QA" |
| Agent view / `claude --bg` | Local background service | Yes (not across shutdown) | Delegated tasks you check on later |
| `automations/launchd` + `claude -p` | Your Mac, on a schedule | Yes; missed runs fire on wake | Daily brief, triage, weekly report |
| Claude Code Desktop scheduled tasks | Your Mac | Yes | Same as launchd, set up through the Desktop UI instead of plists |
| Cloud routines (`/schedule`) | Anthropic cloud | Yes, machine can be off | Only if the cloud can reach your MCP gateway (usually it can't) |
| GitHub Actions (`automations/github`) | Self-hosted runner | Yes | Team-visible scheduled reports; needs gateway service credentials |
| Container (`automations/container`) | Rancher Desktop | Yes | Reproducible toolchain; parity with CI |

Headless jobs run with `--permission-mode dontAsk --permission-prompts none`: anything that would
ask you is denied. Combined with the write guard, scheduled jobs can read every system and write
files, but cannot post, send, activate, or change a system of record. Job output lands in
`~/Library/Logs/claude-jobs/` (`.md` result, `.json` with cost and denials) with a macOS notification.

If launchd jobs fail to authenticate (keychain not available to the job), run `claude setup-token`
and put the token in `.env` as `CLAUDE_CODE_OAUTH_TOKEN`.

## Customizing and growing it

- **New skill:** when you paste the same instructions twice, ask Claude: "Turn what we just did into
  a project skill in .claude/skills/". Side-effectful skills get `disable-model-invocation: true`
  (only you can trigger them). Skills you want in `/loop` or scheduled prompts must stay model-invocable.
- **New specialist:** copy an agent file, narrow its `tools`, and write a one-line `description` that
  says exactly when to use it. Keep descriptions short; they're loaded every session.
- **Agent memory:** `marketo-ops`, `databricks-analyst`, and `data-reconciler` have `memory: project`,
  so they accumulate template IDs, table grains, and known data issues in `.claude/agent-memory/`.
  Review and commit that folder occasionally.
- **Context hygiene:** run `/context` to see what's loaded, `/doctor` for a setup checkup, and
  `/skill-doctor` to find skills you never use.
- **Keep MCP tools out of the main context (advanced):** move a server from `.mcp.json` into a
  specialist's `mcpServers:` frontmatter (as done for Playwright in `qa-browser.md`). Only that
  subagent connects to it.
- **Team rollout:** once stable, package `.claude/` as a plugin in an internal marketplace so
  colleagues install it with `/plugin install`.

## Where Gemini and Google Workspace Studio fit

- Claude Code reads `AGENTS.md` natively, and Gemini CLI can be pointed at the same file as its
  context file, so both tools share one source of conventions and guardrails. `interop/gemini/`
  shows a Gemini CLI custom command, which is where TOML is used.
- Google Workspace Studio is useful as a Workspace-native trigger layer (form responses, labeled
  emails, Sheet changes) that posts into a Slack channel or creates a Jira request. The `/loop` sweep
  and `/intake-triage` then pick those up, so the heavy multi-system work stays in this workspace.

## Safety model summary

1. **Context** (AGENTS.md, rules): tells agents the policy. Helpful, not enforced.
2. **Tool scoping** (agent `tools:`): e.g. comms-drafter can't touch Marketo; analysts can't edit files outside workspace/.
3. **Hooks** (enforced by the client regardless of what the model decides): ask on writes, deny destructive/bulk, SELECT-only SQL.
4. **Headless mode**: every "ask" becomes "deny".
5. **Audit**: `~/.claude/martech-audit/YYYY-MM-DD.jsonl` records every non-read MCP call (`maudit`).

Content that agents read from email, Slack, tickets, or web pages is treated as data; the data-handling
rule tells agents to surface embedded instructions rather than act on them.

## Troubleshooting

- A subagent doesn't show up: check the frontmatter parses (`claude plugin validate .claude/agents`)
  and that `name` and `description` exist.
- A skill doesn't trigger: invoke it directly with `/name`; sharpen its `description`.
- Hooks don't fire: run `/hooks`; confirm `chmod +x .claude/hooks/*.sh` and that `jq` is on PATH.
- MCP server missing in headless runs: `claude -p` loads `.mcp.json`, but OAuth tokens must already
  exist from an interactive `/mcp` login on the same machine.
- Docs: https://code.claude.com/docs (subagents, skills, hooks, agent view, scheduled tasks, headless).
