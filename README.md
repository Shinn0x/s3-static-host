# S3 Static Website + CloudFront

A hands-on AWS project: a single-page **"About Me"** site (plain HTML/CSS/JS, no build step) hosted on **Amazon S3** and delivered globally through **Amazon CloudFront** — served on a custom subdomain with HTTPS, a **private** bucket locked down via **Origin Access Control (OAC)**, and a custom error page.

> 🔗 **Live demo:** _add your URL here_ → `https://s3-static.shinn.life`

---

## What this is

A self-contained landing page I built to learn the full static-hosting stack on AWS — not just dropping files in a bucket, but the production-style setup: a **private** S3 origin, a **CloudFront** CDN in front of it, **HTTPS** via ACM, and **Route 53** for the custom subdomain. The page itself doubles as a mini intro card (role, stack, links).

| File | Purpose |
|------|---------|
| `index.html` | Home / index document — the About Me card |
| `error.html` | Custom error page served on missing keys (404) |

## Architecture

```
Browser
  │  https://s3-static.shinn.life
  ▼
Route 53  (alias record: s3-static.shinn.life → CloudFront)
  ▼
CloudFront distribution   ── HTTPS via ACM cert, edge caching
  │  single origin (OAC-signed)
  ▼
S3 bucket  (PRIVATE — public access blocked; only CloudFront can read)
```

**Why CloudFront on top of S3?**
- **HTTPS** + a real subdomain (S3 website endpoints are HTTP-only)
- **Global edge caching** → faster loads worldwide (incl. Singapore edge locations)
- **Private bucket** via OAC → the bucket is never publicly exposed; only the distribution can fetch objects
- **Custom error responses** for a polished 404

## Features

- **Zero dependencies** — single-file HTML with inline CSS + vanilla JS
- **Light / dark theme toggle** that remembers your choice (`localStorage`)
- **Animated aurora background** and a typing-effect role line
- **Live clock** (Singapore time)
- **Custom 404** (`error.html`) wired through CloudFront error responses
- Fully responsive, accessible markup

## Tech

`HTML` · `CSS` · `Vanilla JS` · `AWS S3` · `CloudFront` · `Route 53` · `ACM (TLS)` · `Origin Access Control`

---

## How it's deployed

### 1. Create a private bucket and upload the site
Create the bucket in **`ap-southeast-1` (Singapore)** — closest origin for the audience.
```bash
aws s3 mb s3://<bucket-name> --region ap-southeast-1

aws s3 sync . s3://<bucket-name>/ \
  --exclude "README.md" --exclude "LICENSE" \
  --exclude ".git/*" --exclude ".gitignore"
```
Leave **Block all public access ON** — CloudFront, not the public, reads this bucket.

### 2. Request a TLS certificate (ACM)
Request a public cert for `s3-static.shinn.life` (or a wildcard `*.shinn.life`) in **us-east-1** — CloudFront requires certs in N. Virginia, regardless of where the bucket lives — and validate it via DNS.

### 3. Create the CloudFront distribution
- **Origin:** the S3 bucket (REST endpoint, *not* the website endpoint)
- **Origin access:** create an **Origin Access Control (OAC)** and let CloudFront update the bucket policy so only this distribution can `s3:GetObject`
- **Viewer protocol policy:** Redirect HTTP → HTTPS
- **Alternate domain (CNAME):** `s3-static.shinn.life`, attached to the ACM cert
- **Default root object:** `index.html`

### 4. Add a custom error response
In the distribution's **Error pages** tab:
- HTTP error code **404** → response page `/error.html`, response code **404**

(S3's built-in error document doesn't apply when the bucket is private behind OAC — CloudFront handles 404s.)

### 5. Point the subdomain at CloudFront (Route 53)
Create an **A / AAAA alias** record for `s3-static.shinn.life` → the CloudFront distribution.

### 6. Redeploy after changes
```bash
aws s3 sync . s3://<bucket-name>/ --exclude "README.md" --exclude ".git/*"
aws cloudfront create-invalidation --distribution-id <id> --paths "/*"
```
The invalidation clears the edge cache so visitors see the latest version immediately.

---

## Run it locally

No build needed — open `index.html` directly, or serve the folder:
```bash
python3 -m http.server 8000
# then visit http://localhost:8000
```

---

## What I learned

- Hosting a static site on S3 with a **private** bucket (no public access)
- Putting **CloudFront** in front of S3 and wiring **OAC** so only the CDN can read the origin
- Provisioning **HTTPS** with ACM (and why the cert must live in us-east-1)
- Routing a custom subdomain with **Route 53** alias records
- Handling 404s with **CloudFront custom error responses** vs. S3 error documents
- Cache invalidation as part of the deploy workflow

---

Built & deployed by **Shinn** · [Portfolio](https://www.shinn.life) · [GitHub](https://github.com/Shinn0x)
