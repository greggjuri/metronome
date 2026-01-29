# Online Metronome

A simple, clean online metronome application.

**Live:** https://metronome.jurigregg.com

## Features

- **BPM Control**: 30-240 BPM range with on-screen number pad and keyboard input
- **Visual Beat Counter**: 4-beat display with active beat highlighting
- **Precise Timing**: Web Audio API scheduler for hardware-accurate beats
- **Start/Stop**: On-screen button or spacebar
- **Theme Toggle**: Light/dark mode with localStorage persistence
- **Mobile Ready**: Touch-friendly, works on iOS and Android

## Tech Stack

- Single HTML file (~580 lines, ~18KB)
- No external dependencies
- Web Audio API for timing (not setInterval)
- CSS custom properties for theming
- Vanilla JavaScript

## Infrastructure

- **Hosting**: AWS S3 + CloudFront
- **Domain**: metronome.jurigregg.com
- **SSL**: ACM wildcard certificate (*.jurigregg.com)
- **DNS**: Route 53

## Project Structure

```
metronome-project/
├── .claude/                 # Claude Code commands
│   ├── commands/
│   │   ├── generate-prp.md  # /generate-prp
│   │   └── execute-prp.md   # /execute-prp
│   └── settings.local.json
├── CLAUDE.md                # Project rules
├── decisions.md             # Architecture decisions
├── task.md                  # Technical notes
├── init/                    # Feature requests
│   ├── init-001-project-overview.md
│   └── init-002-aws-deployment.md
├── prp/                     # Generated PRPs
│   ├── prp-001-online-metronome.md
│   └── prp-002-aws-deployment.md
├── dist/
│   └── index.html           # The app
└── README.md
```

## Development Workflow

This project uses context engineering with Claude Code:

1. Write init file describing the feature (`init/init-###-*.md`)
2. Generate PRP: `/generate-prp init/init-###-*.md`
3. Review PRP with Claude.ai
4. Execute: `/execute-prp prp/prp-###-*.md`

Reference `decisions.md` and `task.md` for project context.

## Updating the App

1. Modify `dist/index.html`
2. Upload to S3:
   ```bash
   aws s3 cp dist/index.html s3://<BUCKET_NAME>/index.html \
       --content-type "text/html; charset=utf-8" \
       --cache-control "max-age=3600"
   ```
3. Invalidate CloudFront cache:
   ```bash
   aws cloudfront create-invalidation \
       --distribution-id <DISTRIBUTION_ID> \
       --paths "/*"
   ```

## Notes

- iOS: Disable silent mode switch for audio to play
- Audio timing uses `audioContext.currentTime` (hardware clock), independent of refresh rate
