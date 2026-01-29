# init-001: Online Metronome Project Overview

> **See also:** `decisions.md` for confirmed architecture decisions, `task.md` for technical implementation notes.

## PROJECT SUMMARY
Build a simple, clean online metronome application deployed to jurigregg.com/metronome on AWS infrastructure.

## DEPLOYMENT TARGET
- **URL**: https://jurigregg.com/metronome
- **Infrastructure**: AWS
- **Storage**: Dedicated S3 bucket (e.g., `jurigregg-metronome`)
- **CloudFront**: Add behavior to EXISTING distribution for jurigregg.com
- **SSL**: Existing certificates for jurigregg.com and *.jurigregg.com

## FUNCTIONAL REQUIREMENTS

### BPM Input
- Text/number input field displaying current BPM
- On-screen number pad (0-9, backspace/clear)
- Physical keyboard input support (numbers, backspace)
- BPM Range: 30 minimum, 240 maximum
- Invalid entries clamped to valid range on start

### Beat Display
- Visual beat indicator cycling 1 → 2 → 3 → 4 → 1...
- Clear visual highlight on current beat
- Resets to beat 1 when stopped

### Audio
- Simple click sound on every beat
- Consistent timing (Web Audio API recommended for precision)

### Controls
- **Start/Stop Button**: On-screen toggle button
- **Spacebar**: Keyboard shortcut for start/stop
- Clear visual state indication (playing vs stopped)

### Theme
- Light/Dark mode toggle
- Persistent theme preference (localStorage)
- Smooth transition between themes

## TECHNICAL STACK
- Single HTML file with embedded CSS/JS
- Web Audio API for precise timing
- CSS custom properties for theming
- No external dependencies
- No backend required (static site)

## AWS DEPLOYMENT NOTES
- S3 bucket with static website hosting enabled
- CloudFront distribution for HTTPS
- Route 53 for DNS (jurigregg.com/metronome path)
- Existing SSL cert via ACM

## SUCCESS CRITERIA
- [ ] Metronome plays accurate beats at specified BPM
- [ ] BPM adjustable via number pad and keyboard
- [ ] Visual beat counter works correctly (1-4 cycle)
- [ ] Spacebar starts/stops metronome
- [ ] On-screen button starts/stops metronome
- [ ] Light/dark mode toggle functions
- [ ] Theme preference persists across sessions
- [ ] Deployed and accessible at jurigregg.com/metronome
