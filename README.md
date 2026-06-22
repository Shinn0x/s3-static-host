<div align="center">

# 🪣 S3 Static Website + CloudFront

**A hands-on AWS project — a static site hosted the production way:**
private S3 origin · CloudFront CDN · HTTPS · custom subdomain · least-privilege IAM

<br>

[![Live](https://img.shields.io/badge/Live-mini--static.shinn.life-2ea44f?style=for-the-badge&logo=googlechrome&logoColor=white)](https://mini-static.shinn.life/)

![Amazon S3](https://img.shields.io/badge/Amazon%20S3-569A31?style=flat&logo=amazons3&logoColor=white)
![Amazon CloudFront](https://img.shields.io/badge/CloudFront-FF9900?style=flat&logo=amazonwebservices&logoColor=white)
![Amazon Route 53](https://img.shields.io/badge/Route%2053-8C4FFF?style=flat&logo=amazonwebservices&logoColor=white)
![AWS Certificate Manager](https://img.shields.io/badge/ACM%20TLS-DD344C?style=flat&logo=amazonwebservices&logoColor=white)
<br>
![HTML5](https://img.shields.io/badge/HTML5-E34F26?style=flat&logo=html5&logoColor=white)
![CSS3](https://img.shields.io/badge/CSS3-1572B6?style=flat&logo=css3&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=flat&logo=javascript&logoColor=black)

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

Built & deployed by **Shinn** · [Portfolio](https://localhostshinn.xyz) · [GitHub](https://github.com/Shinn0x)

</div>
