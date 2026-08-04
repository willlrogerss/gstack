# gstack development

Agent core: the things that bite, the things you cannot derive from the code, and where
everything else lives. Detail moved to `.claude/rules/` 2026-08-03 (context detox, Stage 5)
— **verbatim, nothing deleted**. Pre-split file: `git show HEAD~1:CLAUDE.md`.

## Platform-agnostic design

Skills must NEVER hardcode framework-specific commands, file patterns, or directory
structures. Instead:

1. **Read CLAUDE.md** for project-specific config (test commands, eval commands, etc.)
2. **If missing, AskUserQuestion** — let the user tell you or let gstack search the repo
3. **Persist the answer to CLAUDE.md** so we never have to ask again

This applies to test commands, eval commands, deploy commands, and any other
project-specific behavior. The project owns its config; gstack reads it.

## Dev symlink awareness

When developing gstack, `.claude/skills/gstack` may be a symlink back to this
working directory (gitignored). This means skill changes are **live immediately**,
great for rapid iteration, risky during big refactors where half-written skills
could break other Claude Code sessions using gstack concurrently.

**Check once per session:** Run `ls -la .claude/skills/gstack` to see if it's a
symlink or a real copy. If it's a symlink to your working directory, be aware that:
- Template changes + `bun run gen:skill-docs` immediately affect all gstack invocations
- Breaking changes to SKILL.md.tmpl files can break concurrent gstack sessions
- During large refactors, remove the symlink (`rm .claude/skills/gstack`) so the
  global install at `~/.claude/skills/gstack/` is used instead

**Prefix setting:** Setup creates real directories (not symlinks) at the top level
with a SKILL.md symlink inside (e.g., `qa/SKILL.md -> gstack/qa/SKILL.md`). This
ensures Claude discovers them as top-level skills, not nested under `gstack/`.
Names are either short (`qa`) or namespaced (`gstack-qa`), controlled by
`skill_prefix` in `~/.gstack/config.yaml`. Pass `--no-prefix` or `--prefix` to
skip the interactive prompt.

**Skill exclusion:** `skill_exclude` (comma-separated, NO spaces — `gstack-config get`
parses with `awk '{print $2}'`) leaves named skills unlinked. They stay installed here but
get no `~/.claude/skills` entry, so they cost nothing in session context. `gstack-relink`
tears down entries for excluded names and restores them when a name is removed.

**Note:** Vendoring gstack into a project's repo is deprecated. Use global install
+ `./setup --team` instead. See README.md for team mode instructions.

**For plan reviews:** When reviewing plans that modify skill templates or the
gen-skill-docs pipeline, consider whether the changes should be tested in isolation
before going live (especially if the user is actively using gstack in other windows).

**Upgrade migrations:** When a change modifies on-disk state (directory structure,
config format, stale files) in ways that could break existing user installs, add a
migration script to `gstack-upgrade/migrations/`. Read CONTRIBUTING.md's "Upgrade
migrations" section for the format and testing requirements. The upgrade skill runs
these automatically after `./setup` during `/gstack-upgrade`.

## Compiled binaries — NEVER commit browse/dist/ or design/dist/

The `browse/dist/` and `design/dist/` directories contain compiled Bun binaries
(`browse`, `find-browse`, `design`, ~58MB each). These are Mach-O arm64 only — they
do NOT work on Linux, Windows, or Intel Macs. The `./setup` script already builds
from source for every platform, so the checked-in binaries are redundant. They are
tracked by git due to a historical mistake and should eventually be removed with
`git rm --cached`.

**NEVER stage or commit these files.** They show up as modified in `git status`
because they're tracked despite `.gitignore` — ignore them. When staging files,
always use specific filenames (`git add file1 file2`) — never `git add .` or
`git add -A`, which will accidentally include the binaries.

## Redaction guard (PII / secrets / legal content)

Shared redaction engine catches credentials, PII, and legal/damaging content
before it reaches an external sink (codex dispatch, GitHub issue/PR body, pushed
commit). It is a **guardrail, not airtight enforcement** — `git push --no-verify`,
direct `gh issue create`, and `GSTACK_REDACT_PREPUSH=skip` all bypass it. It
catches accidents and carelessness, the 99% case. Do not claim it stops a
determined leaker (a CHANGELOG line that does would fail a hostile screenshotter).

- **Engine + taxonomy:** `lib/redact-patterns.ts` (the single source of truth —
  3 tiers; HIGH = genuinely-secret credentials that block, MEDIUM = PII/legal/
  internal + high-FP credential shapes that confirm via AskUserQuestion, LOW =
  FYI) and `lib/redact-engine.ts` (pure `scan()` + `applyRedactions()`).
  Calibration matters: a gate that cries wolf gets ignored, so context-variable
  shapes (Stripe `pk_live_`, Google `AIza`, JWT, env `*_KEY=`) sit at MEDIUM.
- **CLI:** `bin/gstack-redact` (exit 0 clean / 2 MEDIUM / 3 HIGH; `--json`,
  `--auto-redact`, `--repo-visibility`, `--from-file`). `bin/gstack-redact-prepush`
  is the opt-in git hook.
- **Skill docs are generated** from `scripts/resolvers/redact-doc.ts`
  (`{{REDACT_TAXONOMY_TABLE}}`, `{{REDACT_INVOCATION_BLOCK:<sink>}}`) so /spec,
  /cso, /ship, /document-release, /document-generate never drift from the engine.
- **Scan-at-sink:** always scan the EXACT bytes that will be sent — write to a
  temp file, scan that file, pass the SAME file to `gh`/`git`. Never scan a string
  then re-render (that reopens a scan-vs-send gap).
- **Visibility (no tier promotion):** resolve once per run, order = local config
  (`gstack-config get redact_repo_visibility`, ~/.gstack so never committed) → gh
  → glab → unknown(=public-strict). Public repos get STERNER per-finding
  confirmation (no batch-acknowledge, no silent-proceed); MEDIUM is never
  auto-promoted to HIGH.
- **Tool-attributed fences:** wrap Codex/Greptile/eval output in ` ```codex-review `
  / ` ```greptile ` fences so example credentials those tools quote WARN-degrade
  instead of blocking. A live-format credential inside the fence still blocks.
- **Config keys:** `redact_repo_visibility` (public|private|unknown, local-only
  override for repos gh/glab can't read), `redact_prepush_hook` (true|false).
  There is intentionally NO key to disable HIGH blocking.
- **Audit:** the /spec semantic pass appends a content-free record (categories +
  body sha256, no spec text) to `~/.gstack/security/semantic-reviews.jsonl` (0600).

## Search before building

Before designing any solution that involves concurrency, unfamiliar patterns,
infrastructure, or anything where the runtime/framework might have a built-in:

1. Search for "{runtime} {thing} built-in"
2. Search for "{thing} best practice {current year}"
3. Check official runtime/framework docs

Three layers of knowledge: tried-and-true (Layer 1), new-and-popular (Layer 2),
first-principles (Layer 3). Prize Layer 3 above all. See ETHOS.md for the full
builder philosophy.

## E2E eval failure blame protocol

When an E2E eval fails during `/ship` or any other workflow, **never claim "not
related to our changes" without proving it.** These systems have invisible couplings —
a preamble text change affects agent behavior, a new helper changes timing, a
regenerated SKILL.md shifts prompt context.

**Required before attributing a failure to "pre-existing":**
1. Run the same eval on main (or base branch) and show it fails there too
2. If it passes on main but fails on the branch — it IS your change. Trace the blame.
3. If you can't run on main, say "unverified — may or may not be related" and flag it
   as a risk in the PR body

"Pre-existing" without receipts is a lazy claim. Prove it or don't say it.

## Long-running tasks: don't give up

When running evals, E2E tests, or any long-running background task, **poll until
completion**. Use `sleep 180 && echo "ready"` + `TaskOutput` in a loop every 3
minutes. Never switch to blocking mode and give up when the poll times out. Never
say "I'll be notified when it completes" and stop checking — keep the loop going
until the task finishes or the user tells you to stop.

The full E2E suite can take 30-45 minutes. That's 10-15 polling cycles. Do all of
them. Report progress at each check (which tests passed, which are running, any
failures so far). The user wants to see the run complete, not a promise that
you'll check later.

## Running evals as an agent: always detach (SIGTERM-proof)

When **you (an agent/harness)** launch a long eval/benchmark run, run it through
`bin/gstack-detach` — NEVER as a plain backgrounded Bash task. A plain background
task lives in the harness's process group, so a SIGTERM ("polite quit") on a turn
boundary, a stopped Monitor, or an interruption kills the run mid-flight (observed:
`script "test:gate" was terminated by signal SIGTERM` ~40 min into a run). On macOS
the run can also die to idle-sleep. `gstack-detach` fixes both: a fresh session
(escapes the group SIGTERM) wrapped in `caffeinate -i` (blocks idle-sleep).

- Use the `eval:bg*` scripts (`eval:bg`, `eval:bg:all`, `eval:bg:gate`,
  `eval:bg:periodic`) — they wrap the eval command in `gstack-detach` with the
  machine-wide `gstack-evals` lock (concurrent worktrees serialize instead of
  saturating the shared model API), a per-tier watchdog, and a **run-scoped** log
  under `~/.gstack-dev/eval-runs/` (no shared-`/tmp` collision). Each prints its
  log path. Or call `gstack-detach [--lock NAME] [--timeout SECS] [--label LBL] --
  <cmd>` directly for any long agent job. Export `ANTHROPIC_API_KEY` first (never
  pass keys in argv).
- Then **poll the printed logfile** with a death-aware watcher: break on the
  guaranteed `### gstack-detach EXIT=<code> ###` sentinel (success AND failure are
  both marked, so silence is never mistaken for success). The detached run survives
  even if your watcher gets reaped, so re-checking the log always works.
- Why the lock: a shared dev box with several Conductor worktrees will rate-limit
  the model API if two eval suites run at once (15-way concurrency each), which
  mass-times-out E2E tests. The lock makes the second run WAIT, not collide.
- Humans running `bun run test:evals` foreground in their own terminal don't need
  this — Ctrl-C is intended there. Detachment is for agent-launched runs only.

## Everything else — `.claude/rules/`

Load the one you need; do not read them all.

- **Commands** → `.claude/rules/commands.md` (70 lines)
- **Testing** → `.claude/rules/testing.md` (11 lines)
- **Project structure** → `.claude/rules/project-structure.md` (75 lines)
- **SKILL.md workflow** → `.claude/rules/skill-md-workflow.md` (28 lines)
- **Writing SKILL templates** → `.claude/rules/writing-skill-templates.md` (17 lines)
- **Writing style (V1)** → `.claude/rules/writing-style-v1.md` (12 lines)
- **Browser interaction** → `.claude/rules/browser-interaction.md` (160 lines)
- **Commit style** → `.claude/rules/commit-style.md` (16 lines)
- **Slop-scan: AI code quality, not AI code hiding** → `.claude/rules/slop-scan-ai-code-quality-not-ai-code-hiding.md` (56 lines)
- **Community PR guardrails** → `.claude/rules/community-pr-guardrails.md` (18 lines)
- **Checking out PRs from garrytan-agents** → `.claude/rules/checking-out-prs-from-garrytan-agents.md` (30 lines)
- **CHANGELOG + VERSION style** → `.claude/rules/changelog-version-style.md` (186 lines)
- **AI effort compression** → `.claude/rules/ai-effort-compression.md` (19 lines)
- **Local plans** → `.claude/rules/local-plans.md` (6 lines)
- **E2E test fixtures: extract, don't copy** → `.claude/rules/e2e-test-fixtures-extract-don-t-copy.md` (24 lines)
- **Publishing native OpenClaw skills to ClawHub** → `.claude/rules/publishing-native-openclaw-skills-to-clawhub.md` (23 lines)
- **Deploying to the active skill** → `.claude/rules/deploying-to-the-active-skill.md` (18 lines)
- **Skill routing** → `.claude/rules/skill-routing.md` (18 lines)
- **Cross-session decision memory** → `.claude/rules/cross-session-decision-memory.md` (25 lines)
- **GBrain Search Guidance (configured by /sync-gbrain)** → `.claude/rules/gbrain-search-guidance-configured-by-sync-gbrain.md` (43 lines)
