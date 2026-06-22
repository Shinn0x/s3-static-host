<div align="center">

# 🪣 S3 Static Website + CloudFront

**A hands-on AWS project — a static site hosted the production way:**
private S3 origin · CloudFront CDN · HTTPS · custom subdomain · least-privilege IAM

[![Live](https://img.shields.io/badge/live-mini--static.shinn.life-2563eb)](https://mini-static.shinn.life/)
![AWS](https://img.shields.io/badge/AWS-S3%20·%20CloudFront%20·%20Route%2053-FF9900?logo=amazonaws&logoColor=white)
![HTTPS](https://img.shields.io/badge/TLS-ACM-3ddc84)
![Bucket](https://img.shields.io/badge/bucket-private%20(OAC)-555)

### 🔗 **[https://mini-static.shinn.life/](https://mini-static.shinn.life/)**

</div>

---

## 📄 What this is

A self-contained landing page I built to learn the full static-hosting stack on AWS — not just dropping files in a bucket, but the production-style setup: a **private** S3 origin, a **CloudFront** CDN in front of it, **HTTPS** via ACM, and **Route 53** for the custom subdomain. The page itself doubles as a mini intro card (role, stack, links).

> **Domain note:** `shinn.life` is **registered at GoDaddy**, but DNS is managed in **AWS Route 53**. I created a Route 53 hosted zone for the domain and pointed GoDaddy's nameservers at it, so all records (the cert validation and the CloudFront alias) live in AWS.

| File | Purpose |
|------|---------|
| `index.html` | Home / index document — the About Me card |
| `error.html` | Custom error page served on missing keys (404) |
| `iam-policy.json` | Least-privilege IAM policy used to deploy this project |

---

## 🏗️ Architecture

```
Browser
  │  https://mini-static.shinn.life
  ▼
GoDaddy (registrar)  ── nameservers delegated to Route 53
  ▼
Route 53  (hosted zone; alias record: mini-static.shinn.life → CloudFront)
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

---

## ⚙️ How it works (request flow)

1. **DNS** — visitor opens `https://mini-static.shinn.life`. The domain is registered at **GoDaddy**, but its nameservers are delegated to **Route 53**, so GoDaddy hands the lookup to AWS.
2. **Route 53** — a hosted-zone **alias** record resolves the subdomain to the **CloudFront** distribution.
3. **CloudFront edge** — the nearest edge location (Singapore) terminates **HTTPS** using the **ACM** certificate. On a **cache hit** it serves the file instantly; on a **miss** it goes to the origin.
4. **OAC → S3** — CloudFront signs the request with its **Origin Access Control** identity and reads from the **private** S3 bucket. The bucket policy trusts **only this distribution**, so direct S3 URLs return `403`.
5. **Response** — S3 returns the object, CloudFront caches it at the edge and serves the visitor.
6. **Errors** — if the path doesn't exist, the private bucket returns `403` (not `404`); CloudFront maps **403/404 → `error.html`** so any bad URL shows the styled page.

### Design decisions (the "why")
- **Manual S3 + CloudFront over Amplify** — to demonstrate the infrastructure, not hide it behind a managed service.
- **Subdomain over path routing** — self-contained, no URL-rewrite function, fewer failure modes.
- **Private bucket + OAC over a public bucket** — security by default; the bucket is never publicly reachable.
- **Least-privilege IAM user** — scoped to only the bucket + hosted zone, no admin, no `iam:*`, programmatic key only.
- **Alias record over CNAME/A** — resolves at the edge, points straight at CloudFront, and is free.

---

## ✨ Features

- **Zero dependencies** — single-file HTML with inline CSS + vanilla JS
- **Custom logo** in the browser tab (inline SVG favicon, no extra request)
- **Light / dark theme toggle** that remembers your choice (`localStorage`)
- **Animated aurora background** and a typing-effect role line
- **Live clock** (Singapore time)
- **Custom 404** (`error.html`) wired through CloudFront error responses
- Fully responsive, accessible markup

## 🧰 Tech

`HTML` · `CSS` · `Vanilla JS` · `AWS S3` · `CloudFront` · `Route 53` · `ACM (TLS)` · `Origin Access Control`

---

## 🚀 How it's deployed

### 0. Deploy as a least-privilege IAM user
Rather than using root or an admin account, this project is deployed by a dedicated
IAM user scoped to **only** the actions it needs — see [`iam-policy.json`](./iam-policy.json).
S3 and Route 53 permissions are locked to the specific bucket and hosted zone; CloudFront
and ACM are limited by action (they don't support resource-level scoping for these calls).

```bash
# create a customer-managed policy from the file, then attach it to the user
aws iam create-policy --policy-name mini-static-deploy \
  --policy-document file://iam-policy.json
```
Replace `<bucket-name>` and `<hosted-zone-id>` in the file first. Use a programmatic
access key (no console login) and deploy via a named profile:
`aws configure --profile mini-static` → add `--profile mini-static` to the commands below.

### 1. Delegate DNS from GoDaddy to Route 53 (one-time, registrar setup)
The domain is registered at **GoDaddy**, but I want **Route 53** to be the DNS authority.
1. In Route 53, **create a hosted zone** for `shinn.life`. AWS assigns 4 nameservers (the **NS record**), e.g. `ns-123.awsdns-45.com`, etc.
2. In the **GoDaddy** domain dashboard → *Nameservers* → switch from "GoDaddy default" to **Custom**, and paste in the 4 Route 53 nameservers.
3. Wait for propagation (minutes to a few hours). After this, all DNS for `shinn.life` is answered by Route 53.

> The domain stays **registered** at GoDaddy; only the **DNS hosting** moves to AWS. Note the hosted zone ID (`Z…`) — it goes into `iam-policy.json` and the record commands.

### 2. Create a private bucket and upload the site
Create the bucket in **`ap-southeast-1` (Singapore)** — closest origin for the audience.
```bash
aws s3 mb s3://<bucket-name> --region ap-southeast-1

aws s3 sync . s3://<bucket-name>/ \
  --exclude "README.md" --exclude "LICENSE" \
  --exclude ".git/*" --exclude ".gitignore"
```
Leave **Block all public access ON** — CloudFront, not the public, reads this bucket.

### 3. Request a TLS certificate (ACM)
Request a public cert for `mini-static.shinn.life` (or a wildcard `*.shinn.life`) in **us-east-1** — CloudFront requires certs in N. Virginia, regardless of where the bucket lives — and validate it via DNS (ACM can add the validation `CNAME` straight into the Route 53 zone).

### 4. Create the CloudFront distribution
- **Origin:** the S3 bucket (REST endpoint `…s3.ap-southeast-1.amazonaws.com`, *not* the `s3-website` endpoint)
- **Origin access:** create an **Origin Access Control (OAC)** and let CloudFront update the bucket policy so only this distribution can `s3:GetObject`
- **Viewer protocol policy:** Redirect HTTP → HTTPS
- **Alternate domain (CNAME):** `mini-static.shinn.life`, attached to the ACM cert
- **Default root object:** `index.html`

#### S3 bucket policy (lets only CloudFront read objects)
With OAC, the bucket stays private and trusts **just this distribution**. CloudFront
offers to write this policy for you; this is what it looks like:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipalReadOnly",
      "Effect": "Allow",
      "Principal": { "Service": "cloudfront.amazonaws.com" },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::<bucket-name>/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::<account-id>:distribution/<distribution-id>"
        }
      }
    }
  ]
}
```
The `Condition` ties read access to *this one distribution* — direct S3 URLs return **403**.

### 5. Add custom error responses (serve `error.html` for bad URLs)
Goal: a visitor hitting a path that doesn't exist (e.g. `mini-static.shinn.life/sadjfjas`)
gets the styled `error.html`, not CloudFront's default error screen.

**Gotcha:** with a **private** bucket + OAC, a missing object returns **`403 AccessDenied`**,
*not* `404` — because the policy grants `s3:GetObject` but **not** `s3:ListBucket`, so S3 won't
confirm the object's absence. CloudFront therefore sees a **403**. So map **both** codes.

In the distribution's **Error pages** tab, create two custom error responses:

| HTTP error code | Response page path | HTTP Response code | Min TTL |
|-----------------|--------------------|--------------------|---------|
| **403** | `/error.html` | **404** | 10 |
| **404** | `/error.html` | **404** | 10 |

The **response code 404** is what the browser receives — translating the origin's 403/404 into
an honest "not found" for the visitor (and `error.html` is marked `noindex`). S3's built-in error
document doesn't apply behind OAC, so CloudFront handles this.

### 6. Point the subdomain at CloudFront (Route 53)
In the `shinn.life` hosted zone, create an **alias** record (alias records are free and resolve at the edge — use them instead of a plain A/CNAME):
- **Record name:** `mini-static` · **Type:** `A` · **Alias:** Yes → *Alias to CloudFront distribution* → pick the distribution
- Optionally add a matching **`AAAA`** alias record for IPv6

---

## 🔄 Updating the live site

Changed `index.html` or `error.html`? Two commands push it live:

```bash
# 1. upload only the changed files
aws s3 sync . s3://<bucket-name>/ \
  --exclude "README.md" --exclude "LICENSE" \
  --exclude ".git/*" --exclude ".gitignore" \
  --profile mini-static

# 2. clear the CloudFront cache so visitors see the new version right away
aws cloudfront create-invalidation \
  --distribution-id <distribution-id> --paths "/*" \
  --profile mini-static
```

> Without step 2, edge locations keep serving the cached copy until the TTL expires — so the invalidation is what makes the update show up immediately.

---

## 💻 Run it locally

No build needed — open `index.html` directly, or serve the folder:
```bash
python3 -m http.server 8000
# then visit http://localhost:8000
```

---

## 🎓 What I learned

- Applying the **least-privilege principle** with a dedicated IAM user scoped to only the actions this deploy needs (and where resource-level scoping is/isn't possible)
- Hosting a static site on S3 with a **private** bucket (no public access)
- Putting **CloudFront** in front of S3 and wiring **OAC** so only the CDN can read the origin
- Provisioning **HTTPS** with ACM (and why the cert must live in us-east-1)
- Delegating DNS from a third-party registrar (**GoDaddy**) to **Route 53** by switching nameservers
- Routing a custom subdomain with **Route 53** alias records
- Writing the **S3 bucket policy** that grants `s3:GetObject` to only the CloudFront distribution (via OAC)
- Handling 404s with **CloudFront custom error responses** vs. S3 error documents
- Cache invalidation as part of the update workflow

---

<div align="center">

Built & deployed by **Shinn** · [Portfolio](https://www.shinn.life) · [GitHub](https://github.com/Shinn0x)

</div>
