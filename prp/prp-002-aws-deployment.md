# PRP-002: AWS Deployment for Metronome (Subdomain)

## Context

- **Init file**: `init/init-002-aws-deployment.md`
- **Decisions**: `decisions.md` - New dedicated CloudFront distribution, subdomain approach
- **Technical notes**: `task.md` - AWS commands and configuration details

### Key Constraints
- Create NEW dedicated CloudFront distribution (not modify existing)
- Use subdomain: `metronome.jurigregg.com`
- Use existing wildcard ACM certificate (`*.jurigregg.com`)
- Create Route 53 DNS record pointing to CloudFront
- Use Origin Access Control (OAC) for secure S3 access
- No CloudFront function needed (simpler architecture)

### Prerequisites
- AWS CLI configured with appropriate credentials
- `dist/index.html` exists (created by PRP-001)
- Existing ACM wildcard certificate for `*.jurigregg.com` in us-east-1
- Existing Route 53 hosted zone for `jurigregg.com`

## Objective

Deploy the metronome application to AWS infrastructure accessible at:
- **https://metronome.jurigregg.com**

## Technical Approach

### Architecture Overview
```
User: https://metronome.jurigregg.com
    ↓
Route 53: metronome.jurigregg.com → CloudFront
    ↓
CloudFront Distribution (NEW dedicated)
    ↓
S3 Bucket: jurigregg-metronome/index.html
```

### Why Subdomain is Simpler
- No CloudFront function for path rewriting
- index.html served as default root object
- Isolated from other jurigregg.com infrastructure
- Easier to manage and delete later

### Components to Create
1. **S3 Bucket**: `jurigregg-metronome` with index.html
2. **Origin Access Control (OAC)**: Secure CloudFront → S3 access
3. **CloudFront Distribution**: New dedicated distribution with SSL
4. **S3 Bucket Policy**: Allow OAC to read objects
5. **Route 53 A Record**: Alias to CloudFront distribution

### Existing Resources to Use
- **ACM Certificate**: `*.jurigregg.com` wildcard cert in us-east-1
- **Route 53 Hosted Zone**: `jurigregg.com`

## Implementation Steps

### Step 1: Verify Prerequisites

Verify `dist/index.html` exists:
```bash
ls -la dist/index.html
```

Verify AWS CLI is configured:
```bash
aws sts get-caller-identity
```

### Step 2: Create S3 Bucket

Create the bucket in us-east-1 (required for CloudFront):
```bash
aws s3 mb s3://jurigregg-metronome --region us-east-1
```

**If bucket name is taken**, try alternatives like `jurigregg-metronome-app`.

Block all public access (OAC will handle access):
```bash
aws s3api put-public-access-block \
    --bucket jurigregg-metronome \
    --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

### Step 3: Upload index.html to S3

```bash
aws s3 cp dist/index.html s3://jurigregg-metronome/index.html \
    --content-type "text/html; charset=utf-8" \
    --cache-control "max-age=3600"
```

Verify upload:
```bash
aws s3 ls s3://jurigregg-metronome/
```

### Step 4: Create Origin Access Control (OAC)

```bash
aws cloudfront create-origin-access-control \
    --origin-access-control-config '{
        "Name": "metronome-s3-oac",
        "Description": "OAC for metronome S3 bucket",
        "SigningProtocol": "sigv4",
        "SigningBehavior": "always",
        "OriginAccessControlOriginType": "s3"
    }'
```

**Save the returned OAC ID** (e.g., `E2QWRUHAPOMQZL`) for Step 6.

### Step 5: Find ACM Certificate ARN

Find the wildcard certificate for `*.jurigregg.com`:
```bash
aws acm list-certificates --region us-east-1 \
    --query "CertificateSummaryList[?contains(DomainName, 'jurigregg.com')].{Domain:DomainName,ARN:CertificateArn}" \
    --output table
```

**Save the certificate ARN** for Step 6.

### Step 6: Create CloudFront Distribution

Create a distribution config file (`/tmp/cf-distribution.json`):

```json
{
    "CallerReference": "metronome-dist-001",
    "Comment": "Metronome application distribution",
    "Enabled": true,
    "DefaultRootObject": "index.html",
    "Origins": {
        "Quantity": 1,
        "Items": [
            {
                "Id": "metronome-s3-origin",
                "DomainName": "jurigregg-metronome.s3.us-east-1.amazonaws.com",
                "S3OriginConfig": {
                    "OriginAccessIdentity": ""
                },
                "OriginAccessControlId": "<OAC_ID>"
            }
        ]
    },
    "DefaultCacheBehavior": {
        "TargetOriginId": "metronome-s3-origin",
        "ViewerProtocolPolicy": "redirect-to-https",
        "AllowedMethods": {
            "Quantity": 2,
            "Items": ["GET", "HEAD"],
            "CachedMethods": {
                "Quantity": 2,
                "Items": ["GET", "HEAD"]
            }
        },
        "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
        "Compress": true
    },
    "Aliases": {
        "Quantity": 1,
        "Items": ["metronome.jurigregg.com"]
    },
    "ViewerCertificate": {
        "ACMCertificateArn": "<CERTIFICATE_ARN>",
        "SSLSupportMethod": "sni-only",
        "MinimumProtocolVersion": "TLSv1.2_2021"
    },
    "PriceClass": "PriceClass_100",
    "HttpVersion": "http2"
}
```

**Replace placeholders**:
- `<OAC_ID>`: OAC ID from Step 4
- `<CERTIFICATE_ARN>`: Certificate ARN from Step 5

**Note**: `658327ea-f89d-4fab-a63d-7e88639e58f6` is the AWS managed "CachingOptimized" policy ID.

Create the distribution:
```bash
aws cloudfront create-distribution \
    --distribution-config file:///tmp/cf-distribution.json
```

**Save from the output**:
- **Distribution ID** (e.g., `E1234567890ABC`)
- **Distribution ARN** (e.g., `arn:aws:cloudfront::123456789012:distribution/E1234567890ABC`)
- **Distribution Domain Name** (e.g., `d1234567890.cloudfront.net`)

### Step 7: Update S3 Bucket Policy

Create bucket policy file (`/tmp/bucket-policy.json`):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowCloudFrontServicePrincipal",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudfront.amazonaws.com"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::jurigregg-metronome/*",
            "Condition": {
                "StringEquals": {
                    "AWS:SourceArn": "<DISTRIBUTION_ARN>"
                }
            }
        }
    ]
}
```

**Replace** `<DISTRIBUTION_ARN>` with the ARN from Step 6.

Apply the policy:
```bash
aws s3api put-bucket-policy \
    --bucket jurigregg-metronome \
    --policy file:///tmp/bucket-policy.json
```

### Step 8: Get Route 53 Hosted Zone ID

```bash
aws route53 list-hosted-zones \
    --query "HostedZones[?Name=='jurigregg.com.'].Id" \
    --output text
```

This returns something like `/hostedzone/Z1234567890ABC`. **Save the ID portion** (e.g., `Z1234567890ABC`).

### Step 9: Create Route 53 DNS Record

Create the change batch file (`/tmp/dns-record.json`):

```json
{
    "Changes": [
        {
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "metronome.jurigregg.com",
                "Type": "A",
                "AliasTarget": {
                    "HostedZoneId": "Z2FDTNDATAQYW2",
                    "DNSName": "<CLOUDFRONT_DOMAIN>",
                    "EvaluateTargetHealth": false
                }
            }
        }
    ]
}
```

**Replace** `<CLOUDFRONT_DOMAIN>` with the CloudFront domain from Step 6 (e.g., `d1234567890.cloudfront.net`).

**Note**: `Z2FDTNDATAQYW2` is the fixed hosted zone ID for all CloudFront distributions.

Apply the DNS change:
```bash
aws route53 change-resource-record-sets \
    --hosted-zone-id <HOSTED_ZONE_ID> \
    --change-batch file:///tmp/dns-record.json
```

### Step 10: Wait for Distribution Deployment

CloudFront distributions take 5-15 minutes to deploy:
```bash
aws cloudfront wait distribution-deployed --id <DISTRIBUTION_ID>
```

Or check status manually:
```bash
aws cloudfront get-distribution --id <DISTRIBUTION_ID> \
    --query "Distribution.Status" --output text
```

Wait until status is `Deployed`.

### Step 11: Verify Deployment

Test the URL:
```bash
curl -I https://metronome.jurigregg.com
```

Expected response:
- HTTP 200 (or 301/302 redirecting to HTTPS)
- `Content-Type: text/html`
- No SSL errors

Test in browser:
- Open https://metronome.jurigregg.com
- Verify metronome UI loads
- Verify no certificate warnings
- Test start/stop and theme toggle

## Validation Gates

- [ ] **Gate 1: S3 Bucket Exists** - `aws s3 ls s3://jurigregg-metronome/` shows index.html
- [ ] **Gate 2: OAC Created** - `aws cloudfront list-origin-access-controls` shows metronome-s3-oac
- [ ] **Gate 3: Distribution Created** - `aws cloudfront list-distributions` shows new distribution
- [ ] **Gate 4: Bucket Policy Applied** - Policy allows CloudFront OAC access
- [ ] **Gate 5: DNS Record Exists** - `dig metronome.jurigregg.com` resolves to CloudFront
- [ ] **Gate 6: Distribution Deployed** - Status is "Deployed"
- [ ] **Gate 7: HTTPS Works** - `curl -I https://metronome.jurigregg.com` returns 200
- [ ] **Gate 8: Content Correct** - Browser loads metronome app correctly
- [ ] **Gate 9: SSL Valid** - No certificate warnings in browser

## Error Handling

### Common Issues

1. **Bucket name already exists**
   - S3 bucket names are globally unique
   - Try alternatives: `jurigregg-metronome-app`, `jg-metronome`
   - Update all subsequent commands with actual bucket name

2. **Certificate not found**
   - ACM certificates must be in us-east-1 for CloudFront
   - Verify wildcard cert exists: `aws acm list-certificates --region us-east-1`
   - If no wildcard, need to request one or use different domain

3. **Access Denied on S3**
   - Verify bucket policy has correct distribution ARN
   - Ensure OAC ID is correctly set in distribution config
   - Check that public access block is configured

4. **Distribution creation fails**
   - Check that certificate ARN is valid
   - Verify OAC ID exists
   - Ensure CNAME (metronome.jurigregg.com) isn't used by another distribution

5. **DNS not resolving**
   - DNS propagation takes up to 48 hours (usually minutes)
   - Verify hosted zone ID is correct
   - Check CloudFront domain name in alias target is correct

6. **SSL certificate error in browser**
   - Verify certificate covers `*.jurigregg.com` or `metronome.jurigregg.com`
   - Check certificate is in "Issued" status
   - Ensure SSLSupportMethod is "sni-only"

### Rollback Procedure

If deployment fails and needs rollback:

1. Delete Route 53 record:
   ```bash
   # Change "Action": "CREATE" to "DELETE" in dns-record.json
   aws route53 change-resource-record-sets \
       --hosted-zone-id <HOSTED_ZONE_ID> \
       --change-batch file:///tmp/dns-record.json
   ```

2. Delete CloudFront distribution (must disable first):
   ```bash
   # Get current config
   aws cloudfront get-distribution-config --id <DIST_ID> > /tmp/dist.json
   # Set Enabled: false, then update
   aws cloudfront update-distribution --id <DIST_ID> --if-match <ETAG> --distribution-config file:///tmp/disabled.json
   # Wait for deployment
   aws cloudfront wait distribution-deployed --id <DIST_ID>
   # Delete
   aws cloudfront delete-distribution --id <DIST_ID> --if-match <NEW_ETAG>
   ```

3. Delete S3 bucket:
   ```bash
   aws s3 rb s3://jurigregg-metronome --force
   ```

4. Delete OAC:
   ```bash
   aws cloudfront delete-origin-access-control --id <OAC_ID> --if-match <ETAG>
   ```

## Output

This PRP produces:
- S3 bucket `jurigregg-metronome` with `index.html`
- CloudFront distribution serving `metronome.jurigregg.com`
- Route 53 DNS record pointing to CloudFront
- Working URL: **https://metronome.jurigregg.com**

No files are created in the repository - all changes are AWS infrastructure.

## Success Criteria

The deployment is complete when:

1. S3 bucket `jurigregg-metronome` contains `index.html`
2. CloudFront distribution is created and status is "Deployed"
3. Route 53 A record exists for `metronome.jurigregg.com`
4. **https://metronome.jurigregg.com** loads the metronome app
5. No SSL/certificate warnings in browser
6. Cache-Control header is present (max-age=3600)
7. Theme toggle and audio playback work correctly

---

## Confidence Score: 8/10

**Areas of high confidence:**
- S3 bucket creation and file upload (straightforward CLI commands)
- OAC creation (simple, well-documented)
- Route 53 DNS record (standard alias pattern)
- Overall architecture is simpler than path-based approach

**Areas of minor uncertainty:**

1. **CloudFront distribution config (-1 point)**: The JSON config for creating a distribution is complex. Minor typos or missing fields will cause failures. However, the config provided is complete and tested patterns.

2. **Certificate ARN lookup (-1 point)**: Depends on existing wildcard certificate. If it doesn't exist or has a different domain pattern, will need adjustment.

**Improvements over previous PRP:**
- No CloudFront function complexity
- No need to modify existing distribution
- Cleaner, more isolated architecture
- Fewer steps with interdependencies

**This PRP can be executed with high confidence** given that the wildcard certificate exists and AWS CLI is properly configured.
