---
name: project-pulse
description: Audit every repo belonging to a multi-repo project by pulling live evidence via `gh` (no local checkout required) in parallel, one agent per repo, then assemble a single lavish-axi dashboard covering stack & integrations, shipped features, next up (open issues/PRs), tech recommendations, and urgent/needs-review items per repo — every claim cited to a file path or PR/issue number, never asserted from a title alone. Use this whenever the user wants a project status check, repo/engineering health audit, "what's the state of X", a release-readiness check, a way to track or share progress across a project's repos, or a consolidated view across several sibling GitHub repos — even if they don't say "dashboard" or "audit" explicitly. Also trigger when the user references multiple repos under one org/project and wants one place to see all of them.
---

# project-pulse

Turns a scattered multi-repo project into one evidence-backed status dashboard. The
value isn't just per-repo detail — it's putting all repos side by side so contradictions
and dependencies between them become visible (one repo says a decision is "locked" while
a sibling repo's own code says it's still open; a frontend feature is already unblocked
because the backend endpoint it needed shipped last week). A single-repo audit never
catches that kind of thing.

## Before you start: ask, don't guess

Two things materially change how this runs — get them wrong and either the audit hallucinates
content or wastes a round-trip on a dead approach. Ask if either is unclear from context:

1. **Which repos, under which org/owner.** Look first for a `CLAUDE.md`/`AGENTS.md`/README
   "repo layout" section naming the sibling repos and their GitHub org (e.g. this project's
   own docs might say `kwail-tech/appname-{backend,frontend,infra}`). If nothing documents
   this, run `gh repo list <org>` and confirm the relevant subset with the user — repo lists
   routinely include unrelated side projects, forks, and archived repos that don't belong in
   the audit.
2. **Evidence source: `gh` only, or a local checkout too.** Default to pulling everything via
   `gh api`, `gh pr list`, `gh issue list`, etc. against GitHub directly, with no clone — this
   works from anywhere and never risks stale local state. Only use a local checkout if the user
   asks for it or the repos are already checked out in the working directory and the user
   confirms that's the copy to use.

## Workflow

### 1. Gather background context once, up front

Before dispatching anything, read whatever the project has that describes its own
architecture — root `CLAUDE.md`/`AGENTS.md`, `docs/ARCHITECTURE.md`, ADRs, a README. Pull
out: the expected tech stack per repo, known integrations, and any architectural rules the
code is supposed to follow (e.g. "payments provider is locked to X", "originals never
public", "webhook-only entitlement"). You'll hand this to each repo's agent as **claims to
verify against real code, not facts to assume true** — the whole point of the audit is
catching where reality has drifted from the docs, not restating the docs.

### 2. Dispatch one agent per repo, in parallel

Launch all of them in the same turn (Agent tool, `general-purpose`, no `fork` — these need
fresh context, not yours). Each agent gets:

- Which repo, and that it must use `gh` only (or a confirmed local path) — never fabricate
  file contents or PR history.
- The relevant slice of background context from step 1, framed as unverified claims.
- The fixed report format below, with the citation rule spelled out explicitly: **every
  claim in every section must end with a file path or a PR/issue number in parens; if it
  can't be verified, say "unverified" rather than asserting it.**
- A closing freshness line: latest commit SHA + date via `gh api repos/<org>/<repo>/commits?per_page=1`.

**Fixed per-repo report format** (keep this identical across repos so cards line up later):

1. **Stack & integrations** — actual dependencies/services wired into the code, with
   file-level evidence, not guesses from a package name.
2. **Shipped** — features demonstrably live: real code *and* merged PR history corroborating
   it landed. A file existing is not evidence of a shipped feature — a stub page or a
   mocked data layer looks identical to the real thing until you check both.
3. **Next up** — open issues/PRs, prioritized by what blocks a real release vs. what's
   polish.
4. **Tech recommendations** — concrete, evidenced gaps (missing tests on the paths that
   matter, stale deps, absent monitoring, security gaps) — each with a reason, never a vibe.
5. **Urgent / needs review** — anything risky, blocked on a decision only a human can make,
   or security/compliance-sensitive. Push agents to actually look for this rather than
   defaulting to "nothing found" — check auth/role enforcement is real and not just UI-level,
   check money-handling and biometric/PII paths for missing authorization, check for
   TODO/FIXME near payment or auth code.

### 3. Synthesize the cross-repo rollup

This is the step a single-repo audit can't do, so don't skip it. Read all reports together
and look specifically for:

- **Contradictions between repos** — one repo's docs/code assert a decision is settled while
  another repo's code (or its own docs) treats it as open.
- **Resolved dependencies** — something you'd expect to be a cross-repo blocker (frontend
  waiting on a backend feature) that turns out to already be shipped on both sides. Worth
  calling out explicitly — it's good news the user shouldn't have to re-derive.
- **Stale tracking issues** — an issue still open after the PR that closes it already merged.
  This is a process-hygiene finding, not a code finding, but it's real and easy to miss.
- Classify each rollup item: **Urgent** (needs a human decision, or security/compliance-sensitive),
  **Watch** (real, not blocking), **Resolved** (suspected blocker, actually fine), or
  **Hygiene** (bookkeeping only).

### 4. Build the dashboard

Load the `lavish` skill, then its `table` and `comparison` playbooks — the rollup is a
table, and framing "resolved vs. still-open" dependencies benefits from comparison framing.
Structure:

- **Top**: a release-readiness rollup — the cross-repo table from step 3, a freshness
  timestamp, and a stat row (repos audited, total open issues, total open PRs, urgent-item
  count).
- **One card per repo**, each holding the same 5 sections from step 2 — tabs or an accordion
  work well so the page stays scannable instead of one long scroll per repo.
- **Design system**: follow lavish's own priority order — a design direction the user named,
  then the *audited* project's own documented brand/theme (not your current working
  directory's), then the DaisyUI CDN fallback only if neither exists. If the project has a
  documented color/theme identity, reflect it even in the fallback rather than shipping
  default DaisyUI colors — it should look like *their* dashboard, not a generic one.
- Publish with `lavish-axi`, then poll for feedback per its normal workflow.

## Lessons that shaped this skill

- **"Shipped" needs two kinds of evidence, not one.** An agent that only reads code will
  report a hand-built mock data layer as a real feature. An agent that only reads PR titles
  will miss that a merged PR actually shipped something different than its title claims.
  Require both: point to the code *and* the PR that landed it.
- **Contradictions between repos are the headline finding, not a footnote.** In practice
  these have been the most actionable items in every rollup — a payments decision "locked"
  in one repo's docs while a sibling repo's own UI says it's still undecided, or a security
  control (role-based auth) that one repo assumes lives in another repo, which turns out
  not to enforce it either. Actively hunt for these when synthesizing step 3.
- **Don't let stale issues read as open work.** An issue that's still open after its fixing
  PR merged isn't a real gap — flag it as hygiene, not as a blocker, or the rollup
  overstates how much is left to do.
- **If a previous project-pulse dashboard has a live feedback poll running**, don't silently
  stack a second one — either resume/end the old one or say explicitly that a new one is
  starting.
