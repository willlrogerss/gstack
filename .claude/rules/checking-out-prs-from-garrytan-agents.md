## Checking out PRs from garrytan-agents

When the user says "check out <PR link>" and the PR is from `garrytan-agents/gstack`
(or any other fork that is NOT a collaborator on `garrytan/gstack`), do NOT just
`gh pr checkout`. Fork PRs don't receive base-repo secrets (`ANTHROPIC_API_KEY`,
`OPENAI_API_KEY`, etc.), so the eval/E2E CI jobs fail with empty-env auth errors
regardless of what's set on the base repo.

**Workflow:** push the branch to `garrytan/gstack` (the base repo) and re-target
the PR from there.

Concretely, after `gh pr checkout <N>`:

1. Note the original PR number and head branch name.
2. Push the same branch to the base repo: `git push origin HEAD:<branch-name>`
   (origin = `garrytan/gstack`, since the worktree is set up with that remote).
3. Close the fork PR (`gh pr close <N> --comment "moving to base-repo branch for secret access"`).
4. Open a new PR from the base-repo branch: `gh pr create --base main --head <branch-name>`.
5. New PR's workflows will get secrets automatically.

Why not fix it on the fork side? `garrytan-agents` isn't a collaborator on
`garrytan/gstack`. Adding it as a collaborator (option A) or flipping the
repo-wide "send secrets to fork PRs" toggle (option B) would let secrets reach
fork PRs from anyone — broader blast radius than just moving this one branch.
Option C (this section) keeps secret-distribution scope tight.

If the user asks you to skip the move (e.g., "just leave it as a fork PR"),
respect that — eval CI will fail with empty-env auth, but check-freshness,
workflow-lint, and windows-tests will still pass on the fork PR.
