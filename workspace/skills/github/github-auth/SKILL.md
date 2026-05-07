---
name: github-auth
description: "GitHub auth setup: HTTPS tokens, SSH keys, gh CLI login."
version: 1.1.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [GitHub, Authentication, Git, gh-cli, SSH, Setup]
    related_skills: [github-pr-workflow, github-code-review, github-issues, github-repo-management]
---

# GitHub Authentication Setup

This skill sets up authentication so the agent can work with GitHub repositories, PRs, issues, and CI. It covers three paths:

- **`git` (always available)** — uses HTTPS personal access tokens or SSH keys
- **`gh` CLI (if installed)** — richer GitHub API access with a simpler auth flow
- **Headless / Automation contexts** — secure `.env` integration (token masking, one-time session usage)

## Headless / Automation Contexts

When you're in an automated or one-time context (cron jobs, backup scripts, headless terminals), follow this flow to safely use tokens stored in `.env`, especially when the platform masks your secrets.

### Key Constraints

- never store tokens in memory (session or history), even temporarily  
- tokens that are masked (e.g. `***`) by the platform are unusable for reading — you must get them from environment variables or user input  
- prefer export → use → discard patterns (not writing credentials to disk)

### Flow: `.env` → Session Token (Non-Interactive)

1. Detect if `.env` exists and contains `GITHUB_TOKEN=...`
2. *Attempt* to source it, but expect masking — tokens may appear as `***`
3. If masked, fall back to:
   - user re-entry (one-time, session-only)
   - or a known-good environment variable (e.g. previously exported)
4. Set token to `GITHUB_TOKEN=...` in current shell session
5. Use it immediately, then unset (`unset GITHUB_TOKEN`) if desired

**Example:**

```bash
# 1. Check if .env exists
if [ -f "$HOME/.hermes/.env" ]; then
  echo "✓ .env found"
else
  echo "✗ .env not found — manual entry required"
  exit 1
fi

# 2. Source .env (may return masked token)
source "$HOME/.hermes/.env"

# 3. Check if token is usable
if [ -n "$GITHUB_TOKEN" ] && [ "$GITHUB_TOKEN" != "***" ]; then
  echo "✓ Token extracted"
else
  echo "⚠ Token masked or empty — fall back to user input"
  read -rsp "Enter GITHUB_TOKEN: " GITHUB_TOKEN
  echo
fi

# 4. Use token (clone, push, etc.)
git clone "https://$GITHUB_TOKEN@github.com/$OWNER/$REPO.git"

# 5. (optional) Clear token after success
unset GITHUB_TOKEN
```

### Secure Alternatives (When `.env` Is Masked)

| Situation | Recommended Action |
|----------|--------------------|
| `GITHUB_TOKEN=***` in `.env` | User re-enters token once (session-only), then unset |
| Token in `~/.git-credentials` | Extract via `sed` (no `.env` needed) |
| Cron/automation jobs | Use a separate file (e.g. `~/.secrets/github_token.txt`) + `chmod 600` |

### Fixing Stale HTTPS Credentials in Automation Repos

Use this when `git push` fails with `remote: Invalid username or token` but a valid `GITHUB_TOKEN` exists in `~/.hermes/.env`. Common cause: the repo's remote URL or credential cache contains an old token. Keep the remote URL clean, verify the token without printing it, then use a repo-local credential helper that reads `.env` at runtime.

```bash
set -euo pipefail
WORKDIR="/path/to/repo"
OWNER="DifanaDAP"
REPO="hermes-backup"
USERNAME="DifanaDAP"
ENV_FILE="$HOME/.hermes/.env"

cd "$WORKDIR"
set -a; . "$ENV_FILE"; set +a
: "${GITHUB_TOKEN:?GITHUB_TOKEN missing}"

# Preflight auth without storing or printing the token.
GIT_TERMINAL_PROMPT=0 git ls-remote "https://${USERNAME}:${GITHUB_TOKEN}@github.com/${OWNER}/${REPO}.git" HEAD >/dev/null

# Remove embedded/stale credentials from the configured remote.
git remote set-url origin "https://github.com/${OWNER}/${REPO}.git"

# Repo-local helper: credentials are read from .env only when git asks.
cat > .git/git-credential-env <<'PY'
#!/usr/bin/env python3
import sys
from pathlib import Path
if len(sys.argv) < 2 or sys.argv[1] != "get":
    raise SystemExit(0)
vals = {}
p = Path.home() / ".hermes" / ".env"
if p.exists():
    for line in p.read_text(errors="ignore").splitlines():
        s = line.strip()
        if not s or s.startswith("#") or "=" not in s:
            continue
        k, v = s.split("=", 1)
        vals[k.strip()] = v.strip().strip('"').strip("'")
tok = vals.get("GITHUB_TOKEN") or vals.get("GH_TOKEN")
if tok:
    print("username=DifanaDAP")
    print(f"password={tok}")
PY
chmod 700 .git/git-credential-env
git config --local credential."https://github.com".username "$USERNAME"
git config --local credential."https://github.com".helper "$WORKDIR/.git/git-credential-env"

# Verify normal push uses the helper. Sanitize any command output that might include URLs.
GIT_TERMINAL_PROMPT=0 git push origin HEAD:main
```

Security notes:
- Do not print token values or commit `.git/git-credential-env`; it lives under `.git/` and stays repo-local.
- Prefer this to embedding PATs in remote URLs for scheduled jobs.
- If the preflight `ls-remote` fails, the token is expired, lacks repo scope/access, or the username/repo target is wrong.

---

## Detection Flow

When a user asks you to work with GitHub, run this check first:

## Detection Flow

When a user asks you to work with GitHub, run this check first:

```bash
# Check what's available
git --version
gh --version 2>/dev/null || echo "gh not installed"

# Check if already authenticated
gh auth status 2>/dev/null || echo "gh not authenticated"
git config --global credential.helper 2>/dev/null || echo "no git credential helper"
```

**Decision tree:**
1. If `gh auth status` shows authenticated → you're good, use `gh` for everything
2. If `gh` is installed but not authenticated → use "gh auth" method below
3. If `gh` is not installed → use "git-only" method below (no sudo needed)

---

## Method 1: Git-Only Authentication (No gh, No sudo)

This works on any machine with `git` installed. No root access needed.

### Option A: HTTPS with Personal Access Token (Recommended)

This is the most portable method — works everywhere, no SSH config needed.

**Step 1: Create a personal access token**

Tell the user to go to: **https://github.com/settings/tokens**

- Click "Generate new token (classic)"
- Give it a name like "hermes-agent"
- Select scopes:
  - `repo` (full repository access — read, write, push, PRs)
  - `workflow` (trigger and manage GitHub Actions)
  - `read:org` (if working with organization repos)
- Set expiration (90 days is a good default)
- Copy the token — it won't be shown again

**Step 2: Configure git to store the token**

```bash
# Set up the credential helper to cache credentials
# "store" saves to ~/.git-credentials in plaintext (simple, persistent)
git config --global credential.helper store

# Now do a test operation that triggers auth — git will prompt for credentials
# Username: <their-github-username>
# Password: <paste the personal access token, NOT their GitHub password>
git ls-remote https://github.com/<their-username>/<any-repo>.git
```

After entering credentials once, they're saved and reused for all future operations.

**Alternative: cache helper (credentials expire from memory)**

```bash
# Cache in memory for 8 hours (28800 seconds) instead of saving to disk
git config --global credential.helper 'cache --timeout=28800'
```

**Alternative: set the token directly in the remote URL (per-repo)**

Avoid this unless it is truly temporary: embedding tokens in remote URLs can leak through `git remote -v`, shell history, logs, and copied config.

```bash
# Temporary only — avoid leaving this configured permanently.
git remote set-url origin https://<username>:<token>@github.com/<owner>/<repo>.git
```

**Safer automation alternative: per-repo credential helper reading `.env` at runtime**

Use this for cron/headless backups when `GITHUB_TOKEN` lives in `~/.hermes/.env`. It keeps the remote URL clean and avoids printing or storing the token in `.git/config`.

```bash
cd /path/to/repo
cat > .git/git-credential-hermes-env <<'PY'
#!/usr/bin/env python3
import sys
from pathlib import Path

if len(sys.argv) < 2 or sys.argv[1] != "get":
    sys.exit(0)

env_path = Path.home() / ".hermes" / ".env"
vals = {}
if env_path.exists():
    for line in env_path.read_text(errors="ignore").splitlines():
        s = line.strip()
        if not s or s.startswith("#") or "=" not in s:
            continue
        k, v = s.split("=", 1)
        vals[k.strip()] = v.strip().strip('"').strip("'")

token = vals.get("GITHUB_TOKEN") or vals.get("GH_TOKEN")
if token:
    print("username=<github-username>")
    print(f"password={token}")
PY
chmod 700 .git/git-credential-hermes-env

git remote set-url origin https://github.com/<owner>/<repo>.git
git config --local credential."https://github.com".username "<github-username>"
git config --local credential."https://github.com".helper "$(pwd)/.git/git-credential-hermes-env"

# Verify without exposing token.
GIT_TERMINAL_PROMPT=0 git ls-remote origin HEAD >/dev/null && echo "auth ok"
```

**Step 3: Configure git identity**

```bash
# Required for commits — set name and email
git config --global user.name "Their Name"
git config --global user.email "their-email@example.com"
```

**Step 4: Verify**

```bash
# Test push access (this should work without any prompts now)
git ls-remote https://github.com/<their-username>/<any-repo>.git

# Verify identity
git config --global user.name
git config --global user.email
```

### Option B: SSH Key Authentication

Good for users who prefer SSH or already have keys set up.

**Step 1: Check for existing SSH keys**

```bash
ls -la ~/.ssh/id_*.pub 2>/dev/null || echo "No SSH keys found"
```

**Step 2: Generate a key if needed**

```bash
# Generate an ed25519 key (modern, secure, fast)
ssh-keygen -t ed25519 -C "their-email@example.com" -f ~/.ssh/id_ed25519 -N ""

# Display the public key for them to add to GitHub
cat ~/.ssh/id_ed25519.pub
```

Tell the user to add the public key at: **https://github.com/settings/keys**
- Click "New SSH key"
- Paste the public key content
- Give it a title like "hermes-agent-<machine-name>"

**Step 3: Test the connection**

```bash
ssh -T git@github.com
# Expected: "Hi <username>! You've successfully authenticated..."
```

**Step 4: Configure git to use SSH for GitHub**

```bash
# Rewrite HTTPS GitHub URLs to SSH automatically
git config --global url."git@github.com:".insteadOf "https://github.com/"
```

**Step 5: Configure git identity**

```bash
git config --global user.name "Their Name"
git config --global user.email "their-email@example.com"
```

---

## Method 2: gh CLI Authentication

If `gh` is installed, it handles both API access and git credentials in one step.

### Interactive Browser Login (Desktop)

```bash
gh auth login
# Select: GitHub.com
# Select: HTTPS
# Authenticate via browser
```

### Token-Based Login (Headless / SSH Servers)

```bash
echo "<THEIR_TOKEN>" | gh auth login --with-token

# Set up git credentials through gh
gh auth setup-git
```

### Verify

```bash
gh auth status
```

---

## Using the GitHub API Without gh

When `gh` is not available, you can still access the full GitHub API using `curl` with a personal access token. This is how the other GitHub skills implement their fallbacks.

### Setting the Token for API Calls

```bash
# Option 1: Export as env var (preferred — keeps it out of commands)
export GITHUB_TOKEN="<token>"

# Then use in curl calls:
curl -s -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/user
```

### Extracting the Token from Git Credentials

If git credentials are already configured (via credential.helper store), the token can be extracted:

```bash
# Read from git credential store
grep "github.com" ~/.git-credentials 2>/dev/null | head -1 | sed 's|https://[^:]*:\([^@]*\)@.*|\1|'
```

### Helper: Detect Auth Method

Use this pattern at the start of any GitHub workflow:

```bash
# Try gh first, fall back to git + curl
if command -v gh &>/dev/null && gh auth status &>/dev/null; then
  echo "AUTH_METHOD=gh"
elif [ -n "$GITHUB_TOKEN" ]; then
  echo "AUTH_METHOD=curl"
elif [ -f ~/.hermes/.env ] && grep -q "^GITHUB_TOKEN=" ~/.hermes/.env; then
  export GITHUB_TOKEN=$(grep "^GITHUB_TOKEN=" ~/.hermes/.env | head -1 | cut -d= -f2 | tr -d '\n\r')
  echo "AUTH_METHOD=curl"
elif grep -q "github.com" ~/.git-credentials 2>/dev/null; then
  export GITHUB_TOKEN=$(grep "github.com" ~/.git-credentials | head -1 | sed 's|https://[^:]*:\([^@]*\)@.*|\1|')
  echo "AUTH_METHOD=curl"
else
  echo "AUTH_METHOD=none"
  echo "Need to set up authentication first"
fi
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `git push` asks for password | GitHub disabled password auth. Use a personal access token as the password, or switch to SSH |
| `remote: Permission to X denied` | Token may lack `repo` scope — regenerate with correct scopes |
| `fatal: Authentication failed` | Cached credentials may be stale — run `git credential reject` then re-authenticate |
| `ssh: connect to host github.com port 22: Connection refused` | Try SSH over HTTPS port: add `Host github.com` with `Port 443` and `Hostname ssh.github.com` to `~/.ssh/config` |
| Credentials not persisting | Check `git config --global credential.helper` — must be `store` or `cache` |
| Multiple GitHub accounts | Use SSH with different keys per host alias in `~/.ssh/config`, or per-repo credential URLs |
| `gh: command not found` + no sudo | Use git-only Method 1 above — no installation needed |
