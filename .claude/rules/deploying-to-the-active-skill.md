## Deploying to the active skill

The active skill lives at `~/.claude/skills/gstack/`. After making changes:

1. Push your branch
2. Fetch and reset in the skill directory: `cd ~/.claude/skills/gstack && git fetch origin && git reset --hard origin/main`
3. Rebuild: `cd ~/.claude/skills/gstack && bun run build`

**If you use gbrain:** the `git reset --hard` in step 2 reverts the brain-aware
(`GBRAIN_CONTEXT_LOAD` / `GBRAIN_SAVE_RESULTS`) blocks that `gstack-config
gbrain-refresh` renders into the install (those generated blocks differ from
`main` by design). After deploying, re-run `gstack-config gbrain-refresh` to
restore them across all your projects' Claude sessions. It's idempotent.

Or copy the binaries directly:
- `cp browse/dist/browse ~/.claude/skills/gstack/browse/dist/browse`
- `cp design/dist/design ~/.claude/skills/gstack/design/dist/design`
