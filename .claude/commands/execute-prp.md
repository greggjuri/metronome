Implement a feature using the PRP (Product Requirements Prompt) file.

## Input
Read the PRP file: $ARGUMENTS

## Required Context - READ THESE FIRST
Before implementing, you MUST read:
1. The PRP file specified in $ARGUMENTS
2. `decisions.md` - Confirmed project decisions
3. `task.md` - Technical gotchas and implementation notes

## Planning Phase
Think hard before you execute. Create a comprehensive plan:

1. Read the entire PRP thoroughly
2. Understand all requirements and constraints
3. Review the validation gates - these must all pass
4. Break down complex tasks into smaller, manageable steps
5. Use the TodoWrite tool to create and track your implementation plan
6. Identify implementation patterns from existing code and task.md

## Implementation Phase
Execute each step from your plan:

1. Follow the Implementation Steps from the PRP in order
2. After each major step, verify against relevant validation gates
3. Write clean, well-documented code
4. Follow patterns specified in task.md
5. Respect constraints from decisions.md

## Validation Phase
After implementation:

1. Run through ALL validation gates from the PRP
2. Test the implementation manually if applicable
3. Fix any issues found
4. Re-validate until all gates pass

## Error Handling
If validation fails:
- Check the Error Handling section of the PRP
- Use error patterns to diagnose and fix
- Retry validation after fixes

## Output
- Place final deliverables in location specified by PRP
- Typically `dist/` for deployable files
- Ensure all Success Criteria from PRP are met

## Completion
Mark implementation complete only when:
- [ ] All Implementation Steps completed
- [ ] All Validation Gates pass
- [ ] All Success Criteria met
- [ ] Output files in correct location

Report any issues or deviations from the PRP.
