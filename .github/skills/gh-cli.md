# Skill: GitHub CLI (gh)

## Authentication

```bash
gh auth login          # interactive login (browser or token)
gh auth status         # verify current auth state
gh auth token          # print the active token
gh auth logout         # logout
```

## Repositories

```bash
gh repo clone terjetyl/<repo>          # clone a repo
gh repo create <repo> --private        # create a new private repo
gh repo view                           # open current repo in browser
gh repo view --json name,description   # JSON output
```

## Pull Requests

```bash
# Create a PR (uses current branch)
gh pr create --title "feat: ..." --body "..." --base main

# Create a draft PR
gh pr create --draft --title "WIP: ..."

# List open PRs
gh pr list

# View a specific PR
gh pr view 42

# Check out a PR locally
gh pr checkout 42

# Merge a PR (squash + delete branch)
gh pr merge 42 --squash --delete-branch

# Review a PR
gh pr review 42 --approve
gh pr review 42 --request-changes --body "Needs tests"

# Merge automatically when checks pass
gh pr merge 42 --auto --squash

# Check PR CI status
gh pr checks 42
```

## Issues

```bash
gh issue create --title "Bug: ..." --body "..." --label bug
gh issue list
gh issue view 7
gh issue close 7
gh issue comment 7 --body "Fixed in #42"
```

## Actions / Workflows

```bash
# List recent workflow runs
gh run list
gh run list --workflow=deploy.yml

# Watch a run in real time
gh run watch <run-id>

# View logs of a run
gh run view <run-id> --log

# Re-run failed jobs only
gh run rerun <run-id> --failed-only

# Trigger a workflow manually (workflow_dispatch)
gh workflow run deploy.yml
gh workflow run deploy.yml --ref main
gh workflow run deploy.yml -f environment=production
```

## Secrets and Variables

```bash
# Secrets (encrypted, available in workflow env)
gh secret set MY_SECRET                        # prompts for value
gh secret set MY_SECRET --body "value"
gh secret set MY_SECRET < ./secret.txt         # read from file
gh secret list
gh secret delete MY_SECRET

# Environment-scoped secrets
gh secret set MY_SECRET --env production

# Variables (plaintext, available in workflow env)
gh variable set MY_VAR --body "value"
gh variable list
gh variable delete MY_VAR
```

## Releases

```bash
# Create a release from a tag
gh release create v1.2.0 --title "v1.2.0" --notes "..."

# Generate release notes automatically from merged PRs
gh release create v1.2.0 --generate-notes

# Upload assets to an existing release
gh release upload v1.2.0 ./dist/app.tar.gz

# List / view / delete releases
gh release list
gh release view v1.2.0
gh release delete v1.2.0
```

## Common patterns in this repo

### Set all secrets from a .env file

```bash
# Read .env.prod and set each key as a GitHub secret
grep -v '^#' .env.prod | grep '=' | while IFS='=' read -r key value; do
  gh secret set "$key" --body "$value"
done
```

### Trigger a deploy and watch it

```bash
gh workflow run deploy.yml --ref main
sleep 3
run_id=$(gh run list --workflow=deploy.yml --limit 1 --json databaseId --jq '.[0].databaseId')
gh run watch "$run_id"
```

### Check CI before merging

```bash
gh pr checks                   # checks on current branch PR
gh pr merge --auto --squash    # merge automatically when all checks pass
```

### GitHub REST API via gh

```bash
# GET
gh api repos/:owner/:repo

# PATCH with fields
gh api -X PATCH repos/:owner/:repo \
  -f description="New description" \
  -F has_wiki=false

# Extract a field with jq
gh api repos/:owner/:repo --jq '.default_branch'
```

## Tips

- `:owner` and `:repo` in `gh api` paths are auto-filled from the current repo.
- Set `GH_TOKEN` env var to override auth in CI scripts.
- Add `--json field1,field2 --jq '.field1'` to most commands for structured output.
- `gh extension install <owner>/<repo>` to add community extensions.
