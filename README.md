# Online Metronome - Context Engineering Project

## Workflow

1. **Claude.ai + Human** brainstorm and write one init file at a time (referencing `decisions.md` and `task.md`)
2. **Claude Code** generates PRP: `/generate-prp init/init-001-project-overview.md`
3. **Claude.ai + Human** review generated PRP
4. **Claude Code** executes approved PRP: `/execute-prp prp/[generated-file].md`

## Project Structure

```
metronome-project/
├── .claude/                 # Claude Code configuration
│   ├── commands/
│   │   ├── generate-prp.md  # /generate-prp command
│   │   └── execute-prp.md   # /execute-prp command
│   └── settings.local.json  # Permissions
├── CLAUDE.md            # Project rules for Code and Claude.ai
├── README.md            # This file
├── decisions.md         # Confirmed decisions (both reference this)
├── task.md              # Current task technical notes (both reference this)
├── init/                # Init files (one at a time)
│   └── init-001-*.md    # Current feature request
├── prp/                 # PRPs (Code generates, we review)
├── examples/            # Reference code (if needed)
└── dist/                # Output directory
```

## Key Files

| File | Purpose | Who Uses |
|------|---------|----------|
| `decisions.md` | Confirmed architecture, design, tech decisions | Claude.ai + Code |
| `task.md` | Technical gotchas, implementation notes | Claude.ai + Code |
| `init/init-###-*.md` | Feature request (one at a time) | Code generates PRP from this |

## Features (Current Task)
- BPM input (30-240 range)
- On-screen number pad
- Physical keyboard support
- Visual beat counter (1-4)
- Click sound on every beat
- Spacebar start/stop
- On-screen start/stop button
- Light/dark theme toggle

## Deployment Target
- URL: https://jurigregg.com/metronome
- Dedicated S3 bucket + existing CloudFront distribution
