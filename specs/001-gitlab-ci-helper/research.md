# Research: GitLab CI/CD Helper

**Branch**: `001-gitlab-ci-helper` | **Date**: 2026-03-03

## R1: GitLab API Client Library

**Decision**: Use `python-gitlab` (v4.x)

**Rationale**: The official, well-maintained Python wrapper for the GitLab REST API v4. Provides typed access to projects, merge requests, pipelines, jobs, and branches. Handles authentication, pagination, and error mapping. Over 2k GitHub stars, actively maintained by the GitLab community.

**Alternatives considered**:
- **Raw `requests`**: Lower-level, would require manual endpoint construction, pagination, error handling. More work, more bugs.
- **`gidgetlab`**: Async-focused, better for bots/webhooks. Overkill for a synchronous CLI tool.

**Key API endpoints used (via python-gitlab)**:
- `project.mergerequests.list(state='opened', source_branch=X)` — check for existing MR
- `project.mergerequests.create(...)` — create MR
- `project.pipelines.list(ref=branch, order_by='id', sort='desc')` — get latest pipeline
- `project.pipelines.create(ref=branch)` — trigger a new pipeline
- `pipeline.jobs.list()` — get jobs for a pipeline
- `job.trace()` — get job log/trace (for failure reasons)
- `project.branches.get(branch)` — check if branch exists on remote
- `project.protectedbranches.list()` — check if branch is protected

## R2: CLI Framework

**Decision**: Use `click` (v8.x)

**Rationale**: The most popular Python CLI framework. Declarative command/option/argument definitions, automatic help generation, built-in support for environment variables, composable commands via groups. Battle-tested and widely adopted. Aligns with the "developer-friendly interface" requirement.

**Alternatives considered**:
- **`argparse`**: Standard library, but verbose and lacks composability. Help output is less polished.
- **`typer`**: Built on click, adds type hints. Adds a dependency for marginal benefit in a small CLI.
- **`fire`**: Auto-generates CLI from functions. Less control over help text and validation.

## R3: Terminal Output Formatting

**Decision**: Use `rich` (v13.x)

**Rationale**: Provides colored output, tables, progress spinners, and panels out of the box. Perfect for the polling progress updates and structured error reporting. Creates the "intuitive and developer-friendly interface" the user specified. Degrades gracefully in non-TTY environments (piped output).

**Alternatives considered**:
- **Plain print**: Functional but visually poor. Does not meet the "developer-friendly" requirement.
- **`colorama`**: Cross-platform color only. No tables, spinners, or rich formatting.
- **`tabulate`**: Tables only. No colors, spinners, or progress.

## R4: Git Operations (Local)

**Decision**: Use `subprocess` to call Git CLI directly

**Rationale**: The script needs to: detect the current branch, check for detached HEAD, check for tags, commit changes, and push to remote. Using the Git CLI via `subprocess` is the most reliable approach — it uses the developer's own Git installation and configuration (SSH keys, credential helpers). No additional dependency needed.

**Alternatives considered**:
- **`GitPython`**: Wraps Git CLI anyway but adds complexity and edge cases. Its `Repo` object model doesn't add value for our simple use cases.
- **`pygit2`**: Low-level libgit2 bindings. Too complex for branch detection and push.

## R5: Configuration & Authentication

**Decision**: Environment variables with sensible defaults

**Rationale**: CLI tools in the CI/CD space universally use environment variables for configuration. This pattern is familiar to developers and works seamlessly in both local development and CI environments. No config file management needed.

**Configuration variables**:
- `GITLAB_TOKEN` (required): Personal access token. **Never logged.**
- `GITLAB_URL` (optional, default: `https://gitlab.com`): GitLab instance URL
- `GITLAB_PROJECT` (optional): Project path or ID. If not set, auto-detect from git remote URL.

**Security constraint**: The token value must never appear in any output, log, or error message. All token validation errors should say "token is invalid" without revealing the token value.

## R6: Pipeline Polling Strategy

**Decision**: Fixed 15-second interval polling with rich spinner

**Rationale**: Per the clarified spec (FR-022), the polling interval is 15 seconds. The script will display a rich spinner/progress indicator during polling and refresh the job status summary at each poll. Ctrl+C is handled via Python's `signal` module or `KeyboardInterrupt` exception to exit gracefully.

**Implementation approach**:
1. Fetch pipeline status
2. If terminal state → report and exit
3. If non-terminal → display spinner with current job statuses
4. Sleep 15 seconds
5. Repeat from step 1
6. On `KeyboardInterrupt` → print last known status and exit cleanly

## R7: Project Auto-Detection from Git Remote

**Decision**: Parse `git remote get-url origin` to extract GitLab project path

**Rationale**: If `GITLAB_PROJECT` is not set, the script can detect the project by parsing the Git remote URL. Supports both SSH (`git@gitlab.com:group/project.git`) and HTTPS (`https://gitlab.com/group/project.git`) formats. This eliminates the need for the developer to manually specify the project in most cases.

**Parsing patterns**:
- SSH: `git@<host>:<namespace>/<project>.git` → `<namespace>/<project>`
- HTTPS: `https://<host>/<namespace>/<project>.git` → `<namespace>/<project>`

## R8: Sensitive Data Protection

**Decision**: Strict no-log policy for tokens; sanitize all GitLab API errors

**Rationale**: Per user requirement, no sensitive data is ever logged. The `GITLAB_TOKEN` value must be masked in all contexts. GitLab API error responses sometimes include the request URL with tokens — these must be sanitized before output.

**Implementation approach**:
- Never interpolate `GITLAB_TOKEN` into any string that reaches stdout
- Wrap all API calls in a try/catch that sanitizes error messages
- Use `rich` console output which can be configured to mask patterns
