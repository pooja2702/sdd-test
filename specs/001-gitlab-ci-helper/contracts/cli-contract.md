# CLI Contract: GitLab CI/CD Helper

**Branch**: `001-gitlab-ci-helper` | **Date**: 2026-03-03

> Defines the command-line interface exposed to developers. This is the primary (and only) external interface of the tool.

## Global Options

| Option              | Short | Type   | Env Variable     | Default               | Description                            |
|---------------------|-------|--------|------------------|-----------------------|----------------------------------------|
| `--gitlab-url`      | `-u`  | str    | `GITLAB_URL`     | `https://gitlab.com`  | GitLab instance URL                    |
| `--token`           | `-t`  | str    | `GITLAB_TOKEN`   | *(required)*          | Personal access token (never logged)   |
| `--project`         | `-p`  | str    | `GITLAB_PROJECT` | *(auto-detect)*       | Project path or ID                     |
| `--help`            | `-h`  | flag   | —                | —                     | Show help and exit                     |
| `--version`         | `-V`  | flag   | —                | —                     | Show version and exit                  |

## Commands

### `cicd mr`

Create a merge request for the current branch, or report the existing one.

**Usage**: `cicd mr [OPTIONS]`

**Options**:

| Option        | Short | Type   | Default | Description                                |
|---------------|-------|--------|---------|--------------------------------------------|
| `--title`     | `-T`  | str    | *(branch name)* | MR title (defaults to branch name) |
| `--target`    |       | str    | *(default branch)* | Target branch for the MR          |
| `--help`      | `-h`  | flag   | —       | Show command help                          |

**Behavior**:

1. Detect current branch (fail if detached HEAD or on master/main)
2. Check if branch exists on remote
   - If not: prompt to push (commit + push if confirmed; exit if declined)
3. Check for existing open MR
   - If exists: print MR URL and exit
4. Create new MR targeting the default branch
5. Print new MR URL

**Exit codes**:

| Code | Meaning                                               |
|------|-------------------------------------------------------|
| 0    | MR created or existing MR found                       |
| 1    | Error: on default branch, detached HEAD, or auth failure |
| 2    | User declined to push (no action taken)               |

**Example output** (new MR):
```
✓ Branch: feature/add-login
✓ Pushed branch to remote
✓ Created merge request
  → https://gitlab.com/group/project/-/merge_requests/42
```

**Example output** (existing MR):
```
✓ Branch: feature/add-login
ℹ Merge request already exists
  → https://gitlab.com/group/project/-/merge_requests/42
```

**Example output** (branch not on remote):
```
✓ Branch: feature/add-login
⚠ Branch does not exist on remote

Push branch to remote? [y/N]: y

✓ Committed local changes (3 files)
✓ Pushed branch to remote
✓ Created merge request
  → https://gitlab.com/group/project/-/merge_requests/42
```

---

### `cicd pipeline`

Fetch pipeline status for the current branch, trigger if none exists, and poll until completion.

**Usage**: `cicd pipeline [OPTIONS]`

**Options**:

| Option           | Short | Type   | Default | Description                              |
|------------------|-------|--------|---------|------------------------------------------|
| `--no-poll`      |       | flag   | False   | Report status once without polling       |
| `--help`         | `-h`  | flag   | —       | Show command help                        |

**Behavior**:

1. Detect current branch
2. Check if branch exists on remote
   - If not: push to remote first
3. Fetch most recent pipeline
   - If none: trigger a new pipeline
4. If pipeline is in a terminal state: report result
5. If pipeline is running/pending: poll every 15 seconds with progress display
6. On completion: report final result with job details
7. On Ctrl+C: report last known status and exit

**Exit codes**:

| Code | Meaning                                               |
|------|-------------------------------------------------------|
| 0    | Pipeline passed                                        |
| 1    | Pipeline failed (details in stdout)                    |
| 2    | Pipeline canceled                                      |
| 3    | Error: no CI config, auth failure, or other error      |

**Example output** (passed):
```
✓ Branch: feature/add-login
✓ Pipeline #1234 — passed

  Jobs:
  ✓ build    (build)    — passed
  ✓ lint     (test)     — passed
  ✓ unittest (test)     — passed
```

**Example output** (failed):
```
✓ Branch: feature/add-login
✗ Pipeline #1234 — failed

  Jobs:
  ✓ build    (build)    — passed
  ✗ lint     (test)     — failed
    Reason: script_failure
  ✓ unittest (test)     — passed
  ~ deploy   (deploy)   — failed (allowed to fail)
    Reason: runner_system_failure
```

**Example output** (polling):
```
✓ Branch: feature/add-login
⟳ Pipeline #1234 — running (polling every 15s, Ctrl+C to stop)

  Jobs:
  ✓ build    (build)    — passed
  ⟳ lint     (test)     — running
  ◌ unittest (test)     — pending
  ◌ deploy   (deploy)   — created
```

---

### `cicd run`

Combined workflow: create MR (if needed) then check pipeline status.

**Usage**: `cicd run [OPTIONS]`

**Options**: Inherits options from both `mr` and `pipeline` commands.

**Behavior**:

1. Execute `cicd mr` logic (create or report MR)
2. Execute `cicd pipeline` logic (fetch/trigger/poll pipeline)

**Exit codes**: Uses the pipeline exit code (0/1/2/3). MR errors exit with code 1.

**Example output**:
```
✓ Branch: feature/add-login
✓ Created merge request
  → https://gitlab.com/group/project/-/merge_requests/42

⟳ Pipeline #1234 — running (polling every 15s, Ctrl+C to stop)

  Jobs:
  ✓ build    (build)    — passed
  ⟳ lint     (test)     — running
  ◌ unittest (test)     — pending

[15 seconds later...]

✓ Pipeline #1234 — passed

  Jobs:
  ✓ build    (build)    — passed
  ✓ lint     (test)     — passed
  ✓ unittest (test)     — passed
```

## Error Messages

All error messages follow this structure:
```
✗ Error: [clear description of the problem]
  → [suggested resolution or next step]
```

| Error Condition         | Message                                                                  |
|-------------------------|--------------------------------------------------------------------------|
| No token                | `✗ Error: GITLAB_TOKEN not set` → `Set via 'export GITLAB_TOKEN=...'`   |
| Invalid token           | `✗ Error: Authentication failed (invalid or expired token)` → `Verify your token at GitLab > Settings > Access Tokens` |
| Network error           | `✗ Error: Cannot reach [url]` → `Check network connection and GITLAB_URL` |
| On default branch       | `✗ Error: Cannot create MR from the default branch (main)` → `Checkout a feature branch first` |
| Detached HEAD           | `✗ Error: HEAD is detached (not on a branch)` → `Checkout a branch first` |
| No CI config            | `✗ Error: No .gitlab-ci.yml found in the project` → `Add a CI configuration file to trigger pipelines` |
| Insufficient permissions| `✗ Error: Insufficient permissions to create merge request` → `Ensure your token has 'api' scope` |
| Push failed             | `✗ Error: Failed to push branch to remote` → `[git error details]`      |
