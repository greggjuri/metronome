# init-002: Deploy Metronome to AWS (Subdomain)

> **See also:** `decisions.md` for architecture, `task.md` for AWS commands.

## OBJECTIVE
Deploy `dist/index.html` to AWS at **https://metronome.jurigregg.com**

## DEPLOYMENT ARCHITECTURE

### Overview
```
User: https://metronome.jurigregg.com
    ↓
Route 53: metronome.jurigregg.com → CloudFront
    ↓
CloudFront Distribution (NEW, dedicated)
    ↓
S3 Bucket: jurigregg-metronome/index.html
```

### Components to Create
1. **S3 Bucket**: `jurigregg-metronome`
2. **Origin Access Control (OAC)**: Secure CloudFront → S3 access
3. **CloudFront Distribution**: New dedicated distribution
4. **Route 53 Record**: metronome.jurigregg.com → CloudFront

### Existing Resources to Use
- **ACM Certificate**: *.jurigregg.com wildcard cert (us-east-1)
- **Route 53 Hosted Zone**: jurigregg.com

## IMPLEMENTATION STEPS

### Step 1: Create S3 Bucket
- Bucket name: `jurigregg-metronome`
- Region: us-east-1
- Block all public access (OAC handles it)

### Step 2: Upload index.html
- Source: `dist/index.html`
- Content-Type: text/html; charset=utf-8
- Cache-Control: max-age=3600

### Step 3: Create Origin Access Control (OAC)
- Name: metronome-s3-oac
- Signing: sigv4, always
- Type: s3

### Step 4: Find ACM Certificate ARN
- Look for *.jurigregg.com or jurigregg.com cert in us-east-1
- This will be used for CloudFront SSL

### Step 5: Create CloudFront Distribution
- Origin: jurigregg-metronome.s3.us-east-1.amazonaws.com
- Origin Access: Use OAC from Step 3
- Default Root Object: index.html
- Alternate Domain (CNAME): metronome.jurigregg.com
- SSL Certificate: ACM cert from Step 4
- Viewer Protocol: Redirect HTTP to HTTPS
- Cache Policy: CachingOptimized

### Step 6: Update S3 Bucket Policy
- Allow CloudFront OAC to read objects
- Use distribution ARN in condition

### Step 7: Create Route 53 DNS Record
- Record name: metronome.jurigregg.com
- Type: A (Alias)
- Alias target: CloudFront distribution domain
- CloudFront Hosted Zone ID: Z2FDTNDATAQYW2 (always this for CloudFront)

### Step 8: Wait for Distribution Deployment
- CloudFront takes 5-15 minutes to deploy
- Wait for status: Deployed

### Step 9: Verify
- Test https://metronome.jurigregg.com loads
- Verify SSL certificate (no warnings)
- Test functionality

## SUCCESS CRITERIA
- [ ] S3 bucket exists with index.html
- [ ] CloudFront distribution created and deployed
- [ ] SSL certificate attached (no browser warnings)
- [ ] DNS record created in Route 53
- [ ] https://metronome.jurigregg.com loads metronome app
- [ ] Theme toggle and audio work correctly

## ADVANTAGES OF SUBDOMAIN APPROACH
- No CloudFront function needed (simpler)
- No path rewriting complexity
- Clean URL: metronome.jurigregg.com
- Isolated from other jurigregg.com infrastructure
- Easier to manage/delete later
