# GitHub CLI for This Course

This course uses the GitHub CLI (`gh`) as the default way to operate GitHub Actions from a terminal. The web interface remains useful for visual workflow graphs and settings that do not have a convenient first-class CLI command.

Commands in this guide assume your terminal is inside the practice repository. Add `--repo OWNER/REPOSITORY` when operating on a different repository.

Uppercase values such as `RUN_ID`, `ENVIRONMENT_ID`, and `WORKFLOW_FILE.yaml` are placeholders. Replace them with values from your repository before running a command. GitHub and Docker Hub examples use your username, `prankbox`.

## Install and Authenticate

Install `gh` using the instructions for your operating system, then authenticate:

```bash
gh auth login
gh auth status
gh repo view
```

Choose GitHub.com, HTTPS or SSH for Git operations, and browser-based authentication when prompted. Never paste an authentication token directly into a command because commands may be retained in shell history.

## Switch Between Multiple GitHub Accounts

GitHub CLI can store credentials for multiple accounts on the same host. Display every authenticated account and identify the active one:

```bash
gh auth status
gh auth status --active --hostname github.com
```

Switch the account used by subsequent `gh` commands:

```bash
# Switch to the prankbox account
gh auth switch --hostname github.com --user prankbox

# Switch back to radiodevops
gh auth switch --hostname github.com --user radiodevops
```

Verify the API identity after switching:

```bash
gh api user --jq '.login'
```

> **Important:** `gh auth switch` changes the account used by GitHub CLI API operations. When Git operations use SSH, it does not necessarily change the SSH key selected by `git clone`, `git fetch`, or `git push`.

### Use a Separate SSH Identity for Each Account

Create a distinct SSH key for each account and add each public key to the corresponding GitHub account. Example key filenames:

```text
~/.ssh/id_ed25519_radiodevops
~/.ssh/id_ed25519_prankbox
```

Configure aliases in **`~/.ssh/config`**:

```sshconfig
Host github-radiodevops
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_radiodevops
    IdentitiesOnly yes

Host github-prankbox
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_prankbox
    IdentitiesOnly yes
```

`IdentitiesOnly yes` prevents SSH from offering unrelated keys loaded in the SSH agent.

Test both identities:

```bash
ssh -T git@github-radiodevops
ssh -T git@github-prankbox
```

GitHub's successful authentication message still commonly accompanies a non-zero SSH exit status because GitHub does not provide interactive shell access. Confirm that the username in the message is the expected account.

Configure the correct alias in each repository's remote URL:

```bash
# Repository owned by radiodevops
git remote set-url origin \
  git@github-radiodevops:radiodevops/REPOSITORY.git

# Repository owned by prankbox
git remote set-url origin \
  git@github-prankbox:prankbox/REPOSITORY.git

git remote -v
```

Clone new repositories through the appropriate alias:

```bash
git clone git@github-radiodevops:radiodevops/REPOSITORY.git
git clone git@github-prankbox:prankbox/REPOSITORY.git
```

### Configure Commit Authorship Per Repository

Authentication and commit authorship are independent. Configure the correct author inside each repository:

```bash
git config --local user.name "prankbox"
git config --local user.email "YOUR_PRANKBOX_EMAIL"

git config --local --get user.name
git config --local --get user.email
```

Use the corresponding values in repositories owned by `radiodevops`. Local configuration applies only to the current repository and overrides global author settings.

The resulting separation is:

```text
gh auth switch           → GitHub CLI and API account
SSH host alias           → git clone, fetch, and push identity
git config --local user  → Commit author name and email
```

## Create the Practice Repository

```bash
# Create an empty private repository under the authenticated account
gh repo create gha-practice --private

# Confirm its metadata and obtain its SSH URL
gh repo view prankbox/gha-practice
gh repo view prankbox/gha-practice --json sshUrl --jq '.sshUrl'
```

If you already have a committed local repository, create the remote, add it as `origin`, and push in one operation:

```bash
gh repo create gha-practice \
  --private \
  --source=. \
  --remote=origin \
  --push
```

## Discover Workflows

```bash
# List active workflows
gh workflow list

# Display a workflow summary or its YAML
gh workflow view WORKFLOW_FILE.yaml
gh workflow view WORKFLOW_FILE.yaml --yaml
```

The workflow must be present on the default branch before `gh workflow run` can dispatch it.

## Trigger a Manual Workflow

```bash
# Run on the default branch
gh workflow run WORKFLOW_FILE.yaml

# Run the workflow version from a specific branch or tag
gh workflow run WORKFLOW_FILE.yaml --ref BRANCH_OR_TAG

# Supply workflow_dispatch inputs
gh workflow run WORKFLOW_FILE.yaml \
  --ref main \
  -f environment=dev \
  -f perform_smoke_test=true
```

Running `gh workflow run` without a workflow name opens an interactive workflow and input selector.

## List, Watch, and Inspect Runs

```bash
# Find the run ID returned by a push, pull request, schedule, or dispatch
gh run list --limit 10
gh run list --workflow WORKFLOW_FILE.yaml --limit 5
gh run list --branch main --limit 5

# Return structured data that is convenient for scripts
gh run list \
  --workflow WORKFLOW_FILE.yaml \
  --limit 5 \
  --json databaseId,status,conclusion,url

# Follow a selected run and return a failure exit code when it fails
gh run watch RUN_ID --exit-status

# Inspect the run, its steps, or its logs
gh run view RUN_ID
gh run view RUN_ID --verbose
gh run view RUN_ID --log
gh run view RUN_ID --log-failed
```

Useful recovery commands:

```bash
gh run rerun RUN_ID --failed
gh run cancel RUN_ID
```

Use `gh pr checks` to inspect workflow checks associated with the current pull request.

## Download Artifacts

```bash
# Download all artifacts from a specific run
gh run download RUN_ID --dir downloaded-artifacts

# Download one named artifact
gh run download RUN_ID \
  --name flask-ci-artifacts \
  --dir downloaded-artifacts
```

The command extracts artifact contents into the destination directory. Use an explicit run ID when reproducibility matters.

## Manage Actions Secrets Safely

Set a repository secret interactively so the value does not appear in shell history:

```bash
gh secret set DOCKERHUB_TOKEN
gh secret list
```

At the prompt, paste the value and press Enter. For automation, read the value from a protected file or standard input instead of using `--body` with a literal secret:

```bash
gh secret set DOCKERHUB_TOKEN < token.txt
```

Delete the temporary plaintext file securely after verifying the secret. GitHub never returns stored secret values:

```bash
gh secret delete DOCKERHUB_TOKEN
```

## Manage Actions Variables

Repository variables are not secrets:

```bash
gh variable set DOCKERHUB_USERNAME --body prankbox
gh variable set DOCKER_IMAGE_NAME --body prankbox/flask-app
gh variable list
```

Environment-scoped values use `--env`:

```bash
gh variable set APPLICATION_URL --env dev --body https://dev.example.com
gh secret set DEPLOYMENT_TOKEN --env dev

gh variable list --env dev
gh secret list --env dev
```

## Manage GitHub Actions Environments

GitHub CLI does not currently provide a dedicated `gh environment` command. Use the authenticated REST API through `gh api`:

```bash
# Create an environment; the command is idempotent
gh api \
  --method PUT \
  repos/{owner}/{repo}/environments/dev

# List environments
gh api \
  repos/{owner}/{repo}/environments \
  --jq '.environments[] | [.id, .name] | @tsv'

# Inspect one environment
gh api repos/{owner}/{repo}/environments/dev
```

The `{owner}` and `{repo}` placeholders are filled from the repository in the current directory. Protection rules such as reviewers and wait timers can also be configured through this endpoint, but the request body is more complex; the environment lesson shows the UI when it is clearer for learning.

## Approve a Pending Environment Deployment

First inspect the pending environments and their numeric IDs:

```bash
gh api repos/{owner}/{repo}/actions/runs/RUN_ID/pending_deployments
```

An authorized reviewer can then approve one environment:

```bash
gh api \
  --method POST \
  repos/{owner}/{repo}/actions/runs/RUN_ID/pending_deployments \
  -F 'environment_ids[]=ENVIRONMENT_ID' \
  -f state=approved \
  -f comment='Approved from GitHub CLI'
```

Approval is an external state change. Verify the run, environment, commit, and deployment target before executing it.

## Use `gh api` for Missing CLI Commands

`gh api` sends authenticated GitHub REST or GraphQL requests:

```bash
gh api repos/{owner}/{repo}
gh api repos/{owner}/{repo}/actions/runs --jq '.workflow_runs[0].html_url'
```

Prefer first-class commands such as `gh workflow`, `gh run`, `gh secret`, and `gh variable` when available. They provide clearer flags and safer defaults.

## Command Help

The installed CLI is the authoritative reference for its version:

```bash
gh help workflow
gh help workflow run
gh help run
gh help secret set
gh help variable set
gh help api
```

Official references:

* [GitHub CLI manual](https://cli.github.com/manual/)
* [`gh workflow run`](https://cli.github.com/manual/gh_workflow_run)
* [`gh run`](https://cli.github.com/manual/gh_run)
* [`gh secret`](https://cli.github.com/manual/gh_secret)
* [`gh variable`](https://cli.github.com/manual/gh_variable)
* [`gh api`](https://cli.github.com/manual/gh_api)
