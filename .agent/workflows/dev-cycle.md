---
description: Orchestrate a complete coding exercise using multiple personas (Dev, QA, Reviewer) and state tags.
---

# Complete Development Cycle Workflow

This workflow orchestrates a full coding cycle as a state machine. The agent tracks state for the overall project and individual tasks via a structured text document located at `./<backlog-item>/tasks.md`.
**Note**: The `./<backlog-item>` directory is a temporary folder used solely to keep track of the dev cycle. It should NOT be staged, and must NOT be cleared until the PR/MR is successfully merged.

When acting in any state, adhere strictly to the **Instructions** and only move to the next state when the **Transitions** conditions are fully met.

## Persona Definitions

To effectively execute this workflow, you must adopt the mindset of the assigned persona for each state:
- **Analyst**: Focuses on parsing input, asking clarifying questions, and strictly codifying requirements.
- **Architect / Lead Dev**: Focuses on high-level system design, defining implementation strategies, and breaking down architecture into logically ordered, independent tasks.
- **Dev**: Focuses on implementing code, autonomously fixing and navigating compilation/runtime issues, and cleanly committing the final changes.
- **QA**: Focuses on rigorous testing, verifying acceptance criteria, guarding against regressions, and returning precise error reports when tests fail.
- **Reviewer**: Operates as a senior, experienced engineer validating code quality, best practices, architectural consistency, and long-term maintainability.

---

## State: `[GATHER_REQUIREMENTS]`
**Persona:** Default / Analyst
**Instructions:**
1. The `<backlog-item>.md` file will be passed as the only argument to trigger the workflow. When the md file is read, initially create the `./<backlog-item>` temporary folder (where `<backlog-item>` is a placeholder filled with the provided markdown filename) and save a copy of the read markdown file inside this folder as `spec.md`. Analyze this `spec.md` file to gain an overview, along with the target codebase and any relevant context.
2. Ask questions to help refine a precise set of requirements for the backlog item. Wait for user answers.
3. Use the **Easy Approach to Requirements Syntax (EARS)** for the requirements and write them to `./<backlog-item>/requirements.md`.
    - **EARS Explanation:** EARS is a mechanism to gently constrain textual requirements. The EARS patterns provide structured guidance that enable authors to write high quality textual requirements.
    - **Generic EARS syntax:** The clauses of a requirement written in EARS always appear in the same order. The basic structure is: `While <optional pre-condition>, when <optional trigger>, the <system name> shall <system response>`
    - **EARS ruleset:** A requirement must have: Zero or many preconditions; Zero or one trigger; One system name; One or many system responses.
**Transitions:**
- If requirements are finalized and written $\rightarrow$ Transition to `[PLANNING]`
- If requirements need clarification $\rightarrow$ Wait for User Input $\rightarrow$ `[GATHER_REQUIREMENTS]`

---

## State: `[PLANNING]`
**Persona:** Architect / Lead Dev
**Instructions:**
1. Draft a comprehensive implementation plan based on the requirements.
2. Detail architectural decisions, components to modify/create, and data flow.
3. Present the plan to the user for explicit approval.
**Transitions:**
- User approves plan $\rightarrow$ Transition to `[TASK_BREAKDOWN]`
- User requests changes $\rightarrow$ `[PLANNING]`

---

## State: `[TASK_BREAKDOWN]`
**Persona:** Architect / Lead Dev
**Instructions:**
1. Break the approved plan into a list of clear, independent, and actionable tasks.
2. Assign a specific priority level to each task.
3. Identify a chronological execution order based on dependencies. Explicitly document which tasks depend on others and which tasks can be executed in parallel.
4. Initialize these prioritized and ordered tasks in a tracking document (`./<backlog-item>/tasks.md`).
5. Label all new tasks with the state tag: `[TODO]`.
**Transitions:**
- Tasks initialized and ordered $\rightarrow$ Transition to `[DEV_PHASE]` following the chronological execution order for `[TODO]` tasks.

---

## State: `[DEV_PHASE]`
**Target Task Tag:** `[TODO]`
**Persona:** Dev
**Instructions:**
1. Before starting a new task, always read `./<backlog-item>/learnings.md` if it exists.
2. Strictly enforce Test-Driven Development (TDD). First, write unit and/or integration tests for the current target task before writing the application code, and run them to ensure they fail appropriately.
3. Implement the minimal code required to complete the current target task and pass the newly written tests.
4. The build and test instructions will live in the project's `AGENTS.md`. Refer to that document to build the project and execute the tests.
5. Build the code to ensure the project compiles successfully without structural errors and that all tests pass.
6. If there are compilation failures or test failures, do NOT pause for user confirmation. Continue to fix errors autonomously until the build passes and tests pass.
7. If anything useful is learned during development that could be helpful for future tasks, append it to `./<backlog-item>/learnings.md` (creating the file if it does not exist).
8. Update the task's state tag in the tracking document from `[TODO]` to `[DEV-COMPLETE]`.
**Transitions:**
- Compilation successful $\rightarrow$ Transition directly to `[QA_PHASE]`
- Compilation fails $\rightarrow$ Fix errors autonomously $\rightarrow$ `[DEV_PHASE]`

---

## State: `[QA_PHASE]`
**Target Task Tag:** `[DEV-COMPLETE]`
**Persona:** QA
**Instructions:**
1. Once the dev agent claims the task is complete, test it. Identify the applicable testing commands based on the modified code and project context.
2. Run the tests locally.
3. Verify that the changes meet the acceptance criteria and ensure there are no regressions.
4. If it does not work, provide detailed errors and feedback to the dev agent until it does pass.
5. If testing passes, update the task's state tag from `[DEV-COMPLETE]` to `[QA-PASSED]`.
**Transitions:**
- Tests pass $\rightarrow$ Transition directly to `[COMMIT_PHASE]`
- Tests fail or do not meet requirements $\rightarrow$ Revert task tag to `[TODO]`, document the failures and feedback $\rightarrow$ Transition autonomously to `[DEV_PHASE]`

---

## State: `[COMMIT_PHASE]`
**Target Task Tag:** `[QA-PASSED]` or `[REVIEW-PASSED]`
**Persona:** Dev
**Instructions:**
1. Update the successfully tested task's status tag to `[READY-FOR-COMMIT]`.
2. Ensure the `./<backlog-item>` folder is explicitly excluded/unstaged from the commit.
3. Formulate a clear, conventional commit message detailing the changes.
4. Autonomously execute the `git commit` command to commit the changes.
5. Clear the context (or reset state).
**Transitions:**
- Commit successful $\rightarrow$ Clear context and then repeat the build step/cycle, transitioning to `[DEV_PHASE]`, until the full plan is implemented. If no `[TODO]` tasks remain, transition to `[REVIEW_PHASE]`.

---

## State: `[REVIEW_PHASE]`
**Target Task Tag:** `[READY-FOR-COMMIT]`
**Persona:** Reviewer
**Instructions:**
1. Check the code against best practices for the programming language of the file.
2. Check the code is clear, easy to read and simple.
3. Check the code is consistent with the majority of the code in the project.
4. Suggest any refactoring opportunities.
5. Verify against the acceptance criteria and surface gaps, if any.
6. Capture all code review comments in `./<backlog-item>/codereview.md`. Document the priority of each comment and explicitly state which comments are blocking and must be addressed before the review can be considered complete.
**Transitions:**
- Code review complete (no blocking comments remain unaddressed) $\rightarrow$ Transition to `[PUSH_PHASE]`
- Blocking comments found $\rightarrow$ **PAUSE AND PROMPT:** Ask for user review of the comments. Once reviewed, revert the relevant task tags to `[TODO]` $\rightarrow$ Transition to `[DEV_PHASE]`

---

## State: `[PUSH_PHASE]`
**Persona:** Dev
**Instructions:**
1. Push the committed code to the remote branch.
2. Create a merge request for the pushed branch.
**Transitions:**
- Push and merge request successful $\rightarrow$ Transition to `[DONE]`

---

## State: `[DONE]`
**Instructions:**
1. Confirm all tasks in the tracking document (`./<backlog-item>/tasks.md`) possess a `[READY-FOR-COMMIT]` or equivalent completed tag.
2. Do NOT clear/delete the `./<backlog-item>` temporary folder until the merge request is successfully merged.
3. Congratulate the team. Operation complete!
