# email-comment: Design Document

## Overview

A simple email-based commenting system for lmctl.com. Visitors submit comments via a web form; the system stores them in DynamoDB and notifies the site owner via AWS SES.

---

## Architecture

```
User Browser
    │
    ▼
API Gateway (HTTP API)
    │
    ▼
Lambda (UI + API handler)
    │
    ├──► DynamoDB  (persist comments)
    │
    └──► SES       (notify owner)

Route53: lmctl.com → SES domain identity (TXT + DKIM CNAMEs via Route53)
```

---

## Components

### 1. Lambda — Web UI + API
- Single Lambda function handles all routes
- `GET /`                      — serve comment form HTML (static, embedded in function)
- `POST /comments`             — accept new comment submission
- `GET /comments?slug=<slug>`  — return approved comments for a page (JSON)
- `GET /admin/approve?id=<id>&token=<secret>` — approve a comment (token-protected, Phase 1; GET so email links work directly)
- Deployed behind API Gateway HTTP API (cheaper, simpler than REST API)

### 2. DynamoDB
- Table name: `email-comment`
- Billing: on-demand (pay-per-request)
- Schema:

| Attribute    | Type   | Role                                        |
|--------------|--------|---------------------------------------------|
| `id`         | String | Partition key (UUID); direct key for approval |
| `slug`       | String | GSI partition key for per-page queries      |
| `created_at` | String | GSI sort key (ISO8601 UTC)                  |
| `name`       | String | Commenter name (required)                   |
| `email`      | String | Commenter email (optional)                  |
| `message`    | String | Comment body (required)                     |
| `approved`   | Bool   | Default `false`; public reads filter on `true` |

- GSI name: `slug-created_at-index` (partition: `slug`, sort: `created_at`)
- `GET /comments?slug=<slug>` queries the GSI; `GET /admin/approve?id=<id>` does a direct table lookup by partition key — no scan needed

### 3. SES
- Verified sending domain: `lmctl.com`
- Sender address: `noreply@lmctl.com`
- Notification email sent to owner on every new submission
- **SES domain identity verification** requires adding SES-provided DNS records to Route53:
  - One TXT record (domain ownership verification)
  - Three CNAME records (DKIM signing)
- MX record on `lmctl.com` is for receiving / custom MAIL FROM — it is separate from sending verification and is not sufficient on its own

### 4. API Gateway
- HTTP API type (not REST API — lower cost, sufficient for this use case)
- Route `$default` catches all paths including `/`; `/{proxy+}` alone does not match the root path

### 5. Route53 / DNS
- MX record on `lmctl.com` is pre-existing (for receiving email)
- **Additional records required for SES sending**: TXT verification record + 3 DKIM CNAME records — all added via Route53 during SES domain identity setup

---

## Data Flow

```
1. User fills form  →  POST /comments  →  Lambda
2. Lambda validates input (slug, name, message required)
3. Lambda writes comment to DynamoDB  (approved=false)
4. Lambda sends SES email to owner:
       Subject: "New comment on <slug>"
       Body:    name, message, approval link
5. Owner clicks approval link  →  GET /admin/approve?id=<id>&token=<secret>
6. Lambda sets approved=true in DynamoDB
7. Comment appears on page via GET /comments?slug=<slug>
```

---

## Recommendations for Missing Requirements

### Moderation
- **Phase 1**: Approval link (`GET /admin/approve?id=<id>&token=<secret>`) embedded in owner notification email; clicking it directly approves the comment
- **Phase 2**: Minimal admin UI page if volume grows

### Spam Protection
- Add a **honeypot hidden field** to the form (zero cost, effective against simple bots)
- If spam becomes a problem: add [Cloudflare Turnstile](https://www.cloudflare.com/products/turnstile/) (free, no CAPTCHA friction)

### Comment Display on Site
- Embed a small `<script>` snippet on each page that fetches `GET /comments?slug=<slug>` and renders approved comments
- Alternatively, server-side render if lmctl.com uses a static site generator

### Admin Authentication
- **Phase 1**: `ADMIN_TOKEN` env var on Lambda; token passed in query string for approval links
- **Phase 2**: Move to a signed URL or Cognito if needed

### Error Handling
- Lambda returns structured JSON errors with appropriate HTTP status codes
- SES send failures are logged to CloudWatch but do not fail the comment submission (comment is still saved)

### Rate Limiting
- API Gateway default throttling (10,000 RPS steady-state with additional burst capacity) is sufficient for Phase 1
- Add per-IP throttling via WAF if abuse occurs

---

## Deployment

- **IaC**: AWS SAM (`template.yaml`) — single file defines all resources
- Resources defined: Lambda function, API Gateway HTTP API, DynamoDB table, IAM role
- SES domain identity setup required: add TXT + 3 DKIM CNAME records to Route53 (done once via SES console or AWS CLI)
- Deploy: `sam build && sam deploy --guided`

### Environment Variables (Lambda)
| Variable       | Description                          |
|----------------|--------------------------------------|
| `TABLE_NAME`   | DynamoDB table name                  |
| `OWNER_EMAIL`  | Destination for owner notifications  |
| `SENDER_EMAIL` | SES verified sender (noreply@...)    |
| `ADMIN_TOKEN`  | Secret for approval endpoint         |

---

## Out of Scope — Phase 1

- Threaded / nested replies
- Rich text or Markdown in comments
- User accounts or OAuth
- Inbound email parsing (reply-to-comment via email)
- Search or pagination of comments
- Multi-site / multi-domain support
