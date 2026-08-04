## Cross-session decision memory

Durable decisions and their rationale are captured in an append-only, event-sourced
store at `~/.gstack/projects/<slug>/decisions.jsonl` so neither you nor the user
re-litigates a settled call or loses the "why" across sessions. This is the reliable,
file-only path: it works with gbrain OFF. (gbrain semantic recall is an optional
enhancement layered on top, never a dependency.)

- **Resurface** active decisions before re-deciding: `bin/gstack-decision-search`
  (`--recent N`, `--scope repo|branch|issue`, `--query KW`, `--all`, `--json`).
  Add `--semantic` (with `--query`) to append related hits from gbrain memory when
  it's up; it degrades silently to the reliable file results when gbrain is off.
  Session start already surfaces scope-relevant active decisions via Context Recovery.
  If a decision is listed, treat it as settled with its rationale; if you're about to
  reverse it, say so explicitly.
- **Capture** a DURABLE decision when you or the user make one:
  `bin/gstack-decision-log '{"decision":"...","rationale":"...","scope":"repo|branch|issue","source":"user|skill|agent","confidence":1-10}'`.
  Reverse a prior call with `--supersede <id>`; expunge an accidental secret with
  `--redact <id>`; rewrite the log to the active set with `--compact`. Non-interactive
  (never prompts), injection-sanitized, and HIGH-secret-blocking on write.
- **Durable means:** architecture choice, scope cut, tool/vendor choice, or a reversal
  of a prior call. NOT a turn-level edit, a phrasing tweak, or anything trivially
  re-derivable. Capture is curated at the source — log durable decisions only, or the
  store becomes noise.
