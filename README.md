# changeflow test target: `scoped_validation`

Validates a fourth idea that none of the other three test targets cover: **a specific PR-time
validation flow for the API's own PRs vs. a human's**, not a different merge mechanism. The
merge mechanism here is plain `CHANGEFLOW_MERGE_MODE=native_auto_merge` (already proven end-to-end
in `../native-auto-merge/`) — what's actually under test is `ci.yml`.

## The idea

A human merging a PR exercises judgement over an arbitrary diff; the API never does — it only
ever makes the one change it's coded to make. That asymmetry means the two paths don't need the
same gate:

| Path | Extra rule | Why |
|---|---|---|
| Human | Naming-convention lint (team names must be lowercase kebab-case) | A human can type anything; the app already validates this before it ever opens a PR, so re-checking it for the bot is redundant |
| The App | Blast-radius guard (diff must touch only `teams.json`, and add exactly one team) | A human reviewer can eyeball an arbitrary diff; an automated credential should be mechanically constrained to the one thing it's supposed to do, since nothing else is looking |

Both rules live in `ci.yml`'s single `validate` job, branching by `github.actor` at the **step**
level, not the job level — see the comment at the top of `ci.yml` for why job-level `if:` would
permanently hang the required check for whichever path got skipped.

## What's here

| File | Purpose |
|---|---|
| `teams.json` | The file changeflow appends `{"team": "<name>"}` to |
| `.github/workflows/ci.yml` | PR-time required check (`validate`) — same context for both paths, different steps inside |
| `.github/workflows/deploy.yml` | The pipeline triggered by the merge, gated by the `production` environment — shared, unscoped by actor (splitting *this* is `ruleset-bypass`'s experiment, not this one) |
| `setup.sh` | One-time `gh` CLI setup — identical to `native-auto-merge`'s (ruleset, "Allow auto-merge", production environment) |

## One-time setup

```bash
export GH_OWNER=<your-github-user-or-org>
export REPO_NAME=changeflow-test-scoped-validation   # or whatever you name it
export REVIEWER_LOGIN=<github-login-for-the-required-reviewer>   # defaults to punitlad
./setup.sh
```

Then **install your GitHub App on this repo** (Settings → GitHub Apps → your App → configure →
add this repo) — `setup.sh` can't do that part.

## Run changeflow against it

```bash
export CHANGEFLOW_TARGET_OWNER=$GH_OWNER
export CHANGEFLOW_TARGET_REPO=$REPO_NAME
export CHANGEFLOW_MERGE_MODE=native_auto_merge
export CHANGEFLOW_MERGE_METHOD=squash
export CHANGEFLOW_APPROVAL_MODE=pending_deployments
export CHANGEFLOW_PIPELINE_ENVIRONMENT=production
export CHANGEFLOW_APPROVER_TOKEN=<a PAT for whoever REVIEWER_LOGIN was set to (punitlad by default), with repo + workflow scope>
# ...plus CHANGEFLOW_APP_ID / CHANGEFLOW_APP_PRIVATE_KEY / CHANGEFLOW_INSTALLATION_ID
uvicorn changeflow.api:app
curl -XPOST localhost:8000/team-onboardings -d '{"team":"payments","requested_by":"you"}'
```

## What "validated" looks like

For the API path: `GET /team-onboardings/{id}` reaches `phase: succeeded`. On GitHub, the
`validate` run for that PR shows the "Blast-radius guard" step executed and the "Naming
convention lint" step skipped.

For the human path: open a PR by hand that adds a team with an invalid name (e.g. `"Team One"`)
and confirm `validate` fails on the naming lint. Then open a PR by hand that only touches
`teams.json` with a valid name and confirm it merges normally — same `native_auto_merge` /
`pending_deployments` flow a human's PR would already use on `../native-auto-merge/`, since this
target doesn't change anything about how merging or approval works, only what's checked
beforehand.

## Trade-offs for team discussion

- **The asymmetry is the point, and also the risk.** Diverging rules by actor means the test
  suite for `ci.yml` has two paths to keep in sync with reality, not one — if the app's PR shape
  ever changes (e.g. it starts touching a second file), the blast-radius guard has to change with
  it or it starts failing the App's own legitimate PRs.
- **Only safe because it's step-level, not job-level.** Anyone editing this workflow needs to keep
  that invariant — a job-level `if:` here reintroduces the "required check never posts, PR hangs
  forever" trap called out in `ruleset-bypass` and `workflow-gated`'s own `ci.yml` comments.
- **Doesn't reduce trust the way `ruleset_bypass` does.** The API still has to pass a required
  check like anyone else; this mode is about *what* that check verifies, not about skipping it.
  Worth pairing with `ruleset_bypass` or `workflow_gated` in a real adoption if the target team
  wants both a lighter check *and* a lighter merge gate for the automated path.
