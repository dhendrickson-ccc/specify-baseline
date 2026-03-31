# How to Create Business Logic Documentation

A comprehensive guide for writing clear, maintainable business logic documentation that serves both technical and non-technical audiences.

---

## Table of Contents

1. [Value of Business Logic Documentation](#value)
2. [Target Audience](#target-audience)
3. [Document Structure Template](#document-structure-template)
4. [Writing Guidelines](#writing-guidelines)
5. [Section-by-Section Breakdown](#section-by-section-breakdown)
6. [Examples and Anti-Patterns](#examples-and-anti-patterns)
7. [Maintenance and Updates](#maintenance-and-updates)
8. [Quality Checklist](#quality-checklist)

---

## Value of Business Logic Documentation {#value}

Business logic documentation serves as a bridge between technical implementation and business understanding. It provides value across multiple dimensions:

### Benefits

**📚 Knowledge Transfer**
- Onboard new team members faster
- Preserve institutional knowledge
- Reduce dependency on specific individuals
- **Example**: New developer understands tag cleanup logic without reading all the code

**🤝 Stakeholder Communication**
- Product owners validate requirements match implementation
- Customer support explains system behavior
- Compliance officers verify regulatory adherence
- **Example**: Support can answer "Why was this rejected?" without escalating to engineering

**🔍 System Understanding**
- Clear explanation of decision logic separate from implementation
- Single source of truth for business rules
- Visual representations of complex flows
- **Example**: Anyone can trace through the decision tree to understand outcomes

**🧪 Test Design**
- QA engineers create comprehensive test cases
- Edge cases explicitly documented
- Expected behavior clearly defined
- **Example**: Test suite covers all documented scenarios

**🛠️ Maintenance**
- Future developers understand intent behind code
- Changes can be validated against documented rules
- Refactoring preserves business logic
- **Example**: Six months later, you remember why the 122-day window exists

### Particularly Valuable For

These types of systems benefit most from explicit business logic documentation:

- **Decision Systems**: Approval/rejection flows, eligibility determination, access control
- **Rule-Based Systems**: Tax calculation, pricing engines, recommendation algorithms
- **Cleanup/Maintenance**: Automated deletion, archival, resource management
- **Compliance-Critical**: Audit trails, regulatory requirements, data handling
- **Complex State Machines**: Multi-step workflows, orchestration, event processing

However, even simple systems can benefit from documenting their business logic when it provides clarity for stakeholders beyond the development team.

---

## Target Audience

### Primary Audiences

**1. Business Stakeholders (Non-Technical)**
- Product owners, managers, compliance officers
- Need: Understand WHAT the system does and WHY
- Read for: Validation, requirement verification, stakeholder communication
- Format preference: Plain language, examples, visual aids

**2. New Developers (Onboarding)**
- Engineers unfamiliar with the system
- Need: Understand business context before reading code
- Read for: Learning domain logic, understanding intent
- Format preference: Examples, decision flows, cross-references to code

**3. QA/Test Engineers**
- Writing tests, validating behavior
- Need: Comprehensive rule coverage, edge cases
- Read for: Test case generation, validation criteria
- Format preference: Complete rule sets, boundary conditions, examples

**4. Customer Support**
- Explaining behavior to customers
- Need: Simple explanations for common scenarios
- Read for: Answering "Why did this happen?" questions
- Format preference: Common examples, FAQs

### Secondary Audiences

**5. External Auditors/Compliance**
- Verifying regulatory compliance
- Need: Proof of business rule implementation
- Read for: Audit trails, compliance verification

**6. Future You (6 months later)**
- Maintaining or extending the system
- Need: Remember why decisions were made
- Read for: Context for changes

---

## Document Structure Template

### Required Sections

Every business logic document should include:

```markdown
# [System Name] - Business Rules

## 1. Overview
- Purpose of the system
- High-level decision logic
- Philosophy/guiding principles

## 2. Core [Action] Rules
- Primary decision criteria
- When [primary action] happens
- Clear, numbered list

## 3. Core [Protection/Exception] Rules
- Exception cases
- When [primary action] does NOT happen
- Clear, numbered list

## 4. [Concept] Logic
- Key concepts explained (e.g., matching, normalization)
- How comparisons work
- What values are considered equal/different

## 5. Edge Cases and Special Scenarios
- Unusual situations
- Boundary conditions
- One subsection per edge case

## 6. [Key Parameter] Details
- Thresholds, timeouts, limits
- Why those values
- Configurable vs fixed

## 7. Decision Flow
- Visual flowchart or decision tree
- ASCII art or description
- Shows complete decision path

## 8. Concrete Examples
- Real-world scenarios
- 4-6 examples covering common cases
- Input → Decision → Reason format

## 9. Test Coverage (Optional)
- Reference to test suites
- How to run validation tests

## 10. Implementation References (Optional)
- Key code locations
- For developers who want to see implementation

## 11. FAQs
- Common questions from stakeholders
- Clarifications on confusing points
```

### Optional Sections

Add these if relevant:

- **Glossary** - Define domain terms
- **Historical Context** - Why rules exist, what problems they solve
- **Future Considerations** - Known limitations, planned changes
- **Related Systems** - How this integrates with other components
- **Compliance Notes** - Regulatory requirements met by these rules

---

## Writing Guidelines

### 1. Use Plain Language

#### ✅ Do:
- Write like you're explaining to a smart colleague from a different field
- Use everyday words
- Define technical terms when first used
- Use active voice

#### ❌ Don't:
- Use code variable names in prose
- Assume reader knows implementation details
- Use jargon without explanation
- Write in passive voice

**Example:**

✅ Good:
```
Tags are deleted when they're older than 92 days and not deployed to production.
```

❌ Bad:
```
When age_days > AGE_THRESHOLD_DAYS and in_production == False, tags are added
to deletion_candidates list.
```

---

### 2. Structure Rules Clearly

#### Format for Rule Statements

Use this structure for clarity:

```
[Action] happens when [ALL/ANY] of the following are true:
1. [Condition 1]
2. [Condition 2]
3. [Condition 3]

If [ANY] condition is false, [alternative action] happens instead.
```

#### ✅ Do:
- Use numbered lists for multiple conditions
- State explicitly whether it's AND (all) or OR (any)
- Show priority when rules conflict

#### Example:

```
A tag is DELETED when ALL of the following are true:
1. Tag exists in GitHub
2. Tag age > 92 days
3. NOT deployed to production
4. Either:
   - Matches a Terraform version (after normalization), OR
   - Has no matching Terraform version (orphaned)

If ANY condition is false, the tag is PROTECTED.
```

---

### 3. Include Context Always

#### ✅ Do:
- Show thresholds alongside values
- Explain what values mean
- Provide units (days, bytes, percentages)
- Reference standards or reasons

#### Example:

✅ Good:
```
Age Threshold: 92 days (approximately 3 months)
Purpose: Provides safety buffer for recent work
Configurable: Yes, via AGE_THRESHOLD_DAYS environment variable
```

❌ Bad:
```
Age Threshold: 92 days
```

---

### 4. Use Examples Extensively

#### Example Format

Every complex rule should have at least one concrete example:

```
**Example: [Scenario Name] → [Outcome]**

[Input values formatted clearly]
Input A:          [value]
Input B:          [value]
Condition:        [state]

Decision:         [KEEP/DELETE/APPROVE/REJECT/etc.]
Reason:           [Plain language explanation]
Rule Applied:     [Which rule from section X]
```

#### ✅ Do:
- Use realistic values
- Show complete input and output
- Explain the reasoning
- Reference back to the rule section

---

### 5. Visual Decision Flows

#### ASCII Flowcharts

Use simple ASCII art for decision trees:

```
Input → Check Condition A?
            ↓
       ┌────┴────┐
      YES        NO
       ↓          ↓
   Check B?    Outcome X
       ↓
   ┌───┴───┐
  YES     NO
   ↓       ↓
Outcome Y Outcome Z
```

#### ✅ Do:
- Keep it simple and readable
- Use consistent symbols
- Label all branches
- Show complete paths

#### ❌ Don't:
- Make it too complex (split into multiple diagrams)
- Use special characters that don't render well
- Assume reader can follow complex branching

---

### 6. Define Edge Cases Explicitly

#### Edge Case Template

```
### [Edge Case Name]

**Scenario**: [What makes this special]

**Example**:
- [Concrete values]

**Decision**: [What happens]

**Reason**: [Why this is the right behavior]

**Rationale**: [Business justification]
```

#### ✅ Do:
- Give each edge case its own subsection
- Explain WHY the edge case exists
- Show what makes it different from normal cases
- Include business rationale

---

### 7. Link to Implementation (For Developers)

#### Implementation Reference Format

At the end of the document, include:

```
## Implementation References

For developers wanting to see the code:

- [Business Rule Name]: [file_path:line_number]
- [Threshold Check]: [file_path:line_number]
- [Decision Logic]: [file_path:line_number-line_number]
```

#### ✅ Do:
- Put this section at the END
- Use specific line numbers
- Keep it brief (just pointers)
- Make it optional reading

#### ❌ Don't:
- Mix code references into business rule explanations
- Show code snippets in main sections
- Assume reader will look at code

---

## Section-by-Section Breakdown

### Section 1: Overview

**Purpose**: Set context and high-level understanding

**Include:**
- What the system does (1-2 sentences)
- Primary decision or action it makes
- Key stakeholders or users affected
- Philosophy or guiding principles

**Length**: 1-2 paragraphs

**Example:**
```
## Overview

The Tag Cleanup Lambda automatically removes outdated GitHub tags and
Terraform Cloud module versions that are no longer needed in production.
This document explains the business logic that determines which tags are
deleted and which are protected.

### Philosophy

The Lambda follows a conservative, fail-safe approach:
- When in doubt, protect the tag
- All API errors result in no deletions
- Recent tags (< 92 days) are always protected
- Production-deployed versions (within last 122 days) are protected
```

---

### Section 2: Core [Action] Rules

**Purpose**: State the primary business rules clearly

**Include:**
- Main action the system takes (DELETE, APPROVE, REJECT, etc.)
- ALL conditions that must be true
- Clear AND/OR logic
- What happens if conditions aren't met

**Format:**
```
## Core [Action] Rules

[Action] happens ONLY when ALL of the following are true:

1. ✅ [Condition 1] - [Brief explanation]
2. ✅ [Condition 2] - [Brief explanation]
3. ✅ [Condition 3] - [Brief explanation]
4. ✅ ONE of the following:
   - [Option A], OR
   - [Option B]

If ANY condition is false, [alternative action] happens.
```

**Length**: 1 page maximum

**Example:**
```
## Core Deletion Rules

A tag is deleted ONLY when ALL of the following are true:

1. ✅ Tag exists in GitHub (can't delete what doesn't exist)
2. ✅ Tag age > 92 days (approximately 3 months)
3. ✅ NOT deployed to production (not found in recent git history)
4. ✅ One of the following:
   - Tag matches a Terraform version (after normalization), OR
   - Tag has NO matching Terraform version at all (orphaned)

If ANY condition is false, the tag is PROTECTED (not deleted).
```

---

### Section 3: Core [Protection/Exception] Rules

**Purpose**: State when the primary action does NOT happen

**Include:**
- Protection/exception conditions
- ANY logic (only one needs to be true)
- Priority ordering if relevant
- Override behavior

**Format:**
```
## Core [Protection/Exception] Rules

[Entity] is [protected/excepted/kept] when ANY of the following are true:

1. 🛡️ [Condition 1] - [Explanation]
2. 🛡️ [Condition 2] - [Explanation]
3. 🛡️ [Condition 3] - [Explanation]

Priority: [Which rule wins if multiple apply]
```

**Example:**
```
## Core Protection Rules

A tag is protected (NEVER deleted) when ANY of the following are true:

1. 🛡️ Tag age ≤ 92 days - Too recent to delete
2. 🛡️ Version in production - Found in git commits (last 122 days)
3. 🛡️ System error occurred - API failures trigger fail-safe

Priority: Protection always wins over deletion eligibility.
```

---

### Section 4: [Concept] Logic

**Purpose**: Explain key concepts that affect decisions

**Include:**
- How key comparisons work
- What normalization/transformation happens
- What values are considered "equal"
- What values are considered "different"

**Format:**
```
## [Concept] Logic

### How [Items] Match

[Explain the comparison logic]

### [Transformation] Rules

What IS transformed:
- ✅ [Rule 1] - [Example: v1.0.0 → 1.0.0]

What is NOT transformed:
- ❌ [Rule 2] - [Example: 1.0.0+build.123 stays as-is]

### Matching Examples

| Input A | Input B | Match? | Reason |
|---------|---------|--------|--------|
| [val]   | [val]   | ✅ YES | [why]  |
| [val]   | [val]   | ❌ NO  | [why]  |
```

**Length**: 1-2 pages with examples

---

### Section 5: Edge Cases

**Purpose**: Document special scenarios and boundary conditions

**Include:**
- One subsection per edge case (4-8 edge cases typical)
- Concrete example
- Why it's handled that way
- Business rationale

**Format:**
```
## Edge Cases and Special Scenarios

### 1. [Edge Case Name]

**Scenario**: [What makes this special]

**Example**:
- Input: [values]
- Context: [special condition]

**Decision**: [What happens]

**Reason**: [Technical explanation]

**Rationale**: [Business justification]

---

### 2. [Next Edge Case]
[Same format]
```

**Example:**
```
## Edge Cases and Special Scenarios

### 1. Orphaned Tags (GitHub-Only)

**Scenario**: GitHub tag exists but has NO matching Terraform version

**Example**:
- GitHub: v0.3.0 (age: 181 days)
- Terraform: (no matching version)

**Decision**: DELETE if age > 92 days

**Reason**: Version was never published to Terraform Cloud, or was removed.

**Rationale**: If not referenced anywhere, safe to clean up.
```

---

### Section 6: [Key Parameters] Details

**Purpose**: Document important thresholds, limits, timeouts

**Include:**
- Default value
- Why that value
- Configurable or fixed
- Units and context
- What happens at boundaries

**Format:**
```
## [Parameter] Details

### Default Value: [X] [units]

- **Value**: [X] [units] ([human-readable equivalent])
- **Configurable**: [Yes/No] via [mechanism]
- **Purpose**: [Why this value]

### How [Parameter] is Calculated

[Step-by-step explanation]

### Why [This Value]?

[Business reasoning for the default]

### [Parameter]-Based [Action]

[What happens based on this parameter]
```

**Length**: 1 page per major parameter

---

### Section 7: Decision Flow

**Purpose**: Visual representation of decision logic

**Include:**
- Complete decision tree
- All paths from input to output
- Clear labels and branches
- Legend if needed

**Format:**
```
## Decision Flow

[ASCII flowchart showing complete decision path]

**Legend**:
- [Symbol] = [Meaning]
- [Symbol] = [Meaning]
```

**Tips:**
- Keep simple - split complex flows into multiple diagrams
- Show happy path first
- Label all decision points
- Show all outcomes

---

### Section 8: Concrete Examples

**Purpose**: Show real-world scenarios with complete details

**Include:**
- 4-6 examples covering major scenarios
- Mix of positive and negative outcomes
- Edge cases represented
- Complete input/output for each

**Format:**
```
## Concrete Examples

### Example 1: [Scenario Name] → [Outcome]

[Input values in readable format]
Input A:          [value]
Input B:          [value]
Condition:        [state]

Decision:         [ACTION]
Reason:           [Plain language explanation]
Rule Applied:     [Reference to section]

**Explanation**: [1-2 sentence walkthrough]

---

### Example 2: [Next Scenario] → [Outcome]
[Same format]
```

**Quality check:**
- Can a non-technical person understand why the decision was made?
- Is every major rule represented in at least one example?
- Do examples show edge cases?

---

## Examples and Anti-Patterns

### Writing Style Examples

#### ❌ Too Technical
```
The system checks if `age_days > age_threshold_days` and `in_production == False`.
If both conditions evaluate to True, the entity is appended to the
`deletion_candidates` list. Otherwise, it's added to `protected_by_age` or
`protected_by_production` based on the condition that evaluated to False.
```

**Problems:**
- Uses code variable names
- Uses programming concepts (evaluate to True, appended)
- Describes implementation, not business logic

---

#### ✅ Business Focused
```
A tag is deleted when it's both old (over 92 days) and not currently deployed
to production. If the tag is too recent, it's automatically protected. If it's
deployed to production, it's also protected regardless of age. This ensures
we never accidentally delete active or recent tags.
```

**Why it works:**
- Plain language
- Business concepts (old, deployed, protected)
- Explains the "why"

---

### Rule Statement Examples

#### ❌ Vague
```
Tags older than the threshold might be deleted, depending on various factors
like whether they're in use or not.
```

**Problems:**
- "Might be" is ambiguous
- "Various factors" is unclear
- "In use" is undefined

---

#### ✅ Precise
```
Tags are deleted when ALL of the following are true:
1. Tag age > 92 days (threshold)
2. NOT found in production git commits (last 122 days)
3. Either matches a Terraform version OR is orphaned

If ANY condition is false, the tag is protected.
```

**Why it works:**
- ALL/ANY logic explicit
- Specific threshold values
- Clear conditions
- Defined alternative outcome

---

### Example Format Comparison

#### ❌ Incomplete
```
Example: Old tag → deleted
Tag: v1.0.0
Result: deleted
```

**Problems:**
- Missing context (age, deployment status)
- No reasoning
- Can't understand why

---

#### ✅ Complete
```
Example 1: Old Tag, Not Deployed → DELETE

GitHub Tag:       v1.0.0
Age:              181 days (> 92 day threshold)
Terraform:        1.0.0 (exists in history)
Deployed:         No

Decision:         DELETE (eligible for cleanup)
Reason:           Old version no longer used in production
Rule Applied:     Age > 92 days + not deployed = deletion candidate

Explanation: Even though this version exists in Terraform history, it's
not currently deployed and has been around for over 6 months. Safe to delete.
```

**Why it works:**
- Complete input context
- Shows threshold comparison
- Explains decision
- References business rule
- Adds walkthrough explanation

---

## Maintenance and Updates

### When to Update Documentation

Update business logic docs when:

1. **Business rules change** - New thresholds, modified logic
2. **Edge cases discovered** - Add to edge cases section
3. **Confusion arises repeatedly** - Add to FAQ
4. **Implementation changes affect behavior** - Update examples
5. **New features added** - Expand scope

### Update Process

```
1. Update business rules doc FIRST (source of truth)
   ↓
2. Update test fixtures/expectations (if tests validate rules)
   ↓
3. Update implementation code (make it match docs)
   ↓
4. Run tests (verify alignment)
   ↓
5. Update test output helpers (if needed for new terminology)
```

### Keeping Docs and Code in Sync

**Strategy 1: Regular Review**
- Schedule quarterly reviews
- Check if docs match current behavior
- Update examples to stay relevant

**Strategy 2: Change Policy**
- Require doc updates with code PRs
- Add "Documentation" section to PR template
- Review docs changes in code review

**Strategy 3: Automated Checks**
- Link docs to tests
- Fail build if tests change without doc updates
- Version docs alongside code

**Strategy 4: Living Document**
- Keep docs in same repo as code
- Link from README
- Make docs searchable/discoverable

---

## Quality Checklist

### Before Publishing Business Logic Documentation

Use this checklist to validate quality:

#### ✅ Audience

- [ ] Identified primary audience (who will read this most?)
- [ ] Written at appropriate technical level
- [ ] No unexplained jargon or code terms
- [ ] Non-technical stakeholder could understand core concepts

#### ✅ Completeness

- [ ] All major decision paths documented
- [ ] Edge cases identified and explained
- [ ] Examples cover common scenarios
- [ ] FAQs address known confusion points

#### ✅ Clarity

- [ ] Rules stated explicitly (not "might" or "could")
- [ ] AND/OR logic clearly indicated
- [ ] Thresholds and limits documented with context
- [ ] Decision flow is traceable

#### ✅ Examples

- [ ] 4-6 concrete examples included
- [ ] Examples show input, decision, reasoning
- [ ] Edge cases represented
- [ ] Examples use realistic values

#### ✅ Structure

- [ ] Sections follow logical flow
- [ ] Core rules stated early
- [ ] Details/implementation references at end
- [ ] Table of contents for navigation

#### ✅ Formatting

- [ ] Consistent formatting throughout
- [ ] Headers properly nested
- [ ] Lists used appropriately
- [ ] Visual elements (flowcharts, tables) render correctly

#### ✅ Integration

- [ ] References to related documentation
- [ ] Links to test suites
- [ ] Cross-references to code (optional, at end)
- [ ] Terminology matches other docs

#### ✅ Maintenance

- [ ] Date of last update shown
- [ ] Clear ownership (who maintains this)
- [ ] Version control (in git)
- [ ] Process for updates documented

---

## Template: Quick Start

Copy this template to start a new business logic document:

```markdown
# [System Name] - Business Rules

> Last Updated: [Date]
> Owner: [Team/Person]

## Overview

[What does this system do? 1-2 sentences]

### Purpose

- [Primary goal]
- [Key stakeholders affected]

### Philosophy

- [Guiding principle 1]
- [Guiding principle 2]

---

## Core [Action] Rules

[Action] happens when ALL of the following are true:

1. ✅ [Condition 1]
2. ✅ [Condition 2]
3. ✅ [Condition 3]

If ANY condition is false, [alternative outcome].

---

## Core [Protection/Exception] Rules

[Entity] is [protected/excepted] when ANY of the following are true:

1. 🛡️ [Condition 1]
2. 🛡️ [Condition 2]

---

## [Key Concept] Logic

### How [Items] Match

[Explain matching/comparison logic]

### [Transformation] Rules

What IS transformed:
- ✅ [Rule]

What is NOT transformed:
- ❌ [Rule]

### Matching Examples

| Input A | Input B | Match? | Reason |
|---------|---------|--------|--------|
| [val]   | [val]   | ✅ YES | [why]  |

---

## Edge Cases and Special Scenarios

### 1. [Edge Case Name]

**Scenario**: [What makes this special]

**Example**: [Concrete values]

**Decision**: [What happens]

**Reason**: [Why]

**Rationale**: [Business justification]

---

## [Key Parameter] Details

### Default Value: [X] [units]

- **Value**: [X]
- **Configurable**: [Yes/No]
- **Purpose**: [Why this value]

### Why [This Value]?

[Business reasoning]

---

## Decision Flow

```
[ASCII flowchart]
```

**Legend**:
- [Symbol] = [Meaning]

---

## Concrete Examples

### Example 1: [Scenario] → [Outcome]

```
Input A:          [value]
Input B:          [value]
Condition:        [state]

Decision:         [ACTION]
Reason:           [Plain language]
Rule Applied:     [Reference to section]
```

**Explanation**: [Walkthrough]

---

## FAQs

### Q: [Common question]?

**A**: [Clear answer with reference to relevant section]

---

## Related Documentation

- [Link to related doc]
- [Link to test suite]

---

## Implementation References

For developers wanting to see implementation:

- [Rule Name]: [file:line]
- [Decision Logic]: [file:line-line]
```

---

## Summary

### Key Principles for Business Logic Documentation:

1. **Write for non-technical readers first** - They're the primary audience
2. **State rules explicitly** - No ambiguity, clear AND/OR logic
3. **Include context always** - Thresholds, units, reasoning
4. **Use examples extensively** - Show, don't just tell
5. **Structure matters** - Core rules first, details later
6. **Keep it updated** - Docs rot faster than code
7. **Make it discoverable** - Link from README, keep in repo

### The Test:

Can a product manager read your business logic doc and:
- ✅ Understand what the system does
- ✅ Explain why specific decisions are made
- ✅ Validate it matches requirements
- ✅ Answer customer questions about behavior

If yes, you've succeeded.

---

## Appendix: Real-World Example

See [../../docs/tag-cleanup-business-rules.md](../../docs/tag-cleanup-business-rules.md) for a complete implementation of this guide.

This example demonstrates:
- Clear rule statements
- Complete edge case documentation
- Extensive examples
- Visual decision flow
- Business-focused language throughout
- Integration with test suite
