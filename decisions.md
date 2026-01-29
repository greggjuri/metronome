# decisions.md - Confirmed Project Decisions

This file tracks all confirmed decisions. Both Claude.ai and Claude Code should reference this file.

---

## Deployment Architecture

### S3 Storage
- **Approach**: Dedicated S3 bucket
- **Bucket name**: `jurigregg-metronome` (or similar unique name)
- **Contents**: Single `index.html` file

### CloudFront
- **Approach**: Add to EXISTING distribution
- **Existing distribution**: jurigregg.com CloudFront distribution
- **New origin**: Point to `jurigregg-metronome` S3 bucket
- **New behavior**: Path pattern `/metronome*` routes to new origin
- **Function**: CloudFront function to rewrite `/metronome` → `/index.html`

### SSL
- Using existing ACM certificate (covers jurigregg.com and *.jurigregg.com)
- No new certificate needed

---

## Design Decisions

- **Layout**: Simple, clean, functional
- **Theme**: Light/dark toggle with sensible default colors
- **No specific styling requirements**: Claude Code can use reasonable defaults

---

## Technical Stack

- Single HTML file with embedded CSS/JS
- No build tools
- No external dependencies (no CDN links, no external audio files)
- Web Audio API for timing (NEVER setInterval/setTimeout for beats)
