# PRP-002: AWS Deployment for Metronome

## Context

- **Init file**: `init/init-002-aws-deployment.md`
- **Decisions**: `decisions.md` - Use existing CloudFront distribution, dedicated S3 bucket
- **Technical notes**: `task.md` - CloudFront function code for path routing

### Key Constraints
- Deploy to EXISTING CloudFront distribution for jurigregg.com
- Create NEW S3 bucket for metronome content
- Use Origin Access Control (OAC) for secure S3 access
- CloudFront function handles `/metronome` → `/index.html` rewriting
- Use existing ACM certificate (no new SSL setup needed)
- Final URL: https://jurigregg.com/metronome

### Prerequisites
- AWS CLI configured with appropriate credentials
- `dist/index.html` exists (created by PRP-001)
- Existing CloudFront distribution serving jurigregg.com

## Objective

Deploy the metronome application to AWS infrastructure so it is accessible at:
- https://jurigregg.com/metronome
- https://jurigregg.com/metronome/

Both URLs should serve the same `index.html` content.

## Technical Approach

### Architecture Overview
```
User Request: https://jurigregg.com/metronome
    ↓
CloudFront Distribution (existing)
    ↓
Cache Behavior: /metronome* → S3 Origin
    ↓
CloudFront Function: /metronome → /index.html
    ↓
S3 Bucket: jurigregg-metronome/index.html
```

### Components to Create/Configure
1. **S3 Bucket**: `jurigregg-metronome` with index.html
2. **Origin Access Control (OAC)**: Secure access from CloudFront to S3
3. **S3 Bucket Policy**: Allow OAC to read objects
4. **CloudFront Function**: Path rewriting for /metronome routes
5. **CloudFront Origin**: New origin pointing to S3 bucket
6. **CloudFront Cache Behavior**: Route /metronome* to new origin

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

Create the bucket in us-east-1 (required for CloudFront integration):
```bash
aws s3 mb s3://jurigregg-metronome --region us-east-1
```

**If bucket name is taken**, try alternatives:
- `jurigregg-metronome-app`
- `jg-metronome`

Block all public access (CloudFront OAC will handle access):
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

### Step 4: Get Existing CloudFront Distribution ID

Find the distribution serving jurigregg.com:
```bash
aws cloudfront list-distributions \
    --query "DistributionList.Items[?Aliases.Items[?contains(@, 'jurigregg.com')]].[Id,DomainName,Aliases.Items[0]]" \
    --output table
```

Store the distribution ID for subsequent steps (e.g., `E1234567890ABC`).

If no distribution is found, ask the user to provide the distribution ID.

### Step 5: Create Origin Access Control (OAC)

Create OAC for S3 access:
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

Save the returned OAC ID (e.g., `E2QWRUHAPOMQZL`).

### Step 6: Create S3 Bucket Policy

Get the CloudFront distribution ARN:
```bash
aws cloudfront get-distribution --id <DISTRIBUTION_ID> \
    --query "Distribution.ARN" --output text
```

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

Apply the policy:
```bash
aws s3api put-bucket-policy \
    --bucket jurigregg-metronome \
    --policy file:///tmp/bucket-policy.json
```

### Step 7: Create CloudFront Function

Create function code file (`/tmp/metronome-router.js`):
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

Create the function:
```bash
aws cloudfront create-function \
    --name metronome-router \
    --function-config '{
        "Comment": "Route /metronome paths to index.html",
        "Runtime": "cloudfront-js-2.0"
    }' \
    --function-code fileb:///tmp/metronome-router.js
```

Publish the function (required before use):
```bash
aws cloudfront publish-function \
    --name metronome-router \
    --if-match <ETAG_FROM_CREATE>
```

Get the function ARN for the cache behavior.

### Step 8: Update CloudFront Distribution

This is the most complex step. It requires:
1. Get current distribution config
2. Add new origin
3. Add new cache behavior
4. Update the distribution

**Get current config:**
```bash
aws cloudfront get-distribution-config --id <DISTRIBUTION_ID> > /tmp/dist-config.json
```

**Extract and modify the config:**

The config needs these additions:

**New Origin** (add to `Origins.Items` array):
```json
{
    "Id": "metronome-s3",
    "DomainName": "jurigregg-metronome.s3.us-east-1.amazonaws.com",
    "S3OriginConfig": {
        "OriginAccessIdentity": ""
    },
    "OriginAccessControlId": "<OAC_ID>",
    "ConnectionAttempts": 3,
    "ConnectionTimeout": 10,
    "OriginShield": {
        "Enabled": false
    }
}
```

**New Cache Behavior** (add to `CacheBehaviors.Items` array, BEFORE default):
```json
{
    "PathPattern": "/metronome*",
    "TargetOriginId": "metronome-s3",
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
    "Compress": true,
    "FunctionAssociations": {
        "Quantity": 1,
        "Items": [
            {
                "FunctionARN": "<FUNCTION_ARN>",
                "EventType": "viewer-request"
            }
        ]
    }
}
```

Note: `658327ea-f89d-4fab-a63d-7e88639e58f6` is the AWS managed "CachingOptimized" policy ID.

**Update the distribution:**
```bash
aws cloudfront update-distribution \
    --id <DISTRIBUTION_ID> \
    --if-match <ETAG> \
    --distribution-config file:///tmp/updated-dist-config.json
```

### Step 9: Wait for Deployment

CloudFront distributions take time to deploy changes:
```bash
aws cloudfront wait distribution-deployed --id <DISTRIBUTION_ID>
```

This can take 5-15 minutes.

### Step 10: Invalidate Cache

Clear any cached content:
```bash
aws cloudfront create-invalidation \
    --distribution-id <DISTRIBUTION_ID> \
    --paths "/metronome" "/metronome/" "/metronome/*"
```

### Step 11: Verify Deployment

Test the URLs:
```bash
curl -I https://jurigregg.com/metronome
curl -I https://jurigregg.com/metronome/
```

Both should return HTTP 200 with `Content-Type: text/html`.

## Validation Gates

- [ ] **Gate 1: S3 Bucket Exists** - `aws s3 ls s3://jurigregg-metronome/` shows index.html
- [ ] **Gate 2: OAC Created** - Origin Access Control exists for metronome
- [ ] **Gate 3: Bucket Policy Applied** - Policy allows CloudFront access
- [ ] **Gate 4: CloudFront Function Published** - Function is in LIVE stage
- [ ] **Gate 5: Distribution Updated** - New origin and behavior exist in config
- [ ] **Gate 6: Distribution Deployed** - Status is "Deployed" not "InProgress"
- [ ] **Gate 7: URL Returns 200** - `curl -I https://jurigregg.com/metronome` returns HTTP 200
- [ ] **Gate 8: Content Correct** - `curl https://jurigregg.com/metronome` returns metronome HTML
- [ ] **Gate 9: Trailing Slash Works** - `https://jurigregg.com/metronome/` also works

## Error Handling

### Common Issues

1. **Bucket name already exists**
   - S3 bucket names are globally unique
   - Try alternative names: `jurigregg-metronome-app`, `jg-metronome-prod`
   - Update all subsequent commands with the actual bucket name

2. **Access Denied on S3**
   - Verify bucket policy has correct distribution ARN
   - Ensure OAC ID is correctly set in the origin config
   - Check that public access block is configured correctly

3. **CloudFront function syntax error**
   - Validate JavaScript syntax before creating
   - CloudFront JS runtime is not full Node.js - limited APIs
   - Test function in AWS Console if CLI fails

4. **Distribution update fails with "PreconditionFailed"**
   - ETag has changed - re-fetch the config and try again
   - Someone else may have modified the distribution

5. **403 Forbidden after deployment**
   - Check S3 bucket policy matches distribution ARN exactly
   - Verify OAC is attached to origin
   - Clear browser cache and retry

6. **CloudFront function not triggering**
   - Verify function is published (not just created)
   - Check function association in cache behavior
   - Ensure path pattern matches (`/metronome*` not `/metronome/*`)

7. **DNS/SSL issues**
   - These should not occur since we're using existing distribution
   - If they do, the existing jurigregg.com setup has issues

### Rollback Procedure

If deployment fails and needs rollback:

1. Remove cache behavior from CloudFront distribution
2. Remove origin from CloudFront distribution
3. Delete CloudFront function
4. Delete S3 bucket:
   ```bash
   aws s3 rb s3://jurigregg-metronome --force
   ```
5. Delete OAC:
   ```bash
   aws cloudfront delete-origin-access-control --id <OAC_ID> --if-match <ETAG>
   ```

## Output

This PRP produces:
- S3 bucket with `index.html` uploaded
- CloudFront configuration updates (origin, behavior, function)
- Working URL: https://jurigregg.com/metronome

No files are created in the repository - all changes are AWS infrastructure.

## Success Criteria

The deployment is complete when:

1. S3 bucket `jurigregg-metronome` (or variant) contains `index.html`
2. CloudFront function `metronome-router` is published and live
3. CloudFront distribution has new origin and cache behavior
4. Distribution status is "Deployed"
5. https://jurigregg.com/metronome returns the metronome app
6. https://jurigregg.com/metronome/ also returns the metronome app
7. No SSL/certificate warnings in browser
8. Cache-Control header is present (max-age=3600)

---

## Confidence Score: 6/10

**Areas of high confidence:**
- S3 bucket creation and file upload (straightforward CLI commands)
- CloudFront function creation (exact code provided in task.md)
- OAC creation (documented AWS procedure)

**Areas of uncertainty:**

1. **Existing distribution configuration (-2 points)**: We don't know the exact structure of the existing CloudFront distribution. The update-distribution command requires merging new config with existing config, which may have unexpected fields or structures.

2. **Distribution ID retrieval (-1 point)**: The query to find the distribution may not work if aliases are configured differently than expected. May need user input.

3. **Interactive nature of deployment (-1 point)**: Multiple steps depend on outputs from previous steps (OAC ID, function ARN, ETag values). This requires careful handling of intermediate values and may need user intervention if something fails.

**Recommendations for execution:**
- Execute steps incrementally with verification after each
- Be prepared to ask user for distribution ID if auto-detection fails
- Save all intermediate IDs and ARNs for use in subsequent steps
- If distribution update is complex, consider using AWS Console for that step and documenting what was done

**This PRP is best executed interactively** rather than as a fully automated script, given the dependencies between steps and potential for environment-specific variations.
