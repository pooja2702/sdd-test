# Quickstart: GitLab CI/CD Helper

**Branch**: `001-gitlab-ci-helper` | **Date**: 2026-03-03

## Prerequisites

- Python 3.10 or later
- Git CLI installed and configured
- A GitLab personal access token with `api` scope

## Installation

```bash
# From the project root
pip install -e .
```

## Configuration

Set your GitLab token as an environment variable:

```bash
# Linux/macOS
export GITLAB_TOKEN="your-token-here"

# Windows (PowerShell)
$env:GITLAB_TOKEN = "your-token-here"
```

Optional configuration:

```bash
# For self-hosted GitLab (default: https://gitlab.com)
export GITLAB_URL="https://gitlab.yourcompany.com"

# Specify project explicitly (default: auto-detected from git remote)
export GITLAB_PROJECT="group/project"
```

## Usage

### Create a Merge Request

```bash
# From any feature branch in your git repo
cicd mr
```

### Check Pipeline Status

```bash
# Fetches the latest pipeline, triggers one if none exists, polls until complete
cicd pipeline
```

### Combined: Create MR + Check Pipeline

```bash
# Creates MR (if needed) then fetches and polls pipeline status
cicd run
```

## Common Workflows

### Start a new feature and push it

```bash
git checkout -b feature/my-new-feature
# ... make changes ...
cicd run
# → Prompts to push if branch isn't on remote
# → Creates merge request
# → Triggers and polls pipeline
```

### Check if your pipeline passed

```bash
cicd pipeline
# → Shows current pipeline status and polls until done
```

## Exit Codes

| Code | Meaning                                     |
|------|---------------------------------------------|
| 0    | Success (MR created/found, pipeline passed) |
| 1    | Failure (error or pipeline failed)          |
| 2    | Pipeline canceled / user declined action    |
| 3    | Configuration error (no CI config, etc.)    |
