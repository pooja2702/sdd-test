<!-- Sync Impact Report
  Version change: 0.0.0 (template) → 1.0.0 (initial ratification)
  Modified principles: N/A (all new)
  Added sections:
    - Core Principles: I. Code Readability, II. Testing, III. Exception Handling, IV. Security
    - Quality Standards (Section 2)
    - Development Workflow (Section 3)
    - Governance
  Removed sections: All template placeholders replaced
  Templates requiring updates:
    - .specify/templates/plan-template.md — ✅ no changes needed (Constitution Check section is generic)
    - .specify/templates/spec-template.md — ✅ no changes needed (requirements section is generic)
    - .specify/templates/tasks-template.md — ✅ no changes needed (test tasks and error handling tasks already present)
  Follow-up TODOs: None
-->

# CI/CD Helper Constitution

## Core Principles

### I. Code Readability

All code MUST be written for human comprehension first, machine execution second.

- Variable, function, and class names MUST be meaningful and self-documenting.
  Names MUST convey purpose and intent (e.g., `pipeline_status` not `ps`,
  `merge_request_url` not `mr_url`).
- Functions MUST have a single clear responsibility. If a function requires
  extensive comments to explain what it does, it MUST be refactored.
- Comments MUST explain *why* something is done, not *what* is done.
  Inline comments are appropriate for non-obvious business logic, edge case
  handling, and architectural decisions. Avoid trivial comments that restate
  the code.
- Module-level docstrings MUST describe the module's purpose and public API.
- Function docstrings MUST describe parameters, return values, and raised
  exceptions for all public functions.

### II. Testing

All features MUST have decent unit test coverage to ensure correctness and
prevent regressions.

- Every public function and class MUST have at least one corresponding unit
  test covering the expected behavior (happy path).
- Error paths and edge cases MUST have dedicated test cases for any behavior
  specified in the feature requirements.
- Tests MUST be independent, deterministic, and fast. External dependencies
  (GitLab API, Git CLI) MUST be mocked in unit tests.
- Test names MUST be descriptive and follow the pattern
  `test_<function>_<scenario>_<expected_outcome>`.
- Test coverage SHOULD target a minimum of 80% line coverage across the
  project. Critical modules (API client, pipeline logic) SHOULD target 90%+.

### III. Exception Handling

All exceptions MUST be handled deliberately and transparently. Silent failures
are prohibited.

- Every external call (API requests, subprocess commands, file I/O) MUST be
  wrapped in appropriate try/except blocks.
- Exceptions MUST be caught at the narrowest scope possible. Bare `except:`
  and `except Exception:` at function level are prohibited unless re-raising.
- All caught exceptions MUST produce a user-facing message that includes:
  (a) what operation failed, (b) a suggested resolution.
- Internal exceptions MUST be logged with full context. User-facing messages
  MUST omit stack traces unless a `--verbose` flag is active.
- Custom exception classes MUST be used to distinguish between recoverable
  errors (e.g., branch not on remote) and fatal errors (e.g., auth failure).

### IV. Security & Logging

No sensitive data MUST ever appear in logs, stdout, stderr, or error messages.

- Authentication tokens, passwords, and API keys MUST never be interpolated
  into log messages, print statements, or error text.
- All intermediate operational logs MUST be piped to stdout for developer
  visibility, but MUST be sanitized to strip any credential material.
- Error messages from external APIs MUST be sanitized before display to
  remove any embedded tokens or credentials from URLs or headers.

## Quality Standards

- Python code MUST follow PEP 8 style guidelines.
- All public modules MUST include type hints for function signatures.
- Functions SHOULD be kept under 30 lines. Functions exceeding 50 lines
  MUST be refactored or justified in a code comment.
- Cyclomatic complexity per function SHOULD remain below 10.
- Dependencies MUST be pinned to specific versions in `requirements.txt`
  or `pyproject.toml` to ensure reproducible builds.

## Development Workflow

- Every feature MUST be developed on a dedicated branch (never directly
  on main/master).
- Code MUST pass all existing tests before being considered complete.
- New code MUST include tests that cover the added behavior.
- Commit messages MUST be descriptive and reference the feature or fix
  being addressed.

## Governance

This constitution defines the non-negotiable quality and coding standards
for the CI/CD Helper project. All code contributions MUST comply with
these principles.

- Amendments to this constitution require documentation of the change,
  the rationale, and a version bump.
- Complexity beyond these standards MUST be explicitly justified in the
  implementation plan or pull request description.
- Version follows semantic versioning: MAJOR for principle
  removals/redefinitions, MINOR for new principles, PATCH for
  clarifications.

**Version**: 1.0.0 | **Ratified**: 2026-03-03 | **Last Amended**: 2026-03-03
