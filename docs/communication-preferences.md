# Communication & Writing Preferences

## Audience Persona
- Technical infrastructure/platform engineers who understand Terraform, Azure, and subscription migrations
- Not beginners — don't over-explain foundational concepts or use analogies ("think of it as...")
- Decision-makers who need to evaluate an approach, anticipate risks, and greenlight execution

## Tone & Style
- Direct and factual, not condescending
- Avoid heavy bolding for emphasis — use it sparingly for structure, not to highlight every key phrase
- Don't bullet-point every word with bold lead-ins — let the content speak
- State implications plainly without dramatizing or softening excessively
- No hand-holding language ("Think of it as...", "Simply put...", "In other words...")

## Content Preferences
- Lead with accessible overview; push gritty technical details (resource IDs, property comparisons, exact settings) into an appendix
- Be accurate about what Terraform actually handles vs what requires manual work — don't overstate risks that the automation already covers
- When citing numbers (e.g. retention windows, timelines), frame them in context so they don't cause unnecessary alarm
- Call out things that are already happening today (e.g. data purging) rather than implying the migration introduces new risk
- When something is queryable or accessible with caveats, state the caveats plainly (e.g. RBAC requirements) rather than using vague terms like "appropriate RBAC"

## Formatting
- Don't use emojis unless requested
- Tables are fine for comparisons
- Code blocks for CLI commands and KQL queries
- Checklist format (- [ ]) for action items the team needs to complete
