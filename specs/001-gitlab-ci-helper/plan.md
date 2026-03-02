# Implementation Plan: GitLab CI/CD Helper

**Branch**: `001-gitlab-ci-helper` | **Date**: 2026-03-03 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-gitlab-ci-helper/spec.md`

## Summary

Build a Python CLI tool that helps developers interact with GitLab CI/CD directly from the terminal. The tool auto-detects the current git branch and provides two core capabilities: (1) creating merge requests (with interactive prompts to push local-only branches), and (2) fetching pipeline status with automatic triggering and polling until completion. All output is human-readable plain text piped to stdout. No sensitive data is ever logged.

## Technical Context

**Language/Version**: Python 3.10+  
**Primary Dependencies**: `python-gitlab` (GitLab API wrapper), `click` (CLI framework), `rich` (terminal formatting & progress)  
**Storage**: N/A (stateless CLI tool; configuration via environment variables)  
**Testing**: `pytest` with `pytest-mock` for mocking GitLab API calls  
**Target Platform**: Cross-platform (Windows, macOS, Linux) — anywhere Python 3.10+ and Git are available  
**Project Type**: CLI tool  
**Performance Goals**: MR creation < 10 seconds; pipeline status retrieval < 10 seconds (excluding poll wait time)  
**Constraints**: No sensitive data logged (tokens must never appear in stdout/stderr); all intermediate logs to stdout; graceful Ctrl+C handling  
**Scale/Scope**: Single-user CLI tool; one project at a time; one branch at a time

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Code Readability | ✅ Pass | Plan specifies modular structure with dedicated files per concern (cli, config, gitlab_client, merge_request, pipeline, git_utils, output). Naming conventions are self-documenting. |
| II. Testing | ✅ Pass | `pytest` with `pytest-mock` specified. Test directory mirrors source structure. All external dependencies (GitLab API, Git CLI) will be mocked. |
| III. Exception Handling | ✅ Pass | Plan includes dedicated error messaging (in cli-contract.md). Custom exceptions and sanitized error output are part of the design. |
| IV. Security & Logging | ✅ Pass | Token is never logged. All API error sanitization is planned. Intermediate logs pipe to stdout per user requirement. |

## Project Structure

### Documentation (this feature)

```text
specs/001-gitlab-ci-helper/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   └── cli-contract.md  # CLI command schema
└── tasks.md             # Phase 2 output (/speckit.tasks command)
```

### Source Code (repository root)

```text
src/
├── __init__.py
├── cli.py              # Click CLI entry point with commands
├── config.py           # Configuration loading (env vars, defaults)
├── gitlab_client.py    # GitLab API wrapper (python-gitlab)
├── merge_request.py    # MR creation & lookup logic
├── pipeline.py         # Pipeline status, triggering, polling logic
├── git_utils.py        # Local git operations (branch detection, push, commit)
└── output.py           # Human-readable output formatting (rich)

tests/
├── __init__.py
├── conftest.py         # Shared fixtures (mock GitLab client, mock git repo)
├── test_cli.py         # CLI integration tests
├── test_config.py      # Config loading tests
├── test_gitlab_client.py
├── test_merge_request.py
├── test_pipeline.py
├── test_git_utils.py
└── test_output.py
```

**Structure Decision**: Single-project CLI layout. The `src/` directory contains all application modules. `tests/` mirrors the source structure. No frontend, no backend split — this is a pure CLI tool.

## Complexity Tracking

No constitution violations to justify.
