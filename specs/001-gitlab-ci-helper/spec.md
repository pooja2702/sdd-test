# Feature Specification: GitLab CI/CD Helper

**Feature Branch**: `001-gitlab-ci-helper`  
**Created**: 2026-03-03  
**Status**: Draft  
**Input**: User description: "Build a script that can work as a ci-cd-helper for any gitlab project. The script should be able to: 1) Create a merge request if one doesn't exist for a given unprotected branch, 2) Fetch the most recent pipeline for a given branch and determine if there are errors, piping them to standard out."

## Clarifications

### Session 2026-03-03

- Q: Should the script require the developer to specify the branch name? → A: No. The script automatically detects the current git branch; no branch parameter is needed.
- Q: Should the script allow creating merge requests for the master/main branch or tags? → A: Never. The script must refuse to create merge requests when on the master/main branch or on a detached HEAD at a tag.
- Q: What format should pipeline error output use? → A: Human-readable plain text, formatted for terminal display.
- Q: What should happen if the current branch does not exist on the remote? → A: The script should prompt the developer to push. If they confirm, the script commits local changes, pushes the branch to the remote, and creates the merge request.
- Q: What should happen if the current branch has no pipelines? → A: The script should automatically trigger a new pipeline for the branch.
- Q: What should happen if the current branch doesn't exist on the remote when checking pipelines? → A: The script should push the branch to the remote first, then trigger a pipeline.
- Q: What if the project has no .gitlab-ci.yml file? → A: The script should detect this and output a clear error explaining that no CI configuration was found and a pipeline cannot be triggered.
- Q: What should happen when a pipeline is currently running? → A: The script should poll at regular intervals until the pipeline reaches a terminal state (passed, failed, or canceled), then report the final result.
- Q: How frequently should the script poll for pipeline status? → A: Every 15 seconds.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Create Merge Request for Current Branch (Priority: P1)

A developer working on a feature branch wants to quickly create a merge request from the command line without switching to the GitLab web interface. They run the helper script from within their repository. The script automatically detects the current git branch, checks whether an open merge request already exists for it, and if none exists, creates one targeting the project's default branch and confirms success. If the current branch does not yet exist on the remote, the script prompts the developer asking if they would like to push it; if they confirm, the script commits any uncommitted changes, pushes the branch to the remote, and then creates the merge request. If a merge request already exists, the script informs the developer and provides a link to the existing merge request. The script refuses to operate when the current branch is master/main or when the HEAD is a tag.

**Why this priority**: Creating merge requests is the most fundamental CI/CD workflow action. Without this capability, the tool provides no value. This is the core feature that enables developers to stay in their terminal and avoid context-switching.

**Independent Test**: Can be fully tested by running the script while checked out on a feature branch and verifying that a merge request is created in GitLab, or that an existing one is reported back with its URL.

**Acceptance Scenarios**:

1. **Given** a developer checked out on an unprotected feature branch with no open merge request, **When** they run the script, **Then** a new merge request is created targeting the project's default branch, and the script outputs the merge request URL.
2. **Given** a developer checked out on an unprotected branch that already has an open merge request, **When** they run the script, **Then** the script does not create a duplicate and instead outputs the existing merge request URL.
3. **Given** the current branch does not exist in the remote repository, **When** the user runs the script, **Then** the script prompts the developer asking if they would like to push the branch to the remote.
4. **Given** the current branch does not exist on the remote and the developer confirms the push prompt, **When** the script proceeds, **Then** it commits any uncommitted local changes, pushes the branch to the remote, and creates a merge request targeting the default branch, outputting the merge request URL.
5. **Given** the current branch does not exist on the remote and the developer declines the push prompt, **When** the script proceeds, **Then** it exits gracefully with a message indicating no action was taken.
6. **Given** the developer is on the `main` or `master` branch, **When** they run the script, **Then** the script outputs a clear error message refusing to create a merge request from the default/protected branch.
7. **Given** the developer's HEAD is a detached HEAD at a tag (not on a branch), **When** they run the script, **Then** the script outputs a clear error message indicating that merge requests cannot be created for tags.

---

### User Story 2 - Fetch Pipeline Status and Surface Errors (Priority: P1)

A developer wants to check the CI/CD pipeline status of their current branch without navigating to the GitLab web interface. They run the helper script from within their repository. The script automatically detects the current git branch and retrieves the latest pipeline for it. If a pipeline exists and has completed, the script reports the final status; if there are errors (failed jobs), it pipes the relevant error details—including job names, stages, and failure reasons—to standard output. If the pipeline is still running, the script polls at regular intervals, showing progress updates in the terminal, until the pipeline reaches a terminal state (passed, failed, or canceled) and then reports the final result. If no pipeline exists for the branch, the script automatically triggers a new pipeline and polls until completion. If the branch does not yet exist on the remote, the script first pushes the branch to the remote and then triggers a pipeline. If the project has no CI configuration file (.gitlab-ci.yml), the script outputs a clear error explaining that a pipeline cannot be triggered.

**Why this priority**: Pipeline visibility is equally critical to merge request creation. Developers need immediate feedback on whether their code changes pass CI/CD checks. This is a co-equal P1 because the two features together form the minimum viable product.

**Independent Test**: Can be fully tested by checking out a branch with a known failed pipeline, running the script, and verifying that error details (job name, stage, failure reason) are printed to standard output.

**Acceptance Scenarios**:

1. **Given** the current branch has a recently completed pipeline with failed jobs, **When** the developer runs the script, **Then** the script outputs the pipeline status as "failed" and lists each failed job with its name, stage, and failure reason to standard output.
2. **Given** the current branch has a recently completed pipeline where all jobs succeeded, **When** the developer runs the script, **Then** the script outputs the pipeline status as "passed" with a success confirmation message.
3. **Given** the current branch has a pipeline currently running, **When** the developer runs the script, **Then** the script polls at regular intervals, displaying progress updates in the terminal, until the pipeline reaches a terminal state (passed, failed, or canceled), and then reports the final result with error details if applicable.
4. **Given** the current branch has no pipelines, **When** the developer runs the script, **Then** the script automatically triggers a new pipeline for the branch and polls until the pipeline completes, reporting the final result.
5. **Given** the current branch does not exist in the remote repository, **When** the developer runs the pipeline status command, **Then** the script pushes the branch to the remote, triggers a new pipeline, and polls until the pipeline completes, reporting the final result.
6. **Given** the project has no `.gitlab-ci.yml` file, **When** the developer runs the pipeline status command, **Then** the script outputs a clear error message indicating that no CI configuration was found and a pipeline cannot be triggered.
7. **Given** a pipeline is being polled and the developer presses Ctrl+C, **When** the script receives the interrupt signal, **Then** it stops polling gracefully and outputs the last known pipeline status.

---

### User Story 3 - Combined Workflow: Create MR and Check Pipeline (Priority: P2)

A developer wants a streamlined workflow where they can create a merge request for their current branch and immediately see the pipeline status in a single command. The script automatically detects the current branch, performs both actions sequentially: first ensuring a merge request exists (creating one if needed), then fetching and displaying the most recent pipeline status.

**Why this priority**: While each capability is independently valuable, combining them into a single operation reduces friction further. This is a natural extension once both P1 features are built.

**Independent Test**: Can be tested by running the combined command while on a feature branch without an existing MR and with a recent pipeline, and verifying both the MR creation confirmation and pipeline status output appear.

**Acceptance Scenarios**:

1. **Given** the developer is on a feature branch with no existing merge request and a completed pipeline, **When** they run the combined command, **Then** the script creates the merge request, outputs its URL, and then outputs the pipeline status and any errors.
2. **Given** the developer is on a branch with an existing merge request and a failed pipeline, **When** they run the combined command, **Then** the script reports the existing merge request URL and outputs the failed pipeline details.

---

### Edge Cases

- What happens when the user's authentication token is invalid or expired? The script should output a clear authentication error and suggest how to provide valid credentials.
- What happens when the GitLab server is unreachable (network error)? The script should output a connection error with the server URL it attempted to reach.
- What happens when the user does not have sufficient permissions to create a merge request in the project? The script should output a permissions error indicating the required access level.
- What happens when multiple pipelines exist for the same branch and the most recent one is still pending? The script should report the most recent pipeline's status regardless of state.
- What happens when a pipeline has jobs with "allowed to fail" status? The script should distinguish between hard failures and allowed failures in its output.
- What happens when the merge request target branch (default branch) does not exist or has been renamed? The script should output an error indicating the target branch could not be determined.
- What happens when the developer confirms a push but there are no local changes to commit? The script should push the branch as-is (with existing commits) without attempting an empty commit.
- What happens when the push to remote fails (e.g., permission denied)? The script should output a clear error message with the failure reason and not attempt to create a merge request.
- What happens when the developer is on a detached HEAD (e.g., checked out a tag or specific commit)? The script should detect this and output an error message refusing to proceed.
- What happens when the developer is on the master/main branch and tries to run the MR creation command? The script should refuse with a clear message explaining that merge requests cannot be created from the default branch.
- What happens when the project has no `.gitlab-ci.yml` file and the developer tries to check pipeline status? The script should detect the missing CI configuration and output a clear error explaining that no pipeline can be triggered without a CI configuration file.
- What happens when a pipeline is triggered but fails to start (e.g., invalid CI config syntax)? The script should report the pipeline creation failure with the error details returned by GitLab.
- What happens when the developer interrupts polling (Ctrl+C)? The script should stop polling gracefully and output the last known pipeline status before exiting.
- What happens when a pipeline is stuck in a running state for an extremely long time? The script should continue polling indefinitely until a terminal state is reached or the developer interrupts; there is no automatic timeout.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The script MUST accept a GitLab project identifier (project path or ID) as an input parameter; the branch is automatically detected from the current git checkout.
- **FR-002**: The script MUST authenticate with the GitLab instance using a personal access token provided via environment variable or configuration.
- **FR-003**: The script MUST automatically detect the current git branch and use it as the source branch for all operations.
- **FR-004**: The script MUST check whether an open merge request already exists for the current branch before attempting to create one.
- **FR-005**: The script MUST create a new merge request targeting the project's default branch when no open merge request exists for the current unprotected branch.
- **FR-006**: The script MUST output the merge request URL upon successful creation or when an existing merge request is found.
- **FR-007**: The script MUST refuse to create a merge request when the current branch is `master` or `main` (the default/protected branch), outputting a clear error message.
- **FR-008**: The script MUST refuse to create a merge request when the HEAD is detached at a tag, outputting a clear error message.
- **FR-009**: The script MUST fetch the most recent pipeline for the current branch.
- **FR-010**: The script MUST determine the overall pipeline status (passed, failed, running, pending, canceled, or other states).
- **FR-011**: The script MUST output detailed error information for each failed job in a pipeline in human-readable plain text format, including: job name, stage name, and failure reason.
- **FR-012**: The script MUST pipe all pipeline error output as human-readable plain text to standard output so it can be consumed by other tools or redirected.
- **FR-013**: The script MUST provide clear, actionable error messages for all failure conditions (invalid branch, authentication failure, network errors, insufficient permissions, detached HEAD, default branch).
- **FR-014**: The script MUST work with any GitLab project the user has access to, not just a specific hardcoded project.
- **FR-015**: The script MUST support specifying the GitLab instance URL to work with both GitLab.com and self-hosted GitLab instances.
- **FR-016**: The script MUST distinguish between jobs that failed and jobs marked as "allowed to fail" in pipeline error reporting.
- **FR-017**: When the current branch does not exist on the remote, the script MUST prompt the developer interactively, asking if they would like to push the branch to the remote.
- **FR-018**: If the developer confirms the push prompt, the script MUST commit any uncommitted local changes, push the branch to the remote, and then proceed to create the merge request.
- **FR-019**: When the current branch has no pipelines, the script MUST automatically trigger a new pipeline for the branch.
- **FR-020**: When the current branch does not exist on the remote and the developer requests pipeline status, the script MUST push the branch to the remote first and then trigger a new pipeline.
- **FR-021**: The script MUST detect when the project has no CI configuration file (.gitlab-ci.yml) and output a clear error message indicating that a pipeline cannot be triggered.
- **FR-022**: When a pipeline is in a non-terminal state (running, pending, or created), the script MUST poll every 15 seconds until the pipeline reaches a terminal state (passed, failed, or canceled), displaying progress updates to the developer during polling.
- **FR-023**: The script MUST handle developer interruption (e.g., Ctrl+C) during polling gracefully, stopping the poll loop and outputting the last known pipeline status before exiting.

### Key Entities

- **Project**: A GitLab project identified by path or numeric ID; has a default branch and protected branch rules.
- **Branch**: A Git branch within a project; may be protected or unprotected; serves as the source for merge requests and pipelines.
- **Merge Request**: A request to merge changes from a source branch into a target branch; has a status (open, merged, closed) and a URL.
- **Pipeline**: A CI/CD execution triggered for a specific branch/commit; has an overall status and contains one or more jobs.
- **Job**: An individual task within a pipeline stage; has a name, stage, status, and failure reason if failed; may be marked as "allowed to fail."

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can create a merge request for any accessible branch within 10 seconds of running the command.
- **SC-002**: Users receive the final pipeline result (passed or failed with error details) after the script polls through to completion, without needing to manually re-run the command.
- **SC-003**: 100% of failed pipeline jobs are reported with their name, stage, and failure reason in the output.
- **SC-004**: The script correctly detects existing merge requests and avoids creating duplicates in 100% of cases.
- **SC-005**: All error conditions (authentication, network, permissions, invalid branch) produce user-readable messages that include the specific cause and a suggested resolution.
- **SC-006**: The script works across any GitLab project the user has access to, regardless of the GitLab instance (cloud or self-hosted).

## Assumptions

- Users have a valid GitLab personal access token with sufficient scopes (read/write access to merge requests and pipelines).
- The GitLab instance exposes a standard interface for project, merge request, and pipeline operations.
- The project's default branch is used as the merge request target unless otherwise specified.
- The script is intended to be run from within a git repository on a developer's local machine, where it can detect the current branch.
- "Unprotected branch" refers to any branch not marked as protected in GitLab's branch protection settings.
- The script never requires a branch name parameter; it always operates on the currently checked out branch.
- The `master` and `main` branches are always treated as default/protected branches from which merge requests cannot be created.
- Tags (detached HEAD at a tag) are not valid sources for merge request creation.
- Pipeline error output is human-readable plain text (formatted for terminal display), not machine-readable JSON. It includes job name, stage, and failure reason for each failed job.
- When the current branch is not on the remote, the script interactively prompts the developer rather than silently failing; the developer must explicitly confirm before any push or commit occurs.
