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

In Codex sandboxed shells, `gh auth status`, `gh repo view`, or `gh api` can fail with authorization-looking messages because the sandbox cannot read credentials, keychain state, config, or the network. Treat that as a possible sandbox problem first: rerun the same `gh` command with shell escalation before concluding GitHub CLI is unauthenticated or before changing project publish authorization.

Inspect optional config without requiring a JSON parser:

```powershell
if (Test-Path -LiteralPath '.\.agents-sync.json') {
  Get-Content -Raw -Encoding UTF8 -LiteralPath '.\.agents-sync.json'
}
```

Persist project-local publish authorization after the user explicitly approves publishing only the root AI-facing `AGENTS.md`:

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
  note = "User authorized agents-md-sync to publish only this project's root AI-facing AGENTS.md to the central rules repository branch for this project."
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

Publish only local root `AGENTS.md` to the central repository branch named after the current GitHub owner/repo. The publish branch must contain no project `docs/`, `skills/`, source, configs, reports, or other files:

```powershell
$localAgentsPath = Join-Path (Get-Location) 'AGENTS.md'
if (-not (Test-Path -LiteralPath $localAgentsPath -PathType Leaf)) {
  Write-Error "Missing root AGENTS.md. agents-md-sync only publishes the target project's root AI-facing AGENTS.md."
  exit 1
}

$tmp = Join-Path $env:TEMP ('agents-md-sync-publish-' + [Guid]::NewGuid().ToString('N'))
git clone --no-checkout git@github.com:tiankongzhise/agents_repo.git $tmp
git -C $tmp fetch origin main

$remoteAgentsPath = Join-Path $tmp '.remote-AGENTS.md'
$remoteHasSameAgents = $false
$remoteRef = "refs/remotes/origin/$publishBranch"
git -C $tmp fetch origin "${publishBranch}:$remoteRef" 2>$null
if ($LASTEXITCODE -eq 0) {
  git -C $tmp show "${remoteRef}:AGENTS.md" 2>$null | Set-Content -Encoding UTF8 -LiteralPath $remoteAgentsPath
  $remotePaths = @(git -C $tmp ls-tree -r --name-only $remoteRef)
  if (
    (Test-Path -LiteralPath $remoteAgentsPath) -and
    ((Get-FileHash -LiteralPath $remoteAgentsPath).Hash -eq (Get-FileHash -LiteralPath $localAgentsPath).Hash) -and
    $remotePaths.Count -eq 1 -and
    $remotePaths[0] -eq 'AGENTS.md'
  ) {
    $remoteHasSameAgents = $true
  }
}

if ($remoteHasSameAgents) {
  Write-Output "Central publish branch already has the same AGENTS.md; skipping commit and push."
} else {
  git -C $tmp switch --orphan $publishBranch
  Remove-Item -LiteralPath $remoteAgentsPath -Force -ErrorAction SilentlyContinue
  Copy-Item -LiteralPath $localAgentsPath -Destination (Join-Path $tmp 'AGENTS.md') -Force
  git -C $tmp add -- AGENTS.md
  $status = git -C $tmp status --porcelain
  $unexpected = $status | Where-Object { $_ -notmatch '^[ MADRCUA?!][ MADRCUA?!] AGENTS\.md$' }
  if ($unexpected) {
    Write-Error "Refusing to publish unexpected paths to central repository:`n$($unexpected -join "`n")"
    exit 1
  }
  git -C $tmp commit -m "同步 $publishBranch 的 AGENTS.md"
  git -C $tmp push -u origin $publishBranch --force-with-lease
}
```

If the remote publish branch already contains only the same `AGENTS.md`, skip the commit and push.

## Bash

Check tools:

```bash
git --version
gh --version
gh auth status
```

In Codex sandboxed shells, `gh auth status`, `gh repo view`, or `gh api` can fail with authorization-looking messages because the sandbox cannot read credentials, keychain state, config, or the network. Treat that as a possible sandbox problem first: rerun the same `gh` command with shell escalation before concluding GitHub CLI is unauthenticated or before changing project publish authorization.

Inspect optional config without requiring a JSON parser:

```bash
test -f .agents-sync.json && cat .agents-sync.json
```

Persist project-local publish authorization after the user explicitly approves publishing only the root AI-facing `AGENTS.md`:

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
    "note": "User authorized agents-md-sync to publish only this project'\''s root AI-facing AGENTS.md to the central rules repository branch for this project."
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
    "note": "User authorized agents-md-sync to publish only this project's root AI-facing AGENTS.md to the central rules repository branch for this project."
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

Publish only local root `AGENTS.md` to the central repository branch named after the current GitHub owner/repo. The publish branch must contain no project `docs/`, `skills/`, source, configs, reports, or other files:

```bash
local_agents_path="$PWD/AGENTS.md"
if [ ! -f "$local_agents_path" ]; then
  printf '%s\n' "Missing root AGENTS.md. agents-md-sync only publishes the target project's root AI-facing AGENTS.md." >&2
  exit 1
fi

tmp="$(mktemp -d)"
git clone --no-checkout git@github.com:tiankongzhise/agents_repo.git "$tmp"
git -C "$tmp" fetch origin main

remote_has_same_agents=false
remote_ref="refs/remotes/origin/$publish_branch"
if git -C "$tmp" fetch origin "$publish_branch:$remote_ref" >/dev/null 2>&1 &&
   git -C "$tmp" show "$remote_ref:AGENTS.md" > "$tmp/.remote-AGENTS.md" 2>/dev/null; then
  remote_paths="$(git -C "$tmp" ls-tree -r --name-only "$remote_ref")"
  if cmp -s "$local_agents_path" "$tmp/.remote-AGENTS.md" &&
     [ "$remote_paths" = "AGENTS.md" ]; then
    remote_has_same_agents=true
  fi
fi

if [ "$remote_has_same_agents" = true ]; then
  printf '%s\n' "Central publish branch already has the same AGENTS.md; skipping commit and push."
else
  git -C "$tmp" switch --orphan "$publish_branch"
  rm -f "$tmp/.remote-AGENTS.md"
  cp "$local_agents_path" "$tmp/AGENTS.md"
  git -C "$tmp" add -- AGENTS.md
  status="$(git -C "$tmp" status --porcelain)"
  unexpected="$(printf '%s\n' "$status" | sed '/^[ MADRCUA?!][ MADRCUA?!] AGENTS\.md$/d;/^$/d')"
  if [ -n "$unexpected" ]; then
    printf '%s\n%s\n' "Refusing to publish unexpected paths to central repository:" "$unexpected" >&2
    exit 1
  fi
  git -C "$tmp" commit -m "同步 $publish_branch 的 AGENTS.md"
  git -C "$tmp" push -u origin "$publish_branch" --force-with-lease
fi
```

If the remote publish branch already contains only the same `AGENTS.md`, skip the commit and push.

## Merge Checklist

- Add only central constraints that are general, applicable, and not already represented locally.
- Preserve all hand-authored local content outside the managed block.
- Put automatic additions between `<!-- agents-sync:start ... -->` and `<!-- agents-sync:end -->`.
- Write a report for every automatic update.
- Do not publish if `.agents-sync.json` has `enabled: false` or `auto_publish_on_agents_change: false`.
- Do not publish unless `.agents-sync.json` has a matching `publish_authorization.granted: true` block created after explicit user authorization.
- Publish only the target project's root AI-facing `AGENTS.md`, copied to central `AGENTS.md`.
- Never publish project `docs/`, `skills/`, source, configs, reports, generated files, or any path other than `AGENTS.md`.
- Never use `git add .` or directory copies in the central temporary clone; stage exactly `AGENTS.md`.
