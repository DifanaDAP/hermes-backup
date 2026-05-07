---
name: github-repo-management
description: "Clone/create/fork repos; manage remotes, releases."
version: 1.1.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [GitHub, Repositories, Git, Releases, Secrets, Configuration]
    related_skills: [github-auth, github-pr-workflow, github-issues]
---

# GitHub Repository Management

Create, clone, fork, configure, and manage GitHub repositories. Each section shows `gh` first, then the `git` + `curl` fallback.

## Prerequisites

- Authenticated with GitHub (see `github-auth` skill)

### Setup

```bash
if command -v gh &>/dev/null && gh auth status &>/dev/null; then
  AUTH="gh"
else
  AUTH="git"
  if [ -z "$GITHUB_TOKEN" ]; then
    if [ -f ~/.hermes/.env ] && grep -q "^GITHUB_TOKEN=" ~/.hermes/.env; then
      GITHUB_TOKEN=$(grep "^GITHUB_TOKEN=" ~/.hermes/.env | head -1 | cut -d= -f2 | tr -d '\n\r')
    elif grep -q "github.com" ~/.git-credentials 2>/dev/null; then
      GITHUB_TOKEN=$(grep "github.com" ~/.git-credentials 2>/dev/null | head -1 | sed 's|https://[^:]*:\([^@]*\)@.*|\1|')
    fi
  fi
fi

# Get your GitHub username (needed for several operations)
if [ "$AUTH" = "gh" ]; then
  GH_USER=$(gh api user --jq '.login')
else
  GH_USER=$(curl -s -H "Authorization: token $GITHUB_TOKEN" https://api.github.com/user | python3 -c "import sys,json; print(json.load(sys.stdin)['login'])")
fi
```

If you're inside a repo already:

```bash
REMOTE_URL=$(git remote get-url origin)
OWNER_REPO=$(echo "$REMOTE_URL" | sed -E 's|.*github\.com[:/]||; s|\.git$||')
OWNER=$(echo "$OWNER_REPO" | cut -d/ -f1)
REPO=$(echo "$OWNER_REPO" | cut -d/ -f2)
```

---

## 1. Cloning Repositories

Cloning is pure `git` — works identically either way:

```bash
# Clone via HTTPS (works with credential helper or token-embedded URL)
git clone https://github.com/owner/repo-name.git

# Clone into a specific directory
git clone https://github.com/owner/repo-name.git ./my-local-dir

# Shallow clone (faster for large repos)
git clone --depth 1 https://github.com/owner/repo-name.git

# Clone a specific branch
git clone --branch develop https://github.com/owner/repo-name.git

# Clone via SSH (if SSH is configured)
git clone git@github.com:owner/repo-name.git
```

**With gh (shorthand):**

```bash
gh repo clone owner/repo-name
gh repo clone owner/repo-name -- --depth 1
```

## 2. Creating Repositories

**With gh:**

```bash
# Create a public repo and clone it
gh repo create my-new-project --public --clone

# Private, with description and license
gh repo create my-new-project --private --description "A useful tool" --license MIT --clone

# Under an organization
gh repo create my-org/my-new-project --public --clone

# From existing local directory
cd /path/to/existing/project
gh repo create my-project --source . --public --push
```

**With git + curl:**

```bash
# Create the remote repo via API
curl -s -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/user/repos \
  -d '{
    "name": "my-new-project",
    "description": "A useful tool",
    "private": false,
    "auto_init": true,
    "license_template": "mit"
  }'

# Clone it
git clone https://github.com/$GH_USER/my-new-project.git
cd my-new-project

# -- OR -- push an existing local directory to the new repo
cd /path/to/existing/project
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/$GH_USER/my-new-project.git
git push -u origin main
```

To create under an organization:

```bash
curl -s -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/orgs/my-org/repos \
  -d '{"name": "my-new-project", "private": false}'
```

### From a Template

**With gh:**

```bash
gh repo create my-new-app --template owner/template-repo --public --clone
```

**With curl:**

```bash
curl -s -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/owner/template-repo/generate \
  -d '{"owner": "'"$GH_USER"'", "name": "my-new-app", "private": false}'
```

## 3. Forking Repositories

**With gh:**

```bash
gh repo fork owner/repo-name --clone
```

**With git + curl:**

```bash
# Create the fork via API
curl -s -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/owner/repo-name/forks

# Wait a moment for GitHub to create it, then clone
sleep 3
git clone https://github.com/$GH_USER/repo-name.git
cd repo-name

# Add the original repo as "upstream" remote
git remote add upstream https://github.com/owner/repo-name.git
```

### Keeping a Fork in Sync

```bash
# Pure git — works everywhere
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

**With gh (shortcut):**

```bash
gh repo sync $GH_USER/repo-name
```

## 4. Repository Information

**With gh:**

```bash
gh repo view owner/repo-name
gh repo list --limit 20
gh search repos "machine learning" --language python --sort stars
```

**With curl:**

```bash
# View repo details
curl -s \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO \
  | python3 -c "
import sys, json
r = json.load(sys.stdin)
print(f\"Name: {r['full_name']}\")
print(f\"Description: {r['description']}\")
print(f\"Stars: {r['stargazers_count']}  Forks: {r['forks_count']}\")
print(f\"Default branch: {r['default_branch']}\")
print(f\"Language: {r['language']}\")"

# List your repos
curl -s \
  -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/user/repos?per_page=20&sort=updated" \
  | python3 -c "
import sys, json
for r in json.load(sys.stdin):
    vis = 'private' if r['private'] else 'public'
    print(f\"  {r['full_name']:40}  {vis:8}  {r.get('language', ''):10}  ★{r['stargazers_count']}\")"

# Search repos
curl -s \
  "https://api.github.com/search/repositories?q=machine+learning+language:python&sort=stars&per_page=10" \
  | python3 -c "
import sys, json
for r in json.load(sys.stdin)['items']:
    print(f\"  {r['full_name']:40}  ★{r['stargazers_count']:6}  {r['description'][:60] if r['description'] else ''}\")"
```

## 5. Repository Settings

**With gh:**

```bash
gh repo edit --description "Updated description" --visibility public
gh repo edit --enable-wiki=false --enable-issues=true
gh repo edit --default-branch main
gh repo edit --add-topic "machine-learning,python"
gh repo edit --enable-auto-merge
```

**With curl:**

```bash
curl -s -X PATCH \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO \
  -d '{
    "description": "Updated description",
    "has_wiki": false,
    "has_issues": true,
    "allow_auto_merge": true
  }'

# Update topics
curl -s -X PUT \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github.mercy-preview+json" \
  https://api.github.com/repos/$OWNER/$REPO/topics \
  -d '{"names": ["machine-learning", "python", "automation"]}'
```

## 6. Branch Protection

```bash
# View current protection
curl -s \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/branches/main/protection

# Set up branch protection
curl -s -X PUT \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/branches/main/protection \
  -d '{
    "required_status_checks": {
      "strict": true,
      "contexts": ["ci/test", "ci/lint"]
    },
    "enforce_admins": false,
    "required_pull_request_reviews": {
      "required_approving_review_count": 1
    },
    "restrictions": null
  }'
```

## 7. Secrets Management (GitHub Actions)

**With gh:**

```bash
gh secret set API_KEY --body "your-secret-value"
gh secret set SSH_KEY < ~/.ssh/id_rsa
gh secret list
gh secret delete API_KEY
```

**With curl:**

Secrets require encryption with the repo's public key — more involved via API:

```bash
# Get the repo's public key for encrypting secrets
curl -s \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/actions/secrets/public-key

# Encrypt and set (requires Python with PyNaCl)
python3 -c "
from base64 import b64encode
from nacl import encoding, public
import json, sys

# Get the public key
key_id = '<key_id_from_above>'
public_key = '<base64_key_from_above>'

# Encrypt
sealed = public.SealedBox(
    public.PublicKey(public_key.encode('utf-8'), encoding.Base64Encoder)
).encrypt('your-secret-value'.encode('utf-8'))
print(json.dumps({
    'encrypted_value': b64encode(sealed).decode('utf-8'),
    'key_id': key_id
}))"

# Then PUT the encrypted secret
curl -s -X PUT \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/actions/secrets/API_KEY \
  -d '<output from python script above>'

# List secrets (names only, values hidden)
curl -s \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/actions/secrets \
  | python3 -c "
import sys, json
for s in json.load(sys.stdin)['secrets']:
    print(f\"  {s['name']:30}  updated: {s['updated_at']}\")"
```

Note: For secrets, `gh secret set` is dramatically simpler. If setting secrets is needed and `gh` isn't available, recommend installing it for just that operation.

## 8. Releases

**With gh:**

```bash
gh release create v1.0.0 --title "v1.0.0" --generate-notes
gh release create v2.0.0-rc1 --draft --prerelease --generate-notes
gh release create v1.0.0 ./dist/binary --title "v1.0.0" --notes "Release notes"
gh release list
gh release download v1.0.0 --dir ./downloads
```

**With curl:**

```bash
# Create a release
curl -s -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/releases \
  -d '{
    "tag_name": "v1.0.0",
    "name": "v1.0.0",
    "body": "## Changelog\n- Feature A\n- Bug fix B",
    "draft": false,
    "prerelease": false,
    "generate_release_notes": true
  }'

# List releases
curl -s \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/releases \
  | python3 -c "
import sys, json
for r in json.load(sys.stdin):
    tag = r.get('tag_name', 'no tag')
    print(f\"  {tag:15}  {r['name']:30}  {'draft' if r['draft'] else 'published'}\")"

# Upload a release asset (binary file)
RELEASE_ID=<id_from_create_response>
curl -s -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Content-Type: application/octet-stream" \
  "https://uploads.github.com/repos/$OWNER/$REPO/releases/$RELEASE_ID/assets?name=binary-amd64" \
  --data-binary @./dist/binary-amd64
```

## 9. GitHub Actions Workflows

**With gh:**

```bash
gh workflow list
gh run list --limit 10
gh run view <RUN_ID>
gh run view <RUN_ID> --log-failed
gh run rerun <RUN_ID>
gh run rerun <RUN_ID> --failed
gh workflow run ci.yml --ref main
gh workflow run deploy.yml -f environment=staging
```

**With curl:**

```bash
# List workflows
curl -s \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/actions/workflows \
  | python3 -c "
import sys, json
for w in json.load(sys.stdin)['workflows']:
    print(f\"  {w['id']:10}  {w['name']:30}  {w['state']}\")"

# List recent runs
curl -s \
  -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/$OWNER/$REPO/actions/runs?per_page=10" \
  | python3 -c "
import sys, json
for r in json.load(sys.stdin)['workflow_runs']:
    print(f\"  Run {r['id']}  {r['name']:30}  {r['conclusion'] or r['status']}\")"

# Download failed run logs
RUN_ID=<run_id>
curl -s -L \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/actions/runs/$RUN_ID/logs \
  -o /tmp/ci-logs.zip
cd /tmp && unzip -o ci-logs.zip -d ci-logs

# Re-run a failed workflow
curl -s -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/actions/runs/$RUN_ID/rerun

# Re-run only failed jobs
curl -s -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/actions/runs/$RUN_ID/rerun-failed-jobs

# Trigger a workflow manually (workflow_dispatch)
WORKFLOW_ID=<workflow_id_or_filename>
curl -s -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/actions/workflows/$WORKFLOW_ID/dispatches \
  -d '{"ref": "main", "inputs": {"environment": "staging"}}'
```

## 10. Gists

**With gh:**

```bash
gh gist create script.py --public --desc "Useful script"
gh gist list
```

**With curl:**

```bash
# Create a gist
curl -s -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/gists \
  -d '{
    "description": "Useful script",
    "public": true,
    "files": {
      "script.py": {"content": "print(\"hello\")"}
    }
  }'

# List your gists
curl -s \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/gists \
  | python3 -c "
import sys, json
for g in json.load(sys.stdin):
    files = ', '.join(g['files'].keys())
    print(f\"  {g['id']}  {g['description'] or '(no desc)':40}  {files}\")"
```

## 11. Hermetic Backup Workflow: Sync Workspace to GitHub

When backing up agent/team state (e.g., Hermes multi-agent system), follow this pattern to keep backups clean, secure, and automated:

### Scenario
- You have a local workspace with sensitive files (`.env`, tokens, API keys)
- You want to push periodic backups to GitHub without exposing secrets
- Remote repo branch structure may diverge from local (`master` vs `main`)

### Key Constraints

- Never push `.env` files or tokens — use `.gitignore` to exclude them
- Prefer allowlist-based backups for agent/team memory. Do **not** use `git add .` or `git add -A` when only a subset should be backed up.
- Detect and reconcile branch divergence before push
- Use descriptive commit messages for backup identification
- If a bad backup already reached the remote (for example `.env`, `config.yaml`, `cron/`, `memories/`, or full `skills/`), do not pull it into the clean local tree. Inspect `git diff --name-status HEAD..origin/main`, get explicit approval, then rewrite the remote with `git push --force-with-lease` from the verified clean tree.

### Full Redacted Hermes Config Backup Pattern

Use this when a scheduled job must back up a Hermes home directory (for example `.env`, `config.yaml`, `skills/`, `agents/`, `cron/`, `memories/`, `workspace/`) into a GitHub repo while replacing secrets with placeholders. This is different from a narrow memory backup: it may intentionally include redacted copies of `.env` and config files.

Key points learned from cron/headless runs:

- Stage into a temp directory first, redact there, then sync the staged tree into the backup repo. Never redact in-place under `~/.hermes`.
- Redact assignment-like secrets (`*_API_KEY`, `*_TOKEN`, `*_SECRET`, `*_PASSWORD`, `client_secret`, `webhook`, bearer/basic auth, common token prefixes) to `[PLACEHOLDER]`.
- If requested, skip `.json` and `.md` redaction because regex replacements can corrupt structured/long-form files; still copy them as-is only if the user accepts that scope.
- Cron environments may not export `GITHUB_TOKEN` even when it exists in `~/.hermes/.env`; load only that variable from `.env` as a fallback, without printing it.
- Do not embed the token in `origin`. Keep `origin` as `https://github.com/owner/repo.git` and push with a temporary `http.https://github.com/.extraheader=Authorization: Basic ...` or a repo-local credential helper.
- Sanitize failures before reporting: scrub `Authorization: Basic ...`, `ghp_...`, `sk-...`, and similar values from exception text and command echoes.
- Verify before reporting success: `git status --porcelain` is clean, `git rev-parse --short HEAD` equals `git rev-parse --short origin/main`, and scan the backup repo for unredacted secret patterns in files that were supposed to be redacted.
- In unattended cron jobs, avoid commands that trigger approval prompts such as `git reset --hard`. If the backup repo should mirror the remote before committing, use `git fetch` followed by `git checkout -B main origin/main`, then write the newly staged backup and commit.
- If a push is rejected because the remote changed, prefer fetch/rebase for normal repos. For generated backup repos where the local tree is fully regenerated from staging each run, it is usually simpler to restart from `origin/main`, reapply the staged backup, recommit, and push.

Minimal Python skeleton:

```python
# 1. copy allowlisted Hermes paths to tempfile staging
# 2. redact staged non-JSON/non-MD text files only
# 3. remove the same top-level backup paths from repo, then copy staged paths in
# 4. git add --all && git commit --allow-empty -m "backup: YYYY-MM-DD Hermes configuration (...)"
# 5. git -c http.https://github.com/.extraheader="Authorization: Basic <base64 x-access-token:TOKEN>" push origin main
# 6. verify clean tree, remote HEAD, and no unredacted secret-pattern hits
```

### Focused Agent/Memory Backup Pattern

Use this when the repository is meant to store only agent memory or team state, not a full Hermes/system backup.

```bash
cd /path/to/workspace

# Stage explicit allowlist only — never `git add .` for focused backups.
git add agents TEAM_GOALS.md PROJECT_STATUS.md .gitignore

# Verify the staged set before committing.
git diff --cached --name-only

# Abort if anything outside the allowlist is staged.
bad=$(git diff --cached --name-only | grep -Ev '^(\.gitignore|TEAM_GOALS\.md|PROJECT_STATUS\.md|agents/[^/]+/(SOUL|notes)\.md)$' || true)
if [ -n "$bad" ]; then
  echo "ERROR: files outside focused backup scope:" >&2
  echo "$bad" >&2
  git reset -- $bad
  exit 1
fi

if ! git diff --cached --quiet; then
  git commit -m "agents: daily memory backup $(date +%F)"
else
  echo "No changes in focused backup scope."
fi

GIT_TERMINAL_PROMPT=0 git push origin HEAD:main
```

If the remote already contains a bad broad backup, verify the clean tree before force-pushing:

```bash
git fetch origin main
git diff --name-status HEAD..origin/main | sed -n '1,120p'

# Only after explicit approval for history rewrite:
files=$(git ls-tree -r --name-only HEAD)
bad=$(printf '%s\n' "$files" | grep -Ev '^(\.gitignore|TEAM_GOALS\.md|PROJECT_STATUS\.md|agents/[^/]+/(SOUL|notes)\.md)$' || true)
test -z "$bad" || { echo "Tree is not clean:"; echo "$bad"; exit 1; }

git push --force-with-lease origin HEAD:main
```

### Step-by-Step Flow

#### Step 1: Create `.gitignore` (hermetic exclusion)

```bash
cat > .gitignore << 'EOF'
# Secrets & Tokens
.env
.env.*
*.env
.env.local
*.pem
*.key
*.p12
*.jks
secrets/
*token*
*credential*

# IDE & Editor
.idea/
.vscode/
*.swp
.DS_Store

# OS Temp & Cache
__pycache__/
*.pyc
.cache/
tmp/
*.tmp

# Logs & Output
logs/
*.log

# Large/Binary
*.zip
*.tar.gz
dist/
build/

# Private/Personal
*.private
.local/
EOF
```

#### Step 2: Detect & Reconcile Branch Divergence

```bash
# Check current branch vs remote
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
REMOTE_BRANCH=$(git rev-parse --abbrev-ref origin/HEAD | sed 's|origin/||')

if [ "$CURRENT_BRANCH" != "$REMOTE_BRANCH" ]; then
  echo "⚠ Branch mismatch: local=$CURRENT_BRANCH remote=$REMOTE_BRANCH"
  # Option A: Push local to remote (creates new remote branch)
  # git push -u origin $CURRENT_BRANCH:$REMOTE_BRANCH
  # Option B: Override remote with local (force push — use with caution)
  # git push --force -u origin $CURRENT_BRANCH:$REMOTE_BRANCH
fi

# Or merge diverged branches
git checkout $REMOTE_BRANCH
git merge $CURRENT_BRANCH 2>/dev/null || echo "Diverged — use rebase or force-push"
```

#### Step 3: Cleanup Redundant Structure

Before backup, remove nested repo structures that cause confusion:

```bash
# Remove subdirectory repos created during clone attempts
rm -rf hermes-backup/ temp-clone/ 2>/dev/null
```

#### Step 4: Commit & Push with Backup Metadata

```bash
# Add all changes (except .gitignore rules)
git add -A

# Commit with timestamped message
git commit -m "Backup: $(date '+%Y-%m-%d %H:%M:%S') - $(pwd | xargs basename) workspace sync"

# Push to remote
git push origin $(git rev-parse --abbrev-ref HEAD)
```

### Example: One-Shot Backup Command

```bash
# Full hermetic backup flow in one command
cd /path/to/workspace && \
git add .gitignore && \
git commit -m "Add .gitignore to exclude sensitive files (tokens, .env, keys) and cache/IDE folders" && \
git add -A && \
git commit -m "Backup: hermetic sync - $(date '+%Y-%m-%d %H:%M:%S') - $(pwd | xargs basename)" && \
git push origin main
```

### Quick Reference: Hermetic Backup Checklist

| Check | Command | Action |
|-------|---------|--------|
| `.gitignore` exists? | `test -f .gitignore` | Create if missing |
| `.env` excluded? | `git check-ignore .env` | Must return `.env` |
| Branch divergence? | `git rev-parse --abbrev-ref HEAD` + `origin/HEAD` | Reconcile if mismatch |
| Nested repos? | `find . -type d -name .git` | Remove sub-repos |
| Commit message descriptive? | `git log -1 --oneline` | Include timestamp/workspace name |

---

## Quick Reference Table

| Action | gh | git + curl |
|--------|-----|-----------|
| Clone | `gh repo clone o/r` | `git clone https://github.com/o/r.git` |
| Create repo | `gh repo create name --public` | `curl POST /user/repos` |
| Fork | `gh repo fork o/r --clone` | `curl POST /repos/o/r/forks` + `git clone` |
| Repo info | `gh repo view o/r` | `curl GET /repos/o/r` |
| Edit settings | `gh repo edit --...` | `curl PATCH /repos/o/r` |
| Create release | `gh release create v1.0` | `curl POST /repos/o/r/releases` |
| List workflows | `gh workflow list` | `curl GET /repos/o/r/actions/workflows` |
| Rerun CI | `gh run rerun ID` | `curl POST /repos/o/r/actions/runs/ID/rerun` |
| Set secret | `gh secret set KEY` | `curl PUT /repos/o/r/actions/secrets/KEY` (+ encryption) |
