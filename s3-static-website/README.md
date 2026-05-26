# S3 Static Website with Access Controls

**Services:** Amazon S3 · Bucket Policies · Static Website Hosting · Versioning  
**Goal:** Host a functional static website on S3 with properly configured public access, bucket policy enforcement, and versioning enabled for object recovery.

---

## What I Built

Configured an S3 bucket as a static website host — not just enabling the feature, but working through the full access control layer: understanding why AWS blocks public access by default, what it takes to open it correctly, and how to enforce those controls with a bucket policy.

---

## Architecture

```
Browser Request
      |
[S3 Bucket: my-static-site]
  - Static website hosting: ENABLED
  - Block all public access: OFF (intentional)
  - Bucket policy: Allow s3:GetObject to Principal: *
  - Versioning: ENABLED
      |
[index.html / error.html served via S3 website endpoint]
```

---

## Key Configuration Decisions

**Block Public Access settings**
S3 buckets block all public access by default — this is correct behavior. Disabling it for a static website requires a conscious, deliberate step. Understanding *why* this block exists (to prevent accidental data exposure) matters more than knowing how to turn it off.

**Bucket policy vs ACLs**
Used a bucket policy (`s3:GetObject` allow for all principals) rather than ACLs. AWS now recommends disabling ACLs entirely in favor of bucket policies — they're more readable, auditable, and controllable.

**Versioning**
Enabled versioning on the bucket so that overwritten or deleted objects are recoverable. In a production environment, this is critical — an accidental `PUT` or `DELETE` without versioning means the data is gone.

---

## Bucket Policy Used

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-static-site/*"
    }
  ]
}
```

---

## Troubleshooting Encountered

**Issue:** Website endpoint returned 403 Forbidden even after enabling static hosting.  
**Root cause:** Block Public Access was still enabled at the bucket level, overriding the bucket policy.  
**Fix:** Disabled the "Block all public access" setting, then re-applied the bucket policy.  
**Lesson:** Block Public Access settings act as a master override — even a correctly written bucket policy will be blocked if the public access block is active. The order of operations matters.

---

## What I'd Do Differently in Production

- Serve the site through **CloudFront** instead of directly from the S3 endpoint — adds HTTPS, global caching, and keeps the bucket itself private
- Use **S3 Origin Access Control (OAC)** with CloudFront so the bucket never needs to be public
- Enable **S3 server access logging** to capture all requests for audit purposes
- Set a **lifecycle policy** to expire old versions after 30–90 days to manage storage costs
