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

## AWS Deployment Notes (Subdomain Approach)

### Simpler Architecture
With `metronome.jurigregg.com` subdomain:
- No CloudFront function needed
- No path rewriting required
- index.html served as default root object
- Cleaner URL

### S3 Bucket Setup
```bash
# Create bucket
aws s3 mb s3://jurigregg-metronome --region us-east-1

# Block public access (OAC will handle)
aws s3api put-public-access-block \
    --bucket jurigregg-metronome \
    --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Upload file
aws s3 cp dist/index.html s3://jurigregg-metronome/index.html \
    --content-type "text/html; charset=utf-8" \
    --cache-control "max-age=3600"
```

### CloudFront Distribution Config
Key settings for new distribution:
- **Origin**: jurigregg-metronome.s3.us-east-1.amazonaws.com
- **Origin Access Control**: Create new OAC for S3
- **Default Root Object**: index.html
- **Alternate Domain Name (CNAME)**: metronome.jurigregg.com
- **SSL Certificate**: Select existing *.jurigregg.com cert from ACM
- **Viewer Protocol Policy**: Redirect HTTP to HTTPS
- **Price Class**: Use only North America and Europe (or all edge locations)

### Find Existing Wildcard Certificate ARN
```bash
aws acm list-certificates --region us-east-1 \
    --query "CertificateSummaryList[?contains(DomainName, '*.jurigregg.com') || contains(DomainName, 'jurigregg.com')].CertificateArn" \
    --output text
```

### Route 53 DNS Record
After CloudFront distribution is created:
```bash
# Get hosted zone ID for jurigregg.com
aws route53 list-hosted-zones --query "HostedZones[?Name=='jurigregg.com.'].Id" --output text

# Create A record (alias) pointing to CloudFront
# CloudFront hosted zone ID is always: Z2FDTNDATAQYW2
```

### Cache Invalidation
After updates:
```bash
aws cloudfront create-invalidation \
    --distribution-id <NEW_DISTRIBUTION_ID> \
    --paths "/*"
```
