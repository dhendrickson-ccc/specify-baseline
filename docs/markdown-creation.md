When creating or editing documentation in this repo, follow these parameters. They reflect established conventions for how docs should be structured, scoped, and written. Apply them to all markdown files — READMEs, guides, and inline docs. If existing documentation violates these parameters, flag it or fix it rather than conforming to the existing pattern. Conciseness is a feature, not a tradeoff.

**1. One source of truth.** Each piece of information lives in exactly one file. If two docs cover the same topic, merge them. Never maintain parallel copies.

**2. Audience-aware urgency.** Match the content to the reader's state of mind. Emergency sections get copy-paste commands that solve the immediate problem. Setup sections can be more explanatory. A panicking engineer and a new-hire onboarding are different readers.

**3. Don't document the obvious.** Skip things the reader already knows from context — `terraform init/plan/apply` in a Terraform repo, basic pytest usage in a Python project, directory trees visible by browsing.

**4. Commands over descriptions.** "Disable the schedule via `schedule_enabled = false`" is worse than a `aws lambda put-function-concurrency` block they can paste. Tell, don't hint.

**5. Tables over prose for reference data.** Configuration variables, matching rules, examples — use tables. Prose is for explaining *why*, not for listing *what*.

**6. Cut what exists elsewhere.** If a sub-README, the code itself, or running the tool already provides the info (test templates, directory structure, coverage percentages), don't duplicate it in the parent doc.

**7. Consolidate aggressively.** Fewer files with clear `##` sections beat many small files. Each file should justify its own existence.

**8. Compress, then explain.** Lead with the dense reference (table, code block), then add a brief explanation only if the reference isn't self-evident.

**9. Section earns its space.** Every section should answer a question someone would actually ask. "How do I stop this right now?" yes. "What are the expected execution times?" no — run it and see.

**10. Link, don't repeat.** If a sub-doc exists for depth (permutation test README, fixture schema), link to it in one line. Don't summarize its contents.
