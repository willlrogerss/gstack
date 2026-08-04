## Testing

```bash
bun test             # run before every commit — free, <2s
bun run test:evals   # run before shipping — paid, diff-based (~$4/run max)
```

`bun test` runs skill validation, gen-skill-docs quality checks, and browse
integration tests. `bun run test:evals` runs LLM-judge quality evals and E2E
tests via `claude -p`. Both must pass before creating a PR.
