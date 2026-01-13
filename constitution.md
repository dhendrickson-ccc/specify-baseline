<!--
SYNC IMPACT REPORT
==================
Version: 1.2.0 → 1.3.0 (MINOR: Materially expanded Principle III - Agent Must Execute Live Tests)
Modified Principles:
  - Principle III: Live Testing Is Mandatory → Expanded to require agent execution and validation
Added Sections: N/A
Removed Sections: N/A
Templates Status:
  ✅ plan-template.md - Already includes phases structure; enhanced to explicitly require Live Testing/Validation phase
  ✅ tasks-template.md - Already includes task structure by user story; enhanced to require live test execution tasks
  ✅ spec-template.md - Already requires Independent Test scenarios per user story (no updates needed)
  ⚠️ specs/001-drift-detection/plan.md - Should verify Live Testing/Validation phase exists
  ⚠️ specs/001-drift-detection/tasks.md - Should verify live test execution tasks exist
Follow-up TODOs:
  - Implementation workflow should verify that live test tasks are executed with real external systems
  - Plan template should enforce Live Testing/Validation phase presence during constitution check
  - Task template should include examples of live test execution tasks
Propagation Notes:
  - Templates already support live testing workflow - Principle III now enforces agent responsibility
  - No breaking changes - principle clarifies that agent (not just developer) executes and validates
  - Shifts accountability from passive (define tests) to active (execute and validate)
Amendment Rationale:
  - Original Principle III required live tests to be defined but not executed by the agent
  - Enhanced version requires agent to actively run live tests and validate results
  - Ensures live testing is not deferred or skipped - agent must execute and document evidence
  - Prevents scenarios where tests are defined but never run against real systems
  - Aligns with real-world validation success (001-drift-detection: 217 Azure resources, 33 drift cases)

Previous Amendments:
  v1.1.0 → 1.2.0 (2026-01-06): Added Principle IV - Commit After Every User Story
  v1.0.0 → 1.1.0 (2026-01-06): Added Principle III - Live Testing Is Mandatory
-->

# Constitution

## Core Principles

### I. Code Duplication Prohibited

Code reuse is mandatory to ensure maintainability, consistency, and reduced technical debt.

**Rules**:
- You MUST search for existing functions before creating a new one
- If there are no functions to be reused, then a new one MAY be introduced
- Duplicate implementations of the same logic are strictly prohibited
- Shared utilities MUST be extracted to common modules

**Rationale**: Duplicated code leads to inconsistent behavior, increased maintenance burden, and bugs when fixes are applied to only one copy. This principle ensures a single source of truth for each piece of functionality.

### II. Shared Data Model Is Canonical

The shared data model serves as the single source of truth for all data structures in the project.

**Rules**:
- The shared data model MUST be maintained at `.specify/memory/data_model.md`
- Before introducing, modifying, or duplicating any data structure, you MUST review the existing data model
- If the required data already exists in the data model, it MUST be reused
- If the required data does not exist, it MUST be added to `data_model.md` before being used elsewhere
- All feature specifications MUST reference the canonical data model for entity definitions

**Rationale**: A canonical data model prevents schema drift, ensures consistency across modules, and provides a single reference point for understanding data relationships. This principle eliminates ambiguity about data structures and prevents duplicated or conflicting definitions.

### III. Live Testing Is Mandatory

All features MUST be validated against real production-like environments, not just unit tests or synthetic data. The implementing agent MUST execute live tests and validate results.

**Rules**:
- Every feature specification MUST define Independent Test scenarios that use real external systems (APIs, databases, cloud resources, etc.)
- Implementation plans MUST include a dedicated Live Testing/Validation phase
- The agent MUST execute live tests against actual external systems with real data or representative test data
- The agent MUST validate live test results and document evidence (output files, logs, screenshots, metrics)
- Live test execution tasks MUST be included in the tasks.md file with explicit commands and expected outcomes
- The agent MUST analyze live test results for correctness, errors, and edge cases
- Bugs discovered during live testing MUST be fixed before the feature is considered complete
- The agent MUST NOT proceed to subsequent user stories until live tests for the current story pass validation
- Features that cannot be live tested MUST justify the exception in writing and document alternative validation approaches

**Rationale**: Unit tests and mocked environments miss real-world edge cases, API behavior differences, and integration issues. Live testing with actual external systems reveals bugs that synthetic tests cannot detect. The drift detection feature (001) demonstrates this principle: live Azure testing with 217 real resources uncovered a critical case-sensitivity bug that unit tests missed. Without live validation, this bug would have shipped to production. Requiring the agent to execute and validate tests (not just define them) ensures validation actually occurs and is not deferred or skipped.

### IV. Commit After Every User Story

Each user story must be committed to a git branch when it is complete.

**Rules**:
- A user story is not considered complete until it is committed to a git branch
- Work on a new user story MUST NOT begin until the previous story is committed
- Commits MUST NOT span multiple user stories
- Completed work MUST NOT remain uncommitted across story boundaries

**Rationale**: Requiring a commit after each user story enforces clear completion boundaries, preserves a clean and traceable history, and prevents unrelated changes from being mixed together. This practice improves reviewability, supports reliable rollback and debugging, and keeps development aligned with incremental delivery.


## Development Workflow

### Feature Development Process

All features MUST follow the speckit workflow:

1. **Specification** (`/speckit.specify`): Create feature spec with user stories and requirements
2. **Planning** (`/speckit.plan`): Generate implementation plan with technical decisions
3. **Data Modeling**: Define or reference entities in `.specify/memory/data_model.md`
4. **Task Generation** (`/speckit.tasks`): Create actionable, dependency-ordered task list
5. **Implementation** (`/speckit.implement`): Execute tasks in phases
6. **Analysis** (`/speckit.analyze`): Validate consistency and alignment

### Documentation Standards

- Each feature MUST maintain documentation in `specs/[###-feature-name]/`
- Plans MUST include a Constitution Check section verifying alignment with these principles
- Data entities MUST be documented in the canonical data model before implementation
- All code MUST include clear comments explaining non-obvious logic

### Quality Gates

- Code duplication checks MUST pass before merge
- Data structures MUST be validated against the canonical data model
- Constitution compliance MUST be verified during code review
- All principles are enforceable through automated checks where possible

## Governance

This constitution supersedes all other development practices and guidelines.

**Amendment Process**:
- Amendments require explicit documentation in this file
- Version MUST be incremented according to semantic versioning:
  - MAJOR: Backward-incompatible governance or principle removals/redefinitions
  - MINOR: New principles or materially expanded guidance
  - PATCH: Clarifications, wording, or non-semantic refinements
- All amendments MUST include rationale and effective date
- Dependent templates and scripts MUST be updated to reflect amendments

**Compliance**:
- All feature plans MUST include a Constitution Check gate
- Code reviews MUST verify adherence to these principles
- Violations MUST be justified in the Complexity Tracking section of plan.md
- Unjustified violations MUST be corrected before merge

**Living Document**:
- This constitution evolves with the project
- Regular reviews SHOULD be conducted as the project matures
- Feedback from development experience SHOULD inform amendments

**Version**: 1.3.0 | **Ratified**: 2026-01-05 | **Last Amended**: 2026-01-07
