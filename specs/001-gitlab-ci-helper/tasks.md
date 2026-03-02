# Tasks: GitLab CI/CD Helper

**Input**: Design documents from `/specs/001-gitlab-ci-helper/`
**Prerequisites**: plan.md (required), spec.md (required), research.md, data-model.md, contracts/cli-contract.md, quickstart.md

**Tests**: Included per constitution principle II (Testing). Unit tests target 80%+ overall coverage, 90%+ for critical modules. Tests are written FIRST per story, then implementation follows.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization, packaging, and basic structure

- [ ] T001 Create project directory structure with `src/__init__.py`, `tests/__init__.py`, and all empty module files per plan.md project structure
- [ ] T002 Create `pyproject.toml` with project metadata, entry point `cicd` → `src.cli:main`, and pinned dependencies: `python-gitlab>=4.0,<5.0`, `click>=8.0,<9.0`, `rich>=13.0,<14.0`
- [ ] T003 [P] Create `requirements-dev.txt` with `pytest>=7.0`, `pytest-mock>=3.0`, `pytest-cov>=4.0`
- [ ] T004 [P] Create `.gitignore` with Python defaults (`.venv/`, `__pycache__/`, `*.pyc`, `.pytest_cache/`, `dist/`, `*.egg-info/`)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T005 Implement custom exception hierarchy in `src/exceptions.py`: `CicdError` (base), `AuthenticationError`, `NetworkError`, `PermissionError`, `BranchError`, `PipelineError`, `GitError`. Each exception MUST include a `user_message` property with an actionable resolution hint per constitution principle III (Exception Handling). Include docstrings for every class.
- [ ] T006 [P] Implement `src/config.py`: `GitLabConfig` dataclass with fields `token` (str), `url` (str, default `https://gitlab.com`), `project_path` (str, optional). Factory method `from_environment()` reads `GITLAB_TOKEN`, `GITLAB_URL`, `GITLAB_PROJECT` from env vars with defaults. Raise `AuthenticationError` if `GITLAB_TOKEN` is missing. The token value MUST never appear in any string representation or log output. Include type hints and docstrings per constitution.
- [ ] T007 [P] Implement `src/output.py`: Output formatter using `rich`. Functions: `print_success(message)`, `print_error(error_message, resolution)`, `print_info(message)`, `print_warning(message)`, `print_branch(branch_name)`, `print_mr_url(url, is_new)`, `print_pipeline_status(pipeline_info, jobs)`, `print_polling_status(pipeline_info, jobs, interval)`. All functions MUST use rich formatting (✓, ✗, ⟳, ◌, ~) per the CLI contract examples. Include type hints and docstrings.
- [ ] T008 Implement `src/gitlab_client.py`: Wrapper around `python-gitlab`. Class `GitLabClient` with `__init__(config: GitLabConfig)` that creates an authenticated `gitlab.Gitlab` instance. Methods: `get_project()` returns the project object; handles `GitlabAuthenticationError` → `AuthenticationError`, `GitlabGetError` → `NetworkError`. All API errors MUST be sanitized to remove any token material before re-raising. Include type hints and docstrings.
- [ ] T009 Implement `src/git_utils.py` (core functions only, shared across stories): Functions: `get_current_branch()` → str (raises `BranchError` if detached HEAD), `is_detached_head()` → bool, `is_default_branch(branch_name)` → bool (checks against `main`/`master`), `is_on_tag()` → bool, `get_remote_url(remote_name)` → str, `parse_project_path(remote_url)` → str (extracts `namespace/project` from SSH or HTTPS URLs), `has_uncommitted_changes()` → bool, `branch_exists_on_remote(branch_name)` → bool (via `git ls-remote`). All functions use `subprocess.run` with `check=True` and handle `subprocess.CalledProcessError` → `GitError`. Include type hints and docstrings.
- [ ] T010 Implement `src/cli.py` (skeleton only): Create click group `main` with global options `--gitlab-url`, `--token`, `--project`, `--version`. Wire env var fallbacks (`GITLAB_URL`, `GITLAB_TOKEN`, `GITLAB_PROJECT`). Add empty `mr`, `pipeline`, and `run` subcommands as placeholders. Include module docstring.
- [ ] T011 Implement shared test fixtures in `tests/conftest.py`: Fixtures for `mock_gitlab_config` (GitLabConfig with test values, token=`test-token-REDACTED`), `mock_gitlab_client` (mocked GitLabClient), `mock_subprocess` (patched `subprocess.run`), `mock_git_repo` (simulated git repo state). All fixtures MUST use `pytest-mock` for patching. Include docstrings.

### Tests for Foundational Phase ⚠️

> **NOTE: Write these tests FIRST, ensure they FAIL before implementation**

- [ ] T012 [P] Unit tests in `tests/test_config.py`: Test `from_environment()` with valid token, missing token raises `AuthenticationError`, default URL fallback, custom URL, project path from env, token never appears in `str()` or `repr()` of config object. Minimum 6 test cases.
- [ ] T013 [P] Unit tests in `tests/test_output.py`: Test each output function produces expected rich markup. Test `print_error` includes resolution text. Test `print_pipeline_status` correctly formats passed, failed, running states with correct symbols. Test `print_mr_url` differentiates new vs existing MRs. Minimum 8 test cases.
- [ ] T014 [P] Unit tests in `tests/test_gitlab_client.py`: Test successful project retrieval, authentication failure maps to `AuthenticationError`, network error maps to `NetworkError`, error messages are sanitized (no token in exception text). Minimum 5 test cases.
- [ ] T015 [P] Unit tests in `tests/test_git_utils.py`: Test `get_current_branch()` returns correct branch, detached HEAD raises `BranchError`, `is_default_branch` for main/master/feature, `is_on_tag` detection, `parse_project_path` for SSH and HTTPS URLs, `has_uncommitted_changes` true/false, `branch_exists_on_remote` true/false. Minimum 10 test cases.

**Checkpoint**: Foundation ready — user story implementation can now begin

---

## Phase 3: User Story 1 — Create Merge Request for Current Branch (Priority: P1) 🎯 MVP

**Goal**: Developer can create a merge request for the current branch from the CLI, with automatic branch push if not on remote

**Independent Test**: Run `cicd mr` while checked out on a feature branch and verify MR is created in GitLab or existing MR URL is reported

### Tests for User Story 1 ⚠️

> **NOTE: Write these tests FIRST, ensure they FAIL before implementation**

- [ ] T016 [P] [US1] Unit tests in `tests/test_merge_request.py`: Test `find_existing_merge_request()` returns MR when one exists, returns `None` when none exists. Test `create_merge_request()` creates MR with correct source/target branches. Test `create_merge_request()` raises `PermissionError` when user lacks access. Test MR info object has correct `url`, `iid`, `created_new` fields. Minimum 6 test cases.
- [ ] T017 [P] [US1] Unit tests in `tests/test_cli.py` (MR command): Test `cicd mr` on feature branch creates MR (exit 0). Test `cicd mr` on branch with existing MR reports URL (exit 0). Test `cicd mr` on main/master exits with error (exit 1). Test `cicd mr` on detached HEAD exits with error (exit 1). Test `cicd mr` on branch not on remote prompts user. Test user confirms push → commits, pushes, creates MR (exit 0). Test user declines push → exits gracefully (exit 2). Minimum 7 test cases using click `CliRunner`.

### Implementation for User Story 1

- [ ] T018 [US1] Implement `src/git_utils.py` (push functions): Add functions `commit_all_changes(message)` (stages all changes, commits with message; skips if nothing to commit), `push_branch_to_remote(branch_name, remote_name)` (pushes branch; raises `GitError` on failure). Both functions use `subprocess.run` with error handling. Include type hints and docstrings.
- [ ] T019 [US1] Implement `src/merge_request.py`: Functions: `find_existing_merge_request(client, project, branch_name)` → `MergeRequestInfo | None` (queries open MRs by source branch), `create_merge_request(client, project, branch_name, title, target_branch)` → `MergeRequestInfo` (creates MR, returns info with `created_new=True`), `get_default_branch(project)` → str (fetches project default branch). Handle `GitlabCreateError` → `PermissionError`. Include type hints and docstrings.
- [ ] T020 [US1] Implement `cicd mr` command in `src/cli.py`: Wire the full MR creation flow per CLI contract behavior: (1) detect branch with guards for detached HEAD, default branch, and tags, (2) check remote existence → prompt to push if missing, (3) check existing MR → report if found, (4) create MR → print URL. Add `--title` and `--target` options. Set correct exit codes (0, 1, 2). All errors MUST produce user-friendly messages via `output.py`.
- [ ] T021 [US1] Add error handling and edge cases to `cicd mr`: Handle token missing/invalid → `AuthenticationError` with resolution. Handle network unreachable → `NetworkError` with server URL. Handle permission denied → `PermissionError` with scope hint. Handle push failure → `GitError` with details. Handle no changes to commit (push as-is). All errors use the `✗ Error: ... → ...` format from the CLI contract.

**Checkpoint**: At this point, User Story 1 should be fully functional — `cicd mr` works end-to-end

---

## Phase 4: User Story 2 — Fetch Pipeline Status and Surface Errors (Priority: P1) 🎯 MVP

**Goal**: Developer can check pipeline status, trigger pipelines, and poll until completion, all from the CLI

**Independent Test**: Run `cicd pipeline` on a branch with a known failed pipeline and verify error details are printed to stdout

### Tests for User Story 2 ⚠️

> **NOTE: Write these tests FIRST, ensure they FAIL before implementation**

- [ ] T022 [P] [US2] Unit tests in `tests/test_pipeline.py`: Test `get_latest_pipeline()` returns most recent pipeline. Test `get_latest_pipeline()` returns `None` when no pipelines exist. Test `trigger_pipeline()` creates new pipeline and returns info with `triggered=True`. Test `get_pipeline_jobs()` returns list of `JobInfo` objects with correct fields. Test `is_terminal_status()` returns True for `success`/`failed`/`canceled`, False for `running`/`pending`/`created`. Test `check_ci_config_exists()` returns True/False correctly. Test `trigger_pipeline()` raises `PipelineError` when no CI config. Minimum 8 test cases.
- [ ] T023 [P] [US2] Unit tests in `tests/test_pipeline.py` (polling): Test `poll_pipeline()` returns immediately on terminal state. Test `poll_pipeline()` calls callback and sleeps on non-terminal state, then returns on terminal. Test `poll_pipeline()` handles `KeyboardInterrupt` gracefully, returning last known status. Test polling distinguishes between hard failures and `allow_failure` jobs. Minimum 4 test cases.
- [ ] T024 [P] [US2] Unit tests in `tests/test_cli.py` (pipeline command): Test `cicd pipeline` with passed pipeline (exit 0). Test `cicd pipeline` with failed pipeline prints job details (exit 1). Test `cicd pipeline` with no pipeline triggers new one (exit 0/1). Test `cicd pipeline` with branch not on remote pushes first. Test `cicd pipeline` with no CI config outputs error (exit 3). Test `cicd pipeline --no-poll` reports once without polling. Test `cicd pipeline` with running pipeline polls until done. Minimum 7 test cases using click `CliRunner`.

### Implementation for User Story 2

- [ ] T025 [US2] Implement `src/pipeline.py` (core functions): Functions: `get_latest_pipeline(client, project, branch_name)` → `PipelineInfo | None`, `trigger_pipeline(client, project, branch_name)` → `PipelineInfo`, `get_pipeline_jobs(client, project, pipeline_id)` → `list[JobInfo]`, `is_terminal_status(status)` → bool, `check_ci_config_exists(client, project)` → bool. Handle API errors and map to custom exceptions. Include type hints and docstrings.
- [ ] T026 [US2] Implement `src/pipeline.py` (polling logic): Function `poll_pipeline(client, project, pipeline_id, interval_seconds=15, on_update=None)` → `tuple[PipelineInfo, list[JobInfo]]`. Polls every 15 seconds, calls `on_update` callback with current status and jobs at each interval. Returns final state on terminal status. Catches `KeyboardInterrupt`, returns last known state. Include type hints and docstrings.
- [ ] T027 [US2] Implement `cicd pipeline` command in `src/cli.py`: Wire the full pipeline flow per CLI contract behavior: (1) detect branch, (2) check remote → push if missing, (3) fetch latest pipeline → trigger if none, (4) if terminal → report, (5) if running → poll with rich progress display, (6) report final result with all job details, distinguishing hard failures from `allow_failure`. Add `--no-poll` option. Set correct exit codes (0, 1, 2, 3). Handle Ctrl+C gracefully.
- [ ] T028 [US2] Implement pipeline output formatting in `src/output.py`: Add functions `print_job_list(jobs)` (formats each job with ✓/✗/⟳/◌/~ symbols per status and allow_failure), `print_polling_header(pipeline_id, interval)` (shows "polling every 15s, Ctrl+C to stop"), `clear_and_reprint_status(pipeline_info, jobs)` (refreshes the terminal display during polling). All output MUST match the CLI contract examples exactly.
- [ ] T029 [US2] Add error handling and edge cases to `cicd pipeline`: Handle missing CI config → `PipelineError` with resolution. Handle pipeline trigger failure (invalid YAML) → display GitLab error details. Handle branch not on remote → auto-push (no prompt for pipeline command, unlike MR command). Handle allowed-to-fail jobs displayed with `~` prefix. All errors use the `✗ Error: ... → ...` format.

**Checkpoint**: At this point, User Stories 1 AND 2 should both work independently — `cicd mr` and `cicd pipeline` are complete

---

## Phase 5: User Story 3 — Combined Workflow: Create MR and Check Pipeline (Priority: P2)

**Goal**: Developer can run a single command to create MR + check pipeline status

**Independent Test**: Run `cicd run` on a feature branch and verify both MR creation and pipeline status output appear

### Tests for User Story 3 ⚠️

- [ ] T030 [P] [US3] Unit tests in `tests/test_cli.py` (run command): Test `cicd run` creates MR then shows pipeline status. Test `cicd run` with existing MR reports URL then shows pipeline. Test `cicd run` with MR error stops before pipeline check (exit 1). Test exit code follows pipeline result. Minimum 4 test cases using click `CliRunner`.

### Implementation for User Story 3

- [ ] T031 [US3] Implement `cicd run` command in `src/cli.py`: Compose the MR and pipeline logic sequentially: (1) execute MR creation flow (from US1), (2) if MR step succeeds, execute pipeline flow (from US2). Use shared branch detection to avoid redundant git calls. Set exit code based on pipeline result, or 1 if MR fails. Include docstring.

**Checkpoint**: All user stories should now be independently functional — `cicd mr`, `cicd pipeline`, and `cicd run` all work

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

- [ ] T032 [P] Create `README.md` at project root with project description, installation instructions, usage examples (from quickstart.md), configuration reference, exit codes table, and contributing section
- [ ] T033 [P] Add `--verbose` flag to CLI global options in `src/cli.py` that enables stack trace output for errors (per constitution principle III, user-facing messages omit stack traces unless verbose is active)
- [ ] T034 Run full test suite with `pytest --cov=src --cov-report=term-missing` and verify minimum 80% overall coverage, 90%+ for `src/gitlab_client.py`, `src/pipeline.py`, and `src/merge_request.py`. Add any missing test cases to reach targets.
- [ ] T035 [P] Validate quickstart.md by performing a manual end-to-end walkthrough of all three commands (`cicd mr`, `cicd pipeline`, `cicd run`) against a real GitLab project
- [ ] T036 Code review pass: Verify all functions have docstrings, all variables have meaningful names, no bare `except:` blocks, no sensitive data in any output, PEP 8 compliance, type hints on all public functions (per constitution principles I-IV)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion — BLOCKS all user stories
- **User Story 1 (Phase 3)**: Depends on Foundational phase completion
- **User Story 2 (Phase 4)**: Depends on Foundational phase completion; can run in parallel with US1
- **User Story 3 (Phase 5)**: Depends on BOTH User Story 1 AND User Story 2 being complete
- **Polish (Phase 6)**: Depends on all user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) — No dependencies on other stories
- **User Story 2 (P1)**: Can start after Foundational (Phase 2) — No dependencies on other stories
- **User Story 3 (P2)**: Depends on US1 + US2 — Composes both into a single command

### Within Each User Story

- Tests MUST be written and FAIL before implementation
- Core logic before CLI wiring
- CLI wiring before error handling polish
- Story complete before moving to next priority

### Parallel Opportunities

- T003, T004 can run in parallel (Setup)
- T006, T007 can run in parallel (Foundational — different files)
- T012, T013, T014, T015 can run in parallel (Foundational tests — different files)
- T016, T017 can run in parallel (US1 tests)
- T022, T023, T024 can run in parallel (US2 tests)
- US1 (Phase 3) and US2 (Phase 4) can run in parallel after Phase 2

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL — blocks all stories)
3. Complete Phase 3: User Story 1
4. **STOP and VALIDATE**: Test `cicd mr` independently
5. Deploy/demo if ready

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready
2. Add User Story 1 → Test `cicd mr` independently → MVP!
3. Add User Story 2 → Test `cicd pipeline` independently → Full P1 scope
4. Add User Story 3 → Test `cicd run` → Complete feature
5. Polish → Production-ready

### Parallel Strategy

With two developers:

1. Both complete Setup + Foundational together
2. Once Foundational is done:
   - Developer A: User Story 1 (MR creation)
   - Developer B: User Story 2 (Pipeline status)
3. Once both done: Either developer implements User Story 3 (composition)
4. Both collaborate on Polish phase

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story is independently completable and testable
- Tests MUST fail before implementing (red-green-refactor)
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Constitution compliance: readability, testing, exception handling, security checked at every task
