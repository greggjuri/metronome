# task.md - Current Task Technical Notes

This file contains technical considerations, gotchas, and implementation notes for the current task. Both Claude.ai and Claude Code should reference this file.

---

## Current Task: Online Metronome

### Audio Timing - CRITICAL

**DO NOT USE setInterval/setTimeout for audio scheduling - will drift and be inconsistent.**

Use Web Audio API scheduler pattern:
1. Create an AudioContext
2. Use `audioContext.currentTime` for precise scheduling
3. Schedule beats slightly ahead (lookahead pattern)
4. Use a lookahead window (e.g., 25ms) and scheduling interval (e.g., 100ms)

Reference: https://www.html5rocks.com/en/tutorials/audio/scheduling/

### Click Sound Generation

Use oscillator-based approach (no external files needed):
```javascript
function playClick(time) {
    const osc = audioContext.createOscillator();
    const gain = audioContext.createGain();
    osc.connect(gain);
    gain.connect(audioContext.destination);
    
    osc.frequency.value = 1000; // Hz
    gain.gain.setValueAtTime(1, time);
    gain.gain.exponentialRampToValueAtTime(0.001, time + 0.05);
    
    osc.start(time);
    osc.stop(time + 0.05);
}
```

### Keyboard Handling

**Spacebar** - prevent page scrolling:
```javascript
document.addEventListener('keydown', (e) => {
    if (e.code === 'Space') {
        e.preventDefault();
        toggleMetronome();
    }
});
```

**Number input** - handle both main keyboard and numpad.

### Theme Persistence

Load theme BEFORE DOM renders to prevent flash:
```javascript
const theme = localStorage.getItem('metronome-theme') || 'light';
document.documentElement.setAttribute('data-theme', theme);
```

### BPM Validation

- On input: allow any typing
- On start: clamp to 30-240 range
- If empty: default to 120 BPM

### Mobile Considerations

**AudioContext on iOS** - requires user interaction:
```javascript
document.addEventListener('click', () => {
    if (audioContext.state === 'suspended') {
        audioContext.resume();
    }
}, { once: true });
```

**Touch targets** - minimum 44x44px for buttons.

### Accessibility

- Focus states visible for keyboard navigation
- ARIA labels for buttons
- Sufficient color contrast in both themes

---

## AWS Deployment Notes

### CloudFront Function for Path Routing
```javascript
function handler(event) {
    var request = event.request;
    var uri = request.uri;
    
    if (uri === '/metronome' || uri === '/metronome/') {
        request.uri = '/index.html';
    } else if (uri.startsWith('/metronome/')) {
        request.uri = uri.replace('/metronome', '');
    }
    
    return request;
}
```

### Cache Invalidation
After deployment:
```bash
aws cloudfront create-invalidation \
    --distribution-id [DISTRIBUTION_ID] \
    --paths "/metronome/*"
```
