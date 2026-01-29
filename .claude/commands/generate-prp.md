Generate a comprehensive PRP (Product Requirements Prompt) for implementing a feature.

## Input
Read the init file: $ARGUMENTS

## Required Context - READ THESE FIRST
Before generating the PRP, you MUST read:
1. `decisions.md` - All confirmed project decisions (architecture, design, tech stack)
2. `task.md` - Technical considerations, gotchas, and implementation notes
3. The init file specified in $ARGUMENTS

## Research Phase
1. Read and understand the feature requirements from the init file
2. Review decisions.md for architectural constraints
3. Study task.md for technical gotchas and code patterns
4. Search codebase for existing patterns to follow
5. If needed, use web search for documentation on APIs/libraries mentioned

## *** CRITICAL ***
*** AFTER RESEARCHING, ULTRATHINK ABOUT THE PRP ***
*** PLAN YOUR APPROACH BEFORE WRITING ***

## PRP Structure
Generate a PRP file in `prp/` directory with this structure:

```markdown
# PRP: [Feature Name]

## Context
- Reference to init file
- Reference to decisions.md and task.md
- Summary of key constraints and decisions

## Objective
Clear statement of what will be built

## Technical Approach
- Architecture decisions
- Key technologies/APIs to use
- Patterns to follow from task.md

## Implementation Steps
Numbered steps with clear deliverables:
1. Step one...
2. Step two...
(Include code snippets or pseudocode where helpful)

## Validation Gates
Checkpoints to verify implementation:
- [ ] Gate 1: Description
- [ ] Gate 2: Description
...

## Error Handling
Common issues and how to address them

## Output
- Files to create
- Location for final output

## Success Criteria
How to know the implementation is complete
```

## Output
Save the PRP to: `prp/prp-[number]-[feature-name].md`

Use the next available number in sequence.

## Confidence Score
At the end, score this PRP 1-10 on likelihood of one-pass implementation success.
Explain any areas of uncertainty.

Remember: The goal is one-pass implementation success through comprehensive context.
The executing agent only gets the PRP content and codebase access - include everything needed.
