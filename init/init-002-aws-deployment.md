# init-002: Deploy Metronome to AWS

> **See also:** `decisions.md` for confirmed architecture, `task.md` for CloudFront function code.

## OBJECTIVE
Deploy `dist/index.html` to AWS so it's accessible at https://jurigregg.com/metronome

## DEPLOYMENT ARCHITECTURE (from decisions.md)

### S3
- Create dedicated bucket: `jurigregg-metronome`
- Enable static website hosting
- Upload `dist/index.html` as `index.html`
- Set content-type: `text/html`
- Set cache-control: `max-age=3600`

### CloudFront
- Add NEW origin to EXISTING jurigregg.com distribution
- Origin domain: `jurigregg-metronome.s3.amazonaws.com` (or S3 website endpoint)
- Create Origin Access Control (OAC) for secure S3 access
- Add cache behavior for path pattern `/metronome*`
- Viewer protocol: Redirect HTTP to HTTPS

### Path Routing
CloudFront function to handle:
- `/metronome` → serves `index.html`
- `/metronome/` → serves `index.html`

Function code is in `task.md`.

## IMPLEMENTATION STEPS

### Step 1: Create S3 Bucket
```bash
aws s3 mb s3://jurigregg-metronome --region us-east-1
```

### Step 2: Upload File
```bash
aws s3 cp dist/index.html s3://jurigregg-metronome/index.html \
    --content-type "text/html" \
    --cache-control "max-age=3600"
```

### Step 3: Create Bucket Policy
Allow CloudFront OAC to read from bucket. Policy needs:
- Principal: CloudFront service
- Action: s3:GetObject
- Resource: arn:aws:s3:::jurigregg-metronome/*
- Condition: StringEquals for CloudFront distribution ARN

### Step 4: Get Existing CloudFront Distribution ID
```bash
aws cloudfront list-distributions --query "DistributionList.Items[?Aliases.Items[?contains(@, 'jurigregg.com')]].Id" --output text
```

### Step 5: Create CloudFront Function
Create function for path rewriting (code in task.md).

### Step 6: Update CloudFront Distribution
Add:
- New S3 origin with OAC
- New cache behavior for `/metronome*` path pattern
- Associate CloudFront function with viewer-request

### Step 7: Invalidate Cache
```bash
aws cloudfront create-invalidation --distribution-id <ID> --paths "/metronome*"
```

## REQUIRED INFORMATION
Claude Code will need to retrieve:
- Existing CloudFront distribution ID for jurigregg.com
- CloudFront distribution ARN (for bucket policy)

## SUCCESS CRITERIA
- [ ] S3 bucket `jurigregg-metronome` exists with index.html
- [ ] CloudFront distribution has new origin pointing to bucket
- [ ] Cache behavior routes `/metronome*` to new origin
- [ ] CloudFront function handles path rewriting
- [ ] https://jurigregg.com/metronome loads the metronome app
- [ ] https://jurigregg.com/metronome/ also works
- [ ] No certificate warnings (uses existing SSL)

## NOTES
- Bucket name must be globally unique - if `jurigregg-metronome` is taken, use variant
- CloudFront changes can take 5-15 minutes to propagate
- User may need to provide distribution ID if auto-detection fails
