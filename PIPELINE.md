# CI Pipeline Design (Day 2 Assignment)

## 1) Events that will trigger CI

The workflow will be triggered by:

- **push** to `main` - Runs on direct pushes to the main branch to validate the committed state. We **exclude** changes where only Markdown files (`*.md`) or files in `docs/` changed (using `paths-ignore`) to avoid running CI for documentation-only updates.
- **pull_request** targeting `main` - Runs for PRs into main to validate proposed changes before merging.
- **workflow_dispatch** (manual) - Allows manual runs with inputs (`python-version` choice, `create-package` boolean) for ad hoc testing.
- **schedule** - Runs every Monday at 07:00 UTC (cron `0 7 * * 1`) to catch issues from external dependencies over time.

## 2) Jobs and job graph

We will create the following jobs:

| Job | Purpose |
|-----|---------|
| `test` | Install dependencies and run pytest with JUnit output (runs first, no dependencies). |
| `package` | Build `.tar.gz` artifact containing `temperature.py` and `README.md` (depends on `test`; runs only if tests pass and conditions allow). |
| `verify` | Download and verify the built package (no checkout) to ensure it is usable (depends on `package`). |
| `pr-comment` | Post a comment on PRs with package version (runs only on `pull_request`, after `package` succeeds). |
| `report` | Aggregate results from upstream jobs and write a job summary; runs after `test` and `package` and always runs (`if: always()`). |

**Job dependencies / execution order:**

- `test` starts first (no `needs`).
- `package` needs `test` (`needs: [test]`). It runs only if `test` succeeded (`if: success() && ...`) and meets package creation conditions (not manually disabled).
- `verify` needs `package` (`needs: [package]`). Runs if `package` succeeded.
- `pr-comment` needs `package` and runs only for pull requests (`if: github.event_name == 'pull_request' && success()`). 
- `report` needs `test` and `package` (`needs: [test, package]`) and runs always (`if: always()`), regardless of success/failure/skip of upstream jobs. This satisfies "report runs in every case".

## 3) Artifact produced

The pipeline produces a distributable package artifact: `dist/temperature-<version>.tar.gz` (uploaded as artifact name `package`). The archive contains `temperature.py` and `README.md`. The version is computed as `0.1.<run_number>` in the `package` job.

Additionally, the `test` job uploads a JUnit XML report as artifact `test-report` (retained 5 days) even when tests fail, to aid debugging.

## 4) CI, Continuous Delivery, or Continuous Deployment?

This is **Continuous Integration (CI)**. We build, test, package, and verify the artifact on every relevant change; we do not automatically deploy/release to any environment (no push to registries/staging/prod on merge). 

## 5) What should happen to `package` job when tests fail, and why?

The `package` job must **not run** when tests fail. Since `package` depends on `test` (`needs: [test]`) and will be gated to run only if tests succeeded, a failing `test` means `package` is skipped (or not started). This is correct because we should not package untested/invalid code. It also saves time/resources and prevents producing artifacts from broken builds.
