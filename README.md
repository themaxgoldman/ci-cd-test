# ci-cd-test

A copy of chariot's staging and release flow, for testing changes to it end to end before they reach chariot.

## How close it is to chariot

A passing test here only proves what the sandbox actually models. Check this table before trusting a result.

| Part of chariot | In this repo | What a test here can't catch |
|---|---|---|
| Staging-CD started by the controller with `workflow_dispatch` on `staging` | Same | Chariot's real jobs: Go/NPM CI, Docker builds, Vercel, Knock. They are stand-ins here. |
| AWS login (OIDC role trusted only for `refs/heads/staging`) | Stand-in step that fails unless the run's identity ends in `ref:refs/heads/staging` | The real role's policy and permissions. This repo's ID uses GitHub's ID-based format (`repo:owner@id/name@id:…`); chariot's uses `repo:chariot-giving/chariot:…`. |
| ArgoCD rollout, then acceptance requests | Stand-in that sets the "live" commit, marks the Staging deployment, then sends `ACCEPTANCE_DISPATCHES` requests (default 3), `DISPATCH_GAP_SECONDS` apart | Real requests come about 2 minutes apart, and a real acceptance run takes about 2 minutes, so later requests cancel runs already in progress. To model that, use `ACCEPTANCE_SECONDS=90 DISPATCH_GAP_SECONDS=40`. With the short defaults, requests cancel runs that are still queued, which hides cancellation bugs. Real syncs can also degrade; here they only pass or hang. |
| Acceptance tests (`cancel-in-progress`, pending then final status) | Same concurrency and status steps. `ACCEPTANCE_SETUP_SECONDS` models the ~40s before chariot posts pending. Also posts an `alignment` status that fails if staging changed mid-run. | The real test suite and how long it takes. |
| Controller and its scripts | Same scripts as chariot's `mise/tasks/github/*` | — |
| Release PR description (`prepare-release`) | A simplified script using the compare API, plus a step that runs chariot's git history fetch and lists unreleased PRs the same way | The real script: Linear linking, gum, the template. Replay it against real chariot data instead. |
| Old `staging → main` release PR (`release_preparer.yml`) | Not modelled | Anything about that PR, such as its diff going stale after a release |
| **Tokens** | `PIPELINE_TOKEN` is a personal access token that can do everything. It stands in for the Nike app. | **Missing app permissions.** The Nike app can't read commit statuses, for example. Check each Nike call against its permissions instead. |
| **Repo visibility** | Public, because rulesets need a paid plan on private personal repos | **Missing `GITHUB_TOKEN` permissions.** Public data is readable without them, for example issues. Check every call against the job's `permissions:` instead. |
| `main` rules | PR, required checks (`staging-verified`, `check-label`, `validate`), required Staging deployment | Chariot also requires `Staging – acceptance`, `check`, `check-description` and a releasers-team approval |
| `staging` rules | None | Chariot requires PRs, linear history, signed commits and checks, including `check-release-freeze` |
| `prod-release` rules | Restrict deletions | Chariot has none yet, so there releases still delete `prod-release` |
| Linear | Not modelled | Release PR linking, which depends on each issue's state |
| Timing | Staging-CD takes about 1 minute; chariot's takes 5–10 | Races that only show with long runs. Use the `TEST_PAUSE_BEFORE_FETCH` variable or longer `sim.env` timings to widen them. |

## Before trusting a test here

1. **Tokens:** for every API call in a changed step, check that the token used there can make it in chariot. That means the job's `permissions:` for `GITHUB_TOKEN`, or the Nike app's permissions.
2. **Other systems:** list each system the change touches (AWS, ArgoCD, Vercel, Knock, Linear) and check the table above for whether it's modelled.
3. **What starts each run:** list the events the change relies on and which token creates each one. Events made with `GITHUB_TOKEN` don't start other workflows, except `workflow_dispatch` and `repository_dispatch`.
4. **PR workflows:** a pull request runs the workflow files from its own code. A change to a PR-triggered workflow only reaches the release PR once `prod-release` contains it.
5. **Real data:** replay changed scripts against real chariot history or commits where you can, and compare old and new output.
6. **The first real run:** watch it in chariot step by step.

## Per-commit knobs (`sim.env`)

`TESTS`, `TESTS_SECONDS`, `BUILD`, `ROLLOUT` (`ok` or `hang`), `ROLLOUT_SECONDS`, `ACCEPTANCE`, `ACCEPTANCE_SECONDS`, `ACCEPTANCE_DISPATCHES`, `DISPATCH_GAP_SECONDS`, `ACCEPTANCE_SETUP_SECONDS`.
