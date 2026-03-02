# Data Model: GitLab CI/CD Helper

**Branch**: `001-gitlab-ci-helper` | **Date**: 2026-03-03

> This is a stateless CLI tool with no persistent storage. The data model describes the domain objects passed through the application at runtime, sourced from Git and the GitLab API.

## Entities

### GitLabConfig

Holds all configuration needed to connect to GitLab.

| Field        | Type   | Source                  | Required | Notes                           |
|--------------|--------|-------------------------|----------|---------------------------------|
| token        | str    | `GITLAB_TOKEN` env var  | Yes      | Never logged or printed         |
| url          | str    | `GITLAB_URL` env var    | No       | Default: `https://gitlab.com`   |
| project_path | str    | `GITLAB_PROJECT` or git remote | Yes | Auto-detected if not set |

### BranchInfo

Represents the current state of the local and remote branch.

| Field              | Type   | Source                  | Notes                                    |
|--------------------|--------|-------------------------|------------------------------------------|
| name               | str    | `git branch --show-current` | Empty string if detached HEAD         |
| is_detached        | bool   | derived                 | True if HEAD is not on a branch          |
| is_default         | bool   | derived                 | True if name is `main` or `master`       |
| exists_on_remote   | bool   | GitLab API              | True if branch exists on the remote      |
| is_protected       | bool   | GitLab API              | True if branch is protected in GitLab    |
| has_uncommitted    | bool   | `git status --porcelain`| True if working tree has changes         |

### MergeRequestInfo

Represents a merge request (existing or newly created).

| Field       | Type   | Source     | Notes                              |
|-------------|--------|------------|------------------------------------|
| iid         | int    | GitLab API | Merge request internal ID          |
| title       | str    | GitLab API | MR title                           |
| url         | str    | GitLab API | Web URL of the merge request       |
| state       | str    | GitLab API | `opened`, `merged`, `closed`       |
| source      | str    | GitLab API | Source branch name                 |
| target      | str    | GitLab API | Target branch name                 |
| created_new | bool   | derived    | True if MR was just created        |

### PipelineInfo

Represents a CI/CD pipeline.

| Field       | Type   | Source     | Notes                                        |
|-------------|--------|------------|----------------------------------------------|
| id          | int    | GitLab API | Pipeline ID                                  |
| status      | str    | GitLab API | `created`, `pending`, `running`, `success`, `failed`, `canceled`, `skipped`, `manual` |
| web_url     | str    | GitLab API | Web URL of the pipeline                      |
| ref         | str    | GitLab API | Branch ref the pipeline runs against         |
| is_terminal | bool   | derived    | True if status is `success`, `failed`, or `canceled` |
| triggered   | bool   | derived    | True if pipeline was just triggered by this script |

### JobInfo

Represents a job within a pipeline.

| Field          | Type   | Source     | Notes                                      |
|----------------|--------|------------|--------------------------------------------|
| id             | int    | GitLab API | Job ID                                     |
| name           | str    | GitLab API | Job name                                   |
| stage          | str    | GitLab API | Stage name (e.g., `build`, `test`, `deploy`) |
| status         | str    | GitLab API | `success`, `failed`, `running`, `pending`, etc. |
| failure_reason | str    | GitLab API | Reason for failure (if failed)             |
| allow_failure  | bool   | GitLab API | True if job is marked as "allowed to fail" |
| web_url        | str    | GitLab API | Web URL of the job                         |

## Relationships

```text
GitLabConfig ──uses──> GitLab API
     │
     ├── BranchInfo (1:1 per invocation)
     │       │
     │       ├── MergeRequestInfo (0..1 open MR per branch)
     │       │
     │       └── PipelineInfo (0..* pipelines per branch, script uses most recent)
     │               │
     │               └── JobInfo (1..* jobs per pipeline)
```

## State Transitions

### Pipeline Lifecycle (relevant statuses)

```text
created → pending → running → success
                            → failed
                            → canceled
```

The script polls through `created`, `pending`, and `running` states until a terminal state (`success`, `failed`, `canceled`) is reached.

### MR Creation Flow

```text
[Branch detected] → [Check remote exists?]
    │                       │
    │ exists                │ not exists
    │                       ▼
    │               [Prompt: push?]
    │                  │       │
    │                yes│      │no
    │                  ▼       ▼
    │            [Commit+Push] [Exit]
    │                  │
    ▼                  ▼
[Check open MR?] ←────┘
    │       │
    │exists │ not exists
    ▼       ▼
[Report] [Create MR]
```
