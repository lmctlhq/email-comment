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

Route53: lmctl.com MX record → SES sending domain verified
```

---

## Components

### 1. Lambda — Web UI + API
- Single Lambda function handles all routes
- `GET /`                      — serve comment form HTML (static, embedded in function)
- `POST /comments`             — accept new comment submission
- `GET /comments?slug=<slug>`  — return approved comments for a page (JSON)
- `POST /admin/approve?id=<id>` — approve a comment (token-protected, Phase 1)
- Deployed behind API Gateway HTTP API (cheaper, simpler than REST API)

### 2. DynamoDB
- Table name: `email-comment`
- Billing: on-demand (pay-per-request)
- Schema:

| Attribute    | Type   | Role                            |
|--------------|--------|---------------------------------|
| `slug`       | String | Partition key (page identifier) |
| `created_at` | String | Sort key (ISO8601 UTC)          |
| `id`         | String | UUID, used for admin actions    |
| `name`       | String | Commenter name (required)       |
| `email`      | String | Commenter email (optional)      |
| `message`    | String | Comment body (required)         |
| `approved`   | Bool   | Default `false`; public reads filter on `true` |

### 3. SES
- Verified sending domain: `lmctl.com`
- Sender address: `noreply@lmctl.com`
- Notification email sent to owner on every new submission
- Route53 MX record already in place for domain verification

### 4. API Gateway
- HTTP API type (not REST API — lower cost, sufficient for this use case)
- Single route `/{proxy+}` forwarding all requests to Lambda

### 5. Route53 / DNS
- Existing MX record on `lmctl.com` used for SES domain verification
- No additional DNS changes required for Phase 1

---

## Data Flow

```
1. User fills form  →  POST /comments  →  Lambda
2. Lambda validates input (slug, name, message required)
3. Lambda writes comment to DynamoDB  (approved=false)
4. Lambda sends SES email to owner:
       Subject: "New comment on <slug>"
       Body:    name, message, approval link
5. Owner clicks approval link  →  POST /admin/approve?id=<id>&token=<secret>
6. Lambda sets approved=true in DynamoDB
7. Comment appears on page via GET /comments?slug=<slug>
```

---

## Recommendations for Missing Requirements

### Moderation
- **Phase 1**: Approval link embedded in owner notification email (token in query string)
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
- API Gateway default throttling (10k req/s burst, 5k req/s steady) is sufficient for Phase 1
- Add per-IP throttling via WAF if abuse occurs

---

## Deployment

- **IaC**: AWS SAM (`template.yaml`) — single file defines all resources
- Resources defined: Lambda function, API Gateway HTTP API, DynamoDB table, IAM role
- SES identity assumed pre-verified (lmctl.com already set up)
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
