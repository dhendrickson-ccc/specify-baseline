# Test Output Design - Meta Documentation

## Purpose

This document explains the design philosophy, target audience, and evaluation criteria for the integration test output system implemented in spec 004.

---

## Executive Summary

The integration test suite uses **business-rule-focused output** instead of technical implementation details. Tests provide two verbosity levels optimized for different audiences:

- **Minimal Mode**: Business stakeholders, product owners, non-technical reviewers
- **Full Mode**: Developers, QA engineers, debugging scenarios

The output is designed to be **self-documenting** - anyone reading it can understand what's being tested and why a decision was made, without needing to read implementation code.

---

## Target Personas

### Primary Persona: Business Stakeholder

**Who they are:**
- Product owners
- Business analysts
- DevOps managers
- Compliance auditors

**What they need:**
- Understand WHAT is being tested
- See WHY a tag was deleted or protected
- Validate business logic matches requirements
- No implementation details

**Example question they ask:**
> "Why was version v1.0.0-rc.1+build.456 deleted? I thought it was in production?"

**How output helps:**
```
Tag:              v1.0.0-rc.1+build.456
Match Status:     'v1.0.0-rc.1+build.456' ≠ '1.0.0-rc.1' (GitHub-only tag)
Age:              181 days (threshold: 92 days)
Deployed:         No
Rule Applied:     Delete (age 181d > 92d threshold + not deployed)
```
Answer: Build metadata makes it different from the production version `1.0.0-rc.1`, so it's treated as orphaned.

---

### Secondary Persona: QA Engineer / Test Maintainer

**Who they are:**
- QA engineers validating behavior
- Test maintainers updating fixtures
- Integration test authors

**What they need:**
- Full context for debugging
- All input values
- Expected vs actual comparison
- Technical details when needed

**Example question they ask:**
> "This test is failing - what are all the inputs and what did the implementation actually return?"

**How output helps:**
```
======================================================================
Test: Complex version: pre-release + build metadata
======================================================================
Tag:              v1.0.0-rc.1+build.456
GitHub Date:      2025-10-01
TF Versions:      ['1.0.0-rc.1']
TF Deployed:      False
Expected Action:  delete
Expected Reason:  old_prerelease
Actual Category:  deletion_candidates
Tag Age (days):   181
Protection Info:  N/A
======================================================================
```
Answer: All inputs visible, can verify expected vs actual matches.

---

### Tertiary Persona: New Developer

**Who they are:**
- Onboarding engineers
- Contributors unfamiliar with the codebase
- External developers reading tests

**What they need:**
- Learn business rules through examples
- Understand tag cleanup logic
- See real scenarios in action

**Example question they ask:**
> "How does the Lambda decide what to delete? I want to understand without reading all the code."

**How output helps:**
Run tests with minimal output, read 44 real scenarios showing business logic in action. Each test shows:
- Input (tag, age, deployment status)
- Business rule applied
- Decision made (keep/delete)
- Reasoning

---

## Design Criteria

### What Makes "Good" Test Output?

#### ✅ Clarity Over Completeness
- Show only what's needed to understand the decision
- Don't overwhelm with technical details
- Use plain language, not code terminology

**Good:**
```
Rule Applied:     Delete (age 181d > 92d threshold + not deployed)
```

**Bad:**
```
Evaluation: age_days=181 > age_threshold_days=92 AND in_production=False → deletion_candidates.append()
```

---

#### ✅ Business Logic Focus
- Explain WHAT rule was applied, not HOW it was implemented
- Reference business concepts (age threshold, production, orphaned)
- Avoid implementation details (variable names, function calls, data structures)

**Good:**
```
Match Status:     'v1.0.0+build.123' ≠ '1.0.0' (GitHub-only tag)
Deployed:         No
```

**Bad:**
```
Normalized versions: tag_normalized='1.0.0+build.123' not in protected_versions dict
in_production = nver in protected_versions → False
```

---

#### ✅ Self-Documenting
- Output should be understandable without external documentation
- Include context (thresholds, what "deployed" means)
- Make comparison logic explicit

**Good:**
```
Age:              181 days (threshold: 92 days)
```
Shows both the value AND the threshold for comparison.

**Bad:**
```
Age:              181 days
```
Requires knowledge of what the threshold is.

---

#### ✅ Traceable to Business Rules
- Every output line should map to a section in [business rules doc](../../docs/tag-cleanup-business-rules.md)
- Use same terminology as business rules
- Show which rule "won" when multiple rules apply

**Good:**
```
Rule Applied:     Keep (age 45d < 92d threshold)
```
Maps directly to: "Tags with age ≤ 92 days are automatically protected"

---

#### ✅ Consistent Format
- Same structure across all tests
- Predictable information placement
- Easy to scan and compare

**Format:**
```
Tag:              [value]
Match Status:     [comparison] (optional)
Age:              [days] (threshold: [threshold])
Deployed:         [yes/no]
Rule Applied:     [decision with reasoning]
──────────────────────────────────────────────────────────────────────
✅/❌ [pass/fail message]
```

---

## Verbosity Levels

### Minimal Mode (`TEST_VERBOSITY=minimal` or `TEST_VERBOSITY=1`)

**Target Audience**: Business stakeholders, product owners

**Philosophy**: "Just enough to understand the decision"

**Shows:**
- Tag name
- Version matching status (if relevant)
- Age vs threshold
- Deployment status
- Rule applied with reasoning
- Pass/fail

**Omits:**
- GitHub commit dates
- Terraform version lists (unless testing matching)
- Expected action/reason (shown only on failure)
- Protection reasons (technical terms)
- Category names (implementation detail)

**Example:**
```
Tag:              v1.0.0
Age:              181 days (threshold: 92 days)
Deployed:         Yes (in production)
Rule Applied:     Keep (deployed to production)
──────────────────────────────────────────────────────────────────────
✅ PASS: Tag correctly protected by production
```

**When to use:** CI/CD pipelines, smoke tests, stakeholder demos

---

### Full Mode (`TEST_VERBOSITY=full` or `TEST_VERBOSITY=2`)

**Target Audience**: Developers, QA engineers, debugging

**Philosophy**: "Everything you need to debug or maintain tests"

**Shows:**
- All minimal mode information
- Test description
- Full input data (GitHub date, TF versions list)
- Expected action and reason
- Actual category (implementation term)
- Protection info (technical term)
- Age in days (exact number)

**Example:**
```
======================================================================
Test: Old tag deployed to production - must keep
======================================================================
Tag:              v1.0.0
GitHub Date:      2025-10-01
TF Versions:      ['1.0.0']
TF Deployed:      True
Expected Action:  keep
Expected Reason:  deployed_to_production
Actual Category:  protected_by_production
Tag Age (days):   181
Protection Info:  used_in_production
======================================================================
✅ PASS: Tag correctly protected by production
```

**When to use:** Local development, debugging test failures, updating fixtures

---

### No Verbosity (Default)

**Shows:** Only pytest standard output (test names, pass/fail)

**When to use:** Quick validation, final CI check, minimal output needed

---

## Evolution of the Design

### Original Approach (Before User Feedback)
```
Test: Complex version: pre-release + build metadata
Tag: v1.0.0-rc.1+build.456
Expected: delete
Actual: deletion_candidates
✅ PASS
```

**Problem:** Didn't explain WHY the decision was made.

---

### First Iteration (Technical Details)
```
Comparing:        v1.0.0-rc.1+build.456 should match ['1.0.0-rc.1']
Normalized Tag:   1.0.0-rc.1+build.456
TF Versions:      ['1.0.0-rc.1']
Actual Category:  deletion_candidates
Tag Age (days):   181
Deployed:         No
```

**Problem:** Too technical. Requires understanding normalization, categories, etc.

**User Feedback:** "This doesn't make sense if you don't have a ton of context about how this is written."

---

### Second Iteration (Business Rules Focus) ✅
```
Tag:              v1.0.0-rc.1+build.456
Match Status:     'v1.0.0-rc.1+build.456' ≠ '1.0.0-rc.1' (GitHub-only tag)
Age:              181 days (threshold: 92 days)
Deployed:         No
Rule Applied:     Delete (age 181d > 92d threshold + not deployed)
```

**Solution:** Focus on business logic, show clear comparison, explain decision in business terms.

**Key changes:**
- Removed "Normalized Tag" → "Match Status" with clear comparison
- Removed "Actual Category" → "Rule Applied" with reasoning
- Added threshold context to age
- Used plain language for deployment status

---

## Success Metrics

### How to Judge if Test Output is Effective

#### 1. **Comprehension Test**
Can someone unfamiliar with the code answer:
- ✅ What is being tested?
- ✅ What was the decision (keep/delete)?
- ✅ Why was that decision made?

**Without:**
- ❌ Reading the implementation code
- ❌ Understanding Python
- ❌ Knowing implementation details

---

#### 2. **Correlation Test**
Does the output map to business rules documentation?
- ✅ Terminology matches [business rules doc](../../docs/tag-cleanup-business-rules.md)
- ✅ Can find the rule in docs that explains the decision
- ✅ Reasoning uses business concepts, not code concepts

---

#### 3. **Debugging Efficiency Test**
When a test fails, can a developer:
- ✅ Understand what went wrong from the output alone?
- ✅ Determine if it's a test bug or implementation bug?
- ✅ Know what fixture values to check?

---

#### 4. **Onboarding Test**
Can a new team member:
- ✅ Learn the business rules by reading test output?
- ✅ Understand tag cleanup logic without reading implementation?
- ✅ Identify edge cases from test descriptions?

---

## Examples: Good vs Bad Output

### Scenario: Testing Build Metadata Handling

#### ❌ Bad (Too Technical)
```
Normalized: tag_name.lstrip('v') = '1.0.0-rc.1+build.456'
Expected: nver in protected_versions → True
Actual: '1.0.0-rc.1+build.456' in {'1.0.0-rc.1': {...}} → False
Result: deletion_candidates
```

**Problems:**
- Shows implementation details (lstrip, dict lookup)
- Uses code variable names (nver, protected_versions)
- No business context

---

#### ✅ Good (Business Logic)
```
Tag:              v1.0.0-rc.1+build.456
Match Status:     'v1.0.0-rc.1+build.456' ≠ '1.0.0-rc.1' (GitHub-only tag)
Age:              181 days (threshold: 92 days)
Deployed:         No
Rule Applied:     Delete (age 181d > 92d threshold + not deployed)
```

**Why it works:**
- Shows business comparison (tag vs version)
- Explains classification (GitHub-only)
- References business rule (age threshold)
- Clear decision with reasoning

---

### Scenario: Testing Age-Based Protection

#### ❌ Bad (Missing Context)
```
Tag: v2.1.0
Age: 45 days
Protected: Yes
```

**Problems:**
- No threshold shown (why is 45 days protected?)
- "Protected: Yes" doesn't explain the rule
- No comparison to make it clear

---

#### ✅ Good (With Context)
```
Tag:              v2.1.0
Age:              45 days (threshold: 92 days)
Deployed:         No
Rule Applied:     Keep (age 45d < 92d threshold)
```

**Why it works:**
- Shows threshold for comparison
- Makes inequality clear (45 < 92)
- Explains the rule being applied

---

## Design Principles Summary

### Core Principles

1. **Business Logic Over Implementation** - Show what rules apply, not how code works
2. **Context Always** - Include thresholds, comparisons, reasoning
3. **Plain Language** - Avoid code terminology and jargon
4. **Scannable Format** - Consistent structure, easy to read
5. **Self-Documenting** - Understandable without external docs
6. **Persona-Aware** - Two modes for different audiences

---

### Language Guidelines

#### Use This:
- ✅ "deployed to production"
- ✅ "GitHub-only tag"
- ✅ "age threshold"
- ✅ "eligible for deletion"
- ✅ "protected by production"

#### Not This:
- ❌ "in_production = True"
- ❌ "not in protected_versions dict"
- ❌ "age_threshold_days constant"
- ❌ "added to deletion_candidates list"
- ❌ "protected_by_production category"

---

### Format Guidelines

#### Every test should show:
1. **Input**: What's being evaluated (tag name, age, deployment)
2. **Comparison**: How it's being evaluated (vs threshold, vs versions)
3. **Decision**: What rule was applied (keep/delete)
4. **Reasoning**: Why that rule was applied (age < threshold, deployed, etc.)

#### Structure:
```
[Input values]
[Comparison/matching if relevant]
[Business rule evaluation]
[Separator line]
[Pass/fail with category]
```

---

## Integration with Documentation

The test output is designed to work in tandem with [business rules documentation](../../docs/tag-cleanup-business-rules.md):

### Documentation Structure:
1. **Business Rules Doc** - Complete reference, detailed explanations
2. **Test Output** - Live examples showing rules in action
3. **Test Fixtures** - Input data for scenarios

### Flow:
```
Business Rule → Test Case → Test Output
     ↓              ↓            ↓
  "Age ≤ 92   →  v2.1.0     →  "Age: 45 days (threshold: 92 days)
   days =          45 days       Rule Applied: Keep (age 45d < 92d threshold)"
   KEEP"
```

### Cross-References:
- Test output uses same terminology as business rules
- Each test maps to a section in business rules doc
- Business rules doc includes concrete examples matching test output format

---

## Future Improvements

### Potential Enhancements:

1. **Color Coding** (if terminal supports):
   - 🟢 Protected tags (green)
   - 🔴 Deletion candidates (red)
   - 🟡 Edge cases (yellow)

2. **Links to Business Rules**:
   - Output could include: "See: docs/tag-cleanup-business-rules.md#age-threshold-details"

3. **Summary Statistics**:
   - Show overall pass rate
   - Show rules coverage (which rules were tested)

4. **Interactive Mode**:
   - Allow drilling into specific tests
   - Show alternative scenarios

5. **Comparison Mode**:
   - Side-by-side comparison of similar tests
   - Highlight what changed

---

## Maintenance Guidelines

### When Adding New Tests:

1. **Choose appropriate verbosity for assertions** - What info does failure message need?
2. **Use business terminology** - Match business rules doc
3. **Include context** - Show thresholds, comparisons
4. **Test the output** - Read it as if you're a stakeholder

### When Updating Output Format:

1. **Update all tests consistently** - Don't leave some with old format
2. **Update this meta doc** - Document the change
3. **Consider both personas** - Does minimal and full mode still work?
4. **Verify against business rules** - Does terminology still match?

### When Business Rules Change:

1. **Update business rules doc first** - Source of truth
2. **Update test fixtures** - Change expectations
3. **Update test output helpers** - Match new terminology
4. **Run tests** - Verify output makes sense

---

## Conclusion

The test output design prioritizes **clarity and business value** over technical completeness. By focusing on business logic and using plain language, the tests become:

- **Documentation** - Teaching tool for new team members
- **Validation** - Proof that business rules are implemented correctly
- **Communication** - Bridge between technical and non-technical stakeholders

The dual-verbosity system ensures both audiences get what they need without overwhelming either group with unnecessary detail.

**Key Takeaway**: Good test output should answer "What and Why" for stakeholders, while also providing "How" details for developers when needed.
