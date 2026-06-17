# Command Templates

Use these templates when executing `agents-md-sync`. Prefer PowerShell on Windows and Bash on Unix-like systems. These commands intentionally avoid Python, Node, Go, Rust, and project-specific runtimes.

## Contents

- PowerShell
- Bash
- Merge Checklist

## PowerShell

Set UTF-8 before reading or writing Chinese text:

```powershell
$OutputEncoding = [System.Text.UTF8Encoding]::new()
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new()
[Console]::InputEncoding = [System.Text.UTF8Encoding]::new()
```

Check tools:

```powershell
git --version
gh --version
gh auth status
```

Inspect optional config without requiring a JSON parser:

```powershell
if (Test-Path -LiteralPath '.\.agents-sync.json') {
  Get-Content -Raw -Encoding UTF8 -LiteralPath '.\.agents-sync.json'
}
```

Persist project-local publish authorization after the user explicitly approves it:

```powershell
$configPath = '.\.agents-sync.json'
$now = (Get-Date).ToUniversalTime().ToString('yyyy-MM-ddTHH:mm:ssZ')
$authorization = [ordered]@{
  granted = $true
  scope = 'push_local_agents_to_central_repo'
  central_repo = 'git@github.com:tiankongzhise/agents_repo.git'
  local_agents_path = 'AGENTS.md'
  granted_at = $now
  granted_by = 'user'
  note = "User authorized agents-md-sync to publish this project's AGENTS.md to the central rules repository branch for this project."
}

if (Test-Path -LiteralPath $configPath) {
  $config = Get-Content -Raw -Encoding UTF8 -LiteralPath $configPath | ConvertFrom-Json
} else {
  $config = [ordered]@{
    enabled = $true
    auto_update = $true
    auto_publish_on_agents_change = $true
    central_repo = 'git@github.com:tiankongzhise/agents_repo.git'
    central_branch = 'main'
    central_agents_path = 'AGENTS.md'
    local_agents_path = 'AGENTS.md'
    change_log_dir = 'docs/agents-sync'
  }
}

$config | Add-Member -NotePropertyName publish_authorization -NotePropertyValue $authorization -Force
$config | ConvertTo-Json -Depth 10 | Set-Content -Encoding UTF8 -LiteralPath $configPath
```

Check project-local publish authorization before pushing:

```powershell
$configPath = '.\.agents-sync.json'
$publishAuthorized = $false
if (Test-Path -LiteralPath $configPath) {
  $config = Get-Content -Raw -Encoding UTF8 -LiteralPath $configPath | ConvertFrom-Json
  $auth = $config.publish_authorization
  $publishAuthorized = (
    $null -ne $auth -and
    $auth.granted -eq $true -and
    $auth.scope -eq 'push_local_agents_to_central_repo' -and
    $auth.central_repo -eq 'git@github.com:tiankongzhise/agents_repo.git' -and
    $auth.local_agents_path -eq 'AGENTS.md'
  )
}
if (-not $publishAuthorized) {
  Write-Error "Missing project-local publish authorization in .agents-sync.json. Ask the user before pushing AGENTS.md."
  exit 1
}
```

Fetch central `AGENTS.md` with `gh`:

```powershell
$centralAgents = gh api 'repos/tiankongzhise/agents_repo/contents/AGENTS.md' -H 'Accept: application/vnd.github.raw+json'
```

Fetch central `AGENTS.md` with plain Git:

```powershell
$tmp = Join-Path $env:TEMP ('agents-md-sync-' + [Guid]::NewGuid().ToString('N'))
New-Item -ItemType Directory -Force -Path $tmp | Out-Null
git -C $tmp init
git -C $tmp remote add origin git@github.com:tiankongzhise/agents_repo.git
git -C $tmp fetch --depth=1 origin main
$centralAgents = git -C $tmp show FETCH_HEAD:AGENTS.md
```

If SSH is blocked, replace the remote URL with:

```powershell
git -C $tmp remote set-url origin https://github.com/tiankongzhise/agents_repo.git
git -C $tmp fetch --depth=1 origin main
```

Resolve the current GitHub owner/repository full name with `gh`:

```powershell
$repoFullName = gh repo view --json nameWithOwner --jq .nameWithOwner
$repoParts = ($repoFullName -replace '\\', '/').Trim('/') -split '/'
if ($repoParts.Count -ne 2 -or [string]::IsNullOrWhiteSpace($repoParts[0]) -or [string]::IsNullOrWhiteSpace($repoParts[1])) {
  Write-Error "Unable to resolve GitHub owner/repo from gh output: $repoFullName"
  exit 1
}
$repoOwner = ($repoParts[0].ToLowerInvariant() -replace '[^a-z0-9._-]', '-')
$repoName = ($repoParts[1].ToLowerInvariant() -replace '[^a-z0-9._-]', '-')
$publishBranch = "$repoOwner/$repoName"
```

Resolve the current GitHub owner/repository full name with Git remote parsing:

```powershell
$originUrl = (git remote get-url origin) -replace '\\', '/'
if ($originUrl -notmatch 'github\.com[:/](?<owner>[^/]+)/(?<repo>[^/]+?)(?:\.git)?/?$') {
  Write-Error "Unable to resolve GitHub owner/repo from origin URL: $originUrl"
  exit 1
}
$repoOwner = ($Matches.owner.ToLowerInvariant() -replace '[^a-z0-9._-]', '-')
$repoName = ($Matches.repo.ToLowerInvariant() -replace '[^a-z0-9._-]', '-')
$publishBranch = "$repoOwner/$repoName"
```

Create a change report directory:

```powershell
$reportDir = Join-Path (Get-Location) 'docs\agents-sync'
New-Item -ItemType Directory -Force -Path $reportDir | Out-Null
$stamp = Get-Date -Format 'yyyyMMdd-HHmmss'
$reportPath = Join-Path $reportDir "$stamp-agents-sync.md"
```

Publish local `AGENTS.md` to the central repository branch named after the current GitHub owner/repo:

```powershell
$tmp = Join-Path $env:TEMP ('agents-md-sync-publish-' + [Guid]::NewGuid().ToString('N'))
git clone git@github.com:tiankongzhise/agents_repo.git $tmp
git -C $tmp fetch origin main
git -C $tmp switch -C $publishBranch origin/main
Copy-Item -LiteralPath '.\AGENTS.md' -Destination (Join-Path $tmp 'AGENTS.md') -Force
if (git -C $tmp status --short AGENTS.md) {
  git -C $tmp add AGENTS.md
  git -C $tmp commit -m "同步 $publishBranch 的 AGENTS.md"
  git -C $tmp push -u origin $publishBranch --force-with-lease
}
```

If `git status --short` is empty before commit, skip the commit and push.

## Bash

Check tools:

```bash
git --version
gh --version
gh auth status
```

Inspect optional config without requiring a JSON parser:

```bash
test -f .agents-sync.json && cat .agents-sync.json
```

Persist project-local publish authorization after the user explicitly approves it:

```bash
now="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
if command -v jq >/dev/null 2>&1 && [ -f .agents-sync.json ]; then
  tmp_config="$(mktemp)"
  jq --arg now "$now" '.publish_authorization = {
    "granted": true,
    "scope": "push_local_agents_to_central_repo",
    "central_repo": "git@github.com:tiankongzhise/agents_repo.git",
    "local_agents_path": "AGENTS.md",
    "granted_at": $now,
    "granted_by": "user",
    "note": "User authorized agents-md-sync to publish this project'\''s AGENTS.md to the central rules repository branch for this project."
  }' .agents-sync.json > "$tmp_config" && mv "$tmp_config" .agents-sync.json
elif [ -f .agents-sync.json ]; then
  printf '%s\n' 'Existing .agents-sync.json found and jq is unavailable. Preserve existing fields and add publish_authorization manually before publishing.' >&2
  exit 1
else
  cat > .agents-sync.json <<EOF
{
  "enabled": true,
  "auto_update": true,
  "auto_publish_on_agents_change": true,
  "central_repo": "git@github.com:tiankongzhise/agents_repo.git",
  "central_branch": "main",
  "central_agents_path": "AGENTS.md",
  "local_agents_path": "AGENTS.md",
  "change_log_dir": "docs/agents-sync",
  "publish_authorization": {
    "granted": true,
    "scope": "push_local_agents_to_central_repo",
    "central_repo": "git@github.com:tiankongzhise/agents_repo.git",
    "local_agents_path": "AGENTS.md",
    "granted_at": "$now",
    "granted_by": "user",
    "note": "User authorized agents-md-sync to publish this project's AGENTS.md to the central rules repository branch for this project."
  }
}
EOF
fi
```

When `jq` is unavailable, check project-local publish authorization with plain text before pushing:

```bash
auth_block="$(sed -n '/"publish_authorization"[[:space:]]*:/,/^[[:space:]]*}[,]*[[:space:]]*$/p' .agents-sync.json)"
if ! printf '%s\n' "$auth_block" | grep -q '"publish_authorization"' ||
   ! printf '%s\n' "$auth_block" | grep -q '"granted"[[:space:]]*:[[:space:]]*true' ||
   ! printf '%s\n' "$auth_block" | grep -q '"scope"[[:space:]]*:[[:space:]]*"push_local_agents_to_central_repo"' ||
   ! printf '%s\n' "$auth_block" | grep -q '"central_repo"[[:space:]]*:[[:space:]]*"git@github.com:tiankongzhise/agents_repo.git"' ||
   ! printf '%s\n' "$auth_block" | grep -q '"local_agents_path"[[:space:]]*:[[:space:]]*"AGENTS.md"'; then
  printf '%s\n' 'Missing project-local publish authorization in .agents-sync.json. Ask the user before pushing AGENTS.md.' >&2
  exit 1
fi
```

Fetch central `AGENTS.md` with `gh`:

```bash
central_agents="$(gh api repos/tiankongzhise/agents_repo/contents/AGENTS.md -H 'Accept: application/vnd.github.raw+json')"
```

Fetch central `AGENTS.md` with plain Git:

```bash
tmp="$(mktemp -d)"
git -C "$tmp" init
git -C "$tmp" remote add origin git@github.com:tiankongzhise/agents_repo.git
git -C "$tmp" fetch --depth=1 origin main
central_agents="$(git -C "$tmp" show FETCH_HEAD:AGENTS.md)"
```

If SSH is blocked, replace the remote URL with:

```bash
git -C "$tmp" remote set-url origin https://github.com/tiankongzhise/agents_repo.git
git -C "$tmp" fetch --depth=1 origin main
```

Resolve the current GitHub owner/repository full name with `gh`:

```bash
repo_full_name="$(gh repo view --json nameWithOwner --jq .nameWithOwner)"
repo_owner="${repo_full_name%%/*}"
repo_name="${repo_full_name#*/}"
if [ "$repo_full_name" = "$repo_name" ] || [ -z "$repo_owner" ] || [ -z "$repo_name" ]; then
  printf '%s\n' "Unable to resolve GitHub owner/repo from gh output: $repo_full_name" >&2
  exit 1
fi
repo_owner="$(printf '%s' "$repo_owner" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9._-]/-/g')"
repo_name="$(printf '%s' "$repo_name" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9._-]/-/g')"
publish_branch="$repo_owner/$repo_name"
```

Resolve the current GitHub owner/repository full name with Git remote parsing:

```bash
origin_url="$(git remote get-url origin)"
normalized_origin_url="$(printf '%s' "$origin_url" | sed 's#\\#/#g')"
repo_full_name="$(printf '%s' "$normalized_origin_url" | sed -E 's#^.*github\.com[:/]([^/]+/[^/]+)(\.git)?/?$#\1#')"
repo_full_name="${repo_full_name%.git}"
if [ "$repo_full_name" = "$normalized_origin_url" ] || ! printf '%s' "$repo_full_name" | grep -Eq '^[^/]+/[^/]+$'; then
  printf '%s\n' "Unable to resolve GitHub owner/repo from origin URL: $origin_url" >&2
  exit 1
fi
repo_owner="${repo_full_name%%/*}"
repo_name="${repo_full_name#*/}"
repo_owner="$(printf '%s' "$repo_owner" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9._-]/-/g')"
repo_name="$(printf '%s' "$repo_name" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9._-]/-/g')"
publish_branch="$repo_owner/$repo_name"
```

Create a change report directory:

```bash
report_dir="docs/agents-sync"
mkdir -p "$report_dir"
stamp="$(date +%Y%m%d-%H%M%S)"
report_path="$report_dir/$stamp-agents-sync.md"
```

Publish local `AGENTS.md` to the central repository branch named after the current GitHub owner/repo:

```bash
tmp="$(mktemp -d)"
git clone git@github.com:tiankongzhise/agents_repo.git "$tmp"
git -C "$tmp" fetch origin main
git -C "$tmp" switch -C "$publish_branch" origin/main
cp AGENTS.md "$tmp/AGENTS.md"
if [ -n "$(git -C "$tmp" status --short AGENTS.md)" ]; then
  git -C "$tmp" add AGENTS.md
  git -C "$tmp" commit -m "同步 $publish_branch 的 AGENTS.md"
  git -C "$tmp" push -u origin "$publish_branch" --force-with-lease
fi
```

If `git status --short` is empty before commit, skip the commit and push.

## Merge Checklist

- Add only central constraints that are general, applicable, and not already represented locally.
- Preserve all hand-authored local content outside the managed block.
- Put automatic additions between `<!-- agents-sync:start ... -->` and `<!-- agents-sync:end -->`.
- Write a report for every automatic update.
- Do not publish if `.agents-sync.json` has `enabled: false` or `auto_publish_on_agents_change: false`.
- Do not publish unless `.agents-sync.json` has a matching `publish_authorization.granted: true` block created after explicit user authorization.
