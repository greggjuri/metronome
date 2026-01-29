# PRP-001: Online Metronome Application

## Context

- **Init file**: `init/init-001-project-overview.md`
- **Decisions**: `decisions.md` - Single HTML file, no dependencies, Web Audio API
- **Technical notes**: `task.md` - Critical timing patterns, accessibility requirements

### Key Constraints
- Single HTML file with embedded CSS and JavaScript
- No external dependencies (no CDN links, no external audio files)
- Web Audio API for timing (NEVER setInterval/setTimeout for beat scheduling)
- Light/dark theme with localStorage persistence
- Accessible: focus states, ARIA labels, minimum 44x44px touch targets

## Objective

Build a fully functional online metronome application as a single `index.html` file that:
- Plays accurate click sounds at user-specified BPM (30-240 range)
- Displays a visual 4-beat counter
- Supports both on-screen number pad and physical keyboard input
- Toggles between light and dark themes with persistence
- Works reliably on desktop and mobile browsers

Output: `dist/index.html`

## Technical Approach

### Architecture
Single HTML file with three embedded sections:
1. `<style>` - CSS with custom properties for theming
2. `<body>` - Semantic HTML structure
3. `<script>` - JavaScript at end of body

### Audio Timing (Critical)
Use the Web Audio API lookahead scheduler pattern:
- `AudioContext.currentTime` for precise scheduling
- Lookahead window: 25ms
- Scheduler interval: 100ms (using setInterval ONLY for scheduler loop, not beat timing)
- Schedule beats ahead of time into the audio context's timeline

### Click Sound
Oscillator-based generation (no external files):
- Frequency: 1000Hz for regular beats, 1200Hz for beat 1 (accent)
- Duration: 50ms with exponential decay
- Creates fresh oscillator/gain nodes per click

### Theming
CSS custom properties with `data-theme` attribute on `<html>`:
- Load theme from localStorage BEFORE DOM renders (in `<head>`)
- Toggle updates attribute and localStorage
- Smooth 200ms transition on theme change

## Implementation Steps

### Step 1: HTML Structure

Create the document with this structure:

```html
<!DOCTYPE html>
<html lang="en" data-theme="light">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Metronome</title>
    <!-- Theme initialization script (prevents flash) -->
    <script>
        (function() {
            const theme = localStorage.getItem('metronome-theme') || 'light';
            document.documentElement.setAttribute('data-theme', theme);
        })();
    </script>
    <style>/* CSS here */</style>
</head>
<body>
    <div class="container">
        <header>
            <h1>Metronome</h1>
            <button id="theme-toggle" aria-label="Toggle dark mode">
                <span class="icon-light">☀️</span>
                <span class="icon-dark">🌙</span>
            </button>
        </header>

        <main>
            <!-- BPM Display -->
            <div class="bpm-display">
                <input type="text" id="bpm-input" value="120"
                       inputmode="numeric" pattern="[0-9]*"
                       aria-label="Beats per minute">
                <span class="bpm-label">BPM</span>
            </div>

            <!-- Beat Indicator -->
            <div class="beat-display" aria-live="polite" aria-label="Beat counter">
                <div class="beat" data-beat="1">1</div>
                <div class="beat" data-beat="2">2</div>
                <div class="beat" data-beat="3">3</div>
                <div class="beat" data-beat="4">4</div>
            </div>

            <!-- Start/Stop Button -->
            <button id="start-stop" class="start-stop-btn" aria-label="Start metronome">
                Start
            </button>

            <!-- Number Pad -->
            <div class="numpad" role="group" aria-label="Number pad">
                <button data-num="1">1</button>
                <button data-num="2">2</button>
                <button data-num="3">3</button>
                <button data-num="4">4</button>
                <button data-num="5">5</button>
                <button data-num="6">6</button>
                <button data-num="7">7</button>
                <button data-num="8">8</button>
                <button data-num="9">9</button>
                <button data-num="clear" aria-label="Clear">C</button>
                <button data-num="0">0</button>
                <button data-num="backspace" aria-label="Backspace">⌫</button>
            </div>
        </main>
    </div>

    <script>/* JavaScript here */</script>
</body>
</html>
```

### Step 2: CSS Styles

Implement CSS with these requirements:

```css
/* CSS Custom Properties for Theming */
:root {
    --bg-primary: #ffffff;
    --bg-secondary: #f5f5f5;
    --text-primary: #1a1a1a;
    --text-secondary: #666666;
    --accent: #2563eb;
    --accent-hover: #1d4ed8;
    --beat-inactive: #e5e5e5;
    --beat-active: #2563eb;
    --border: #d4d4d4;
    --shadow: rgba(0, 0, 0, 0.1);
    --transition-speed: 200ms;
}

[data-theme="dark"] {
    --bg-primary: #1a1a1a;
    --bg-secondary: #262626;
    --text-primary: #f5f5f5;
    --text-secondary: #a3a3a3;
    --accent: #3b82f6;
    --accent-hover: #60a5fa;
    --beat-inactive: #404040;
    --beat-active: #3b82f6;
    --border: #404040;
    --shadow: rgba(0, 0, 0, 0.3);
}

/* Apply transitions for smooth theme change */
body, .container, .beat, button, input {
    transition: background-color var(--transition-speed),
                color var(--transition-speed),
                border-color var(--transition-speed);
}
```

Key styling requirements:
- **Container**: Max-width 400px, centered, padding
- **BPM Input**: Large font (3rem+), centered, no spinner arrows
- **Beat indicators**: 4 circles/squares in a row, minimum 48px each
- **Active beat**: Highlighted with accent color, subtle scale transform
- **Start/Stop button**: Large (minimum 60px height), full width, clear state colors
- **Number pad**: 3x4 grid, minimum 44x44px buttons
- **Focus states**: Visible outline (2px solid accent) for keyboard navigation
- **Theme toggle**: Fixed position top-right, clear icons for each mode

### Step 3: JavaScript - Audio System

```javascript
// Audio system setup
let audioContext = null;
let isPlaying = false;
let currentBeat = 0;
let nextNoteTime = 0.0;
let schedulerTimerId = null;

// Timing constants
const LOOKAHEAD = 25.0;        // ms - how far ahead to schedule
const SCHEDULE_AHEAD = 0.1;    // seconds - how far ahead to schedule audio
const SCHEDULER_INTERVAL = 25; // ms - how often to call scheduler

// Initialize AudioContext on first user interaction
function initAudio() {
    if (!audioContext) {
        audioContext = new (window.AudioContext || window.webkitAudioContext)();
    }
    // Resume if suspended (iOS requirement)
    if (audioContext.state === 'suspended') {
        audioContext.resume();
    }
}

// Play a click at the specified time
function playClick(time, isAccent) {
    const osc = audioContext.createOscillator();
    const gain = audioContext.createGain();

    osc.connect(gain);
    gain.connect(audioContext.destination);

    // Higher pitch for beat 1 (accent)
    osc.frequency.value = isAccent ? 1200 : 1000;
    osc.type = 'sine';

    // Quick attack, exponential decay
    gain.gain.setValueAtTime(1, time);
    gain.gain.exponentialRampToValueAtTime(0.001, time + 0.05);

    osc.start(time);
    osc.stop(time + 0.05);
}

// Schedule upcoming notes
function scheduler() {
    const bpm = getCurrentBPM();
    const secondsPerBeat = 60.0 / bpm;

    // Schedule all notes that fall within the lookahead window
    while (nextNoteTime < audioContext.currentTime + SCHEDULE_AHEAD) {
        // Schedule this beat
        const isAccent = currentBeat === 0;
        playClick(nextNoteTime, isAccent);

        // Schedule visual update (use setTimeout for UI, it's okay here)
        const beatToShow = currentBeat;
        const timeUntilBeat = (nextNoteTime - audioContext.currentTime) * 1000;
        setTimeout(() => updateBeatDisplay(beatToShow), Math.max(0, timeUntilBeat));

        // Advance to next beat
        nextNoteTime += secondsPerBeat;
        currentBeat = (currentBeat + 1) % 4;
    }
}

// Get current BPM with validation
function getCurrentBPM() {
    let bpm = parseInt(bpmInput.value, 10);
    if (isNaN(bpm) || bpm < 30) bpm = 30;
    if (bpm > 240) bpm = 240;
    return bpm;
}
```

### Step 4: JavaScript - Control Logic

```javascript
// DOM elements
const bpmInput = document.getElementById('bpm-input');
const startStopBtn = document.getElementById('start-stop');
const beats = document.querySelectorAll('.beat');
const themeToggle = document.getElementById('theme-toggle');

// Start the metronome
function start() {
    initAudio();

    // Validate and clamp BPM
    let bpm = parseInt(bpmInput.value, 10);
    if (isNaN(bpm) || bpm === 0) bpm = 120;
    bpm = Math.max(30, Math.min(240, bpm));
    bpmInput.value = bpm;

    isPlaying = true;
    currentBeat = 0;
    nextNoteTime = audioContext.currentTime;

    schedulerTimerId = setInterval(scheduler, SCHEDULER_INTERVAL);

    startStopBtn.textContent = 'Stop';
    startStopBtn.setAttribute('aria-label', 'Stop metronome');
    startStopBtn.classList.add('playing');
}

// Stop the metronome
function stop() {
    isPlaying = false;

    if (schedulerTimerId) {
        clearInterval(schedulerTimerId);
        schedulerTimerId = null;
    }

    // Reset beat display to beat 1
    resetBeatDisplay();

    startStopBtn.textContent = 'Start';
    startStopBtn.setAttribute('aria-label', 'Start metronome');
    startStopBtn.classList.remove('playing');
}

// Toggle start/stop
function toggle() {
    if (isPlaying) {
        stop();
    } else {
        start();
    }
}

// Update visual beat display
function updateBeatDisplay(beatIndex) {
    beats.forEach((beat, i) => {
        beat.classList.toggle('active', i === beatIndex);
    });
}

// Reset beat display
function resetBeatDisplay() {
    beats.forEach(beat => beat.classList.remove('active'));
}
```

### Step 5: JavaScript - Input Handling

```javascript
// Number pad handling
document.querySelectorAll('.numpad button').forEach(btn => {
    btn.addEventListener('click', () => {
        initAudio(); // Init on any interaction
        const num = btn.dataset.num;

        if (num === 'clear') {
            bpmInput.value = '';
        } else if (num === 'backspace') {
            bpmInput.value = bpmInput.value.slice(0, -1);
        } else {
            // Append digit (limit to 3 digits for BPM)
            if (bpmInput.value.length < 3) {
                bpmInput.value += num;
            }
        }

        bpmInput.focus();
    });
});

// Keyboard handling
document.addEventListener('keydown', (e) => {
    // Spacebar - toggle metronome
    if (e.code === 'Space') {
        e.preventDefault(); // Prevent page scroll
        toggle();
        return;
    }

    // Only handle number keys when not focused on input
    // (input handles its own keystrokes)
    if (document.activeElement === bpmInput) return;

    // Number keys (main keyboard and numpad)
    if (e.key >= '0' && e.key <= '9') {
        if (bpmInput.value.length < 3) {
            bpmInput.value += e.key;
        }
    }

    // Backspace
    if (e.key === 'Backspace') {
        bpmInput.value = bpmInput.value.slice(0, -1);
    }
});

// BPM input - filter non-numeric input
bpmInput.addEventListener('input', () => {
    bpmInput.value = bpmInput.value.replace(/[^0-9]/g, '').slice(0, 3);
});

// Start/Stop button
startStopBtn.addEventListener('click', toggle);
```

### Step 6: JavaScript - Theme Toggle

```javascript
// Theme toggle
function setTheme(theme) {
    document.documentElement.setAttribute('data-theme', theme);
    localStorage.setItem('metronome-theme', theme);
}

function toggleTheme() {
    const current = document.documentElement.getAttribute('data-theme');
    const next = current === 'dark' ? 'light' : 'dark';
    setTheme(next);
}

themeToggle.addEventListener('click', toggleTheme);

// Update theme toggle button state based on current theme
function updateThemeToggleVisual() {
    const theme = document.documentElement.getAttribute('data-theme');
    // CSS handles showing/hiding icons based on data-theme
}
```

### Step 7: Final Assembly

Combine all sections into a single `dist/index.html` file:

1. Start with HTML doctype and `<html>` with initial theme script in `<head>`
2. Add complete CSS in `<style>` tag
3. Add HTML body structure
4. Add complete JavaScript in `<script>` tag at end of body
5. Ensure all IDs and classes match between HTML and JavaScript

## Validation Gates

- [ ] **Gate 1: File Structure** - Single HTML file exists at `dist/index.html`, no external dependencies
- [ ] **Gate 2: Audio Plays** - Opening file in browser and clicking Start produces audible clicks
- [ ] **Gate 3: Timing Accuracy** - Set to 60 BPM, beats align with a second timer (1 beat per second)
- [ ] **Gate 4: BPM Changes** - Changing BPM via numpad/keyboard reflects in playback speed
- [ ] **Gate 5: Beat Display** - Visual counter cycles 1→2→3→4→1 in sync with audio
- [ ] **Gate 6: Start/Stop** - Button and spacebar both toggle playback; display resets on stop
- [ ] **Gate 7: Theme Toggle** - Light/dark toggle works; preference persists after page reload
- [ ] **Gate 8: Mobile Ready** - Works on mobile; no audio issues on iOS Safari after first tap
- [ ] **Gate 9: Accessibility** - Keyboard navigation works; focus states visible; ARIA labels present

## Error Handling

### Common Issues

1. **AudioContext not starting on iOS**
   - Solution: Initialize AudioContext and call `resume()` on first user interaction
   - Code pattern provided in Step 3

2. **Timing drift at high BPM**
   - Solution: Use lookahead scheduler pattern, never schedule beats with setTimeout
   - Schedule beats into the future using audioContext.currentTime

3. **Flash of wrong theme on load**
   - Solution: Inline script in `<head>` sets theme BEFORE CSS loads
   - Script must be synchronous (no defer/async)

4. **Number input allows invalid characters**
   - Solution: Filter input to digits only, limit to 3 characters
   - Validate and clamp on start, not during typing

5. **Spacebar scrolls page**
   - Solution: Call `e.preventDefault()` in keydown handler for Space

## Output

**File to create**: `dist/index.html`

This single file contains all HTML, CSS, and JavaScript needed for the metronome application.

## Success Criteria

The implementation is complete when:

1. A single `dist/index.html` file exists with no external dependencies
2. Opening the file in a modern browser (Chrome, Firefox, Safari, Edge) shows the metronome UI
3. Clicking Start plays accurate click sounds at the displayed BPM
4. BPM can be changed via on-screen numpad and physical keyboard
5. Visual beat indicator (1-4) cycles in sync with audio
6. Spacebar toggles start/stop
7. Theme toggle switches between light and dark modes
8. Theme preference persists across browser sessions
9. No console errors during normal operation
10. Interface is usable on mobile devices with touch

---

## Confidence Score: 9/10

**Areas of high confidence:**
- HTML structure and CSS theming (well-defined patterns)
- Web Audio API scheduler (exact code patterns provided in task.md)
- Input handling (straightforward DOM events)
- Theme persistence (simple localStorage pattern)

**Minor uncertainty:**
- Visual beat indicator timing synchronization (-1 point): Using setTimeout for UI updates introduces slight visual lag relative to audio. This is acceptable since human perception won't notice <25ms delay, but perfect sync requires requestAnimationFrame with audio time comparison. The provided approach is the pragmatic choice.

**No uncertainty about:**
- The core audio scheduling will be accurate
- All features described can be implemented as specified
- Single-file delivery is straightforward

This PRP provides comprehensive implementation details sufficient for one-pass success.
