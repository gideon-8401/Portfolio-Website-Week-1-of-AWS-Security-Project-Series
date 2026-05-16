# AWS Portfolio Website — Week 1 of AWS Security Project Series

A hardened static portfolio site for **Gideon Oteng**, served from a private S3 bucket through CloudFront with AWS WAF and an ACM-issued certificate. This is project one in a five-week AWS portfolio series focused on cloud security and architecture.

## Overview

The site itself is plain HTML, CSS (Tailwind via the Play CDN), and a small amount of vanilla JavaScript — no build step, no framework, no bundler. Everything in this repository is deployable directly to S3 as static files.

The interesting work for Week 1 is the **AWS infrastructure that fronts** these files: a CloudFront distribution that terminates TLS via ACM, evaluates WAF managed rules, and pulls origin content from a private S3 bucket using Origin Access Control (OAC). The bucket itself blocks all public access; CloudFront is the only path in.

## Architecture

```
User ──▶ Route 53 ──▶ CloudFront (TLS via ACM, AWS WAF) ──▶ S3 (private, OAC)
```

A diagram is included at `assets/architecture-week1.svg` — replace it with your finalized version once the infrastructure is built.

## AWS services used

- **Amazon S3** — origin bucket for the static files (private, public access blocked).
- **Amazon CloudFront** — global CDN, TLS termination, OAC origin access.
- **AWS WAF** — web ACL with AWS managed rule groups attached to the distribution.
- **AWS Certificate Manager (ACM)** — public TLS certificate, issued in `us-east-1` for use with CloudFront.
- **Amazon Route 53** — DNS hosted zone with alias records pointing to the distribution.
- **AWS IAM** — least-privilege policies for deployment and OAC.

## Security features implemented

- HTTPS enforced end-to-end via ACM, with HTTP redirected to HTTPS at CloudFront.
- AWS WAF web ACL attached to the distribution (managed rule groups: Common, Known Bad Inputs, optionally rate-based).
- S3 bucket has **all four public access block settings enabled** and no bucket-level public ACLs.
- Origin access uses **CloudFront Origin Access Control (OAC)** with SigV4; the bucket policy grants `s3:GetObject` only to the distribution principal.
- Modern TLS: minimum protocol set to `TLSv1.2_2021` on the CloudFront viewer policy.
- Strong default cache behavior; no cookies forwarded; query strings normalized.
- Optional: CloudFront access logs delivered to a separate, lifecycle-managed log bucket.

## Repository layout

```
portfolio-site/
├── index.html                  # single-page site
├── css/
│   └── custom.css              # minimal layer on top of Tailwind
├── assets/
│   ├── favicon.svg
│   ├── profile-placeholder.svg # OG / share image placeholder
│   └── architecture-week1.svg  # placeholder diagram
├── robots.txt
└── README.md
```

## Deploy.

Set these in your shell first:

```bash
BUCKET=your-portfolio-bucket-name
DISTRIBUTION_ID=YOURDISTRIBUTIONID
```

Then sync the site and invalidate the CloudFront cache:

```bash
# Upload the site (delete removes stale objects from previous deploys)
aws s3 sync ./portfolio-site "s3://$BUCKET" \
    --delete \
    --exclude "README.md" \
    --exclude ".git/*" \
    --exclude ".DS_Store"

# Invalidate the CloudFront edge caches so visitors see the new version
aws cloudfront create-invalidation \
    --distribution-id "$DISTRIBUTION_ID" \
    --paths "/*"
```

For better cache behavior on subsequent deploys, you can split the sync into two passes — long cache for hashed assets, short cache for `index.html`:

```bash
# Long-lived assets
aws s3 sync ./portfolio-site "s3://$BUCKET" \
    --delete \
    --exclude "index.html" \
    --exclude "README.md" \
    --cache-control "public,max-age=31536000,immutable"

# Short-lived HTML
aws s3 cp ./portfolio-site/index.html "s3://$BUCKET/index.html" \
    --cache-control "public,max-age=60,must-revalidate"
```

## Lessons learned

_Placeholder — fill this in after the build:_

- What you learned about CloudFront + OAC vs. legacy OAI.
- Surprises with WAF managed rules (false positives, costs).
- Anything about ACM certificate validation timing.
- Cache invalidation gotchas.

## License

MIT
