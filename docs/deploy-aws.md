# AWS deployment notes

Working notes for hosting `google-docs-mcp` on AWS. Upstream targets Google Cloud Run + Firestore; this doc records what changes for AWS.

## Local dev (verified)

```bash
cp .env.example .env                 # then fill GOOGLE_CLIENT_ID / _SECRET
docker compose up -d --build
curl -s http://localhost:8088/.well-known/oauth-authorization-server
```

Container listens on 8080 internally; compose maps host 8088 → container 8080. `BASE_URL` must match the **host-visible** URL (`http://localhost:8088` locally, the public HTTPS URL in AWS).

## Runtime contract

- Single Node process, HTTP on `PORT` (default 8080)
- Stateless per request, but **session state lives in memory by default** — OAuth sessions are lost on restart unless a persistent token store is wired in
- Needs outbound HTTPS to `accounts.google.com`, `oauth2.googleapis.com`, and the Google Docs/Sheets/Drive/Gmail/Calendar APIs
- No inbound traffic other than MCP clients + OAuth redirects

## Required env vars (production)

| Variable               | Source                  | Notes                                                      |
| ---------------------- | ----------------------- | ---------------------------------------------------------- |
| `MCP_TRANSPORT`        | static                  | `httpStream`                                               |
| `BASE_URL`             | infra output            | Public HTTPS URL (ALB / App Runner / CloudFront)           |
| `PORT`                 | static                  | `8080`                                                     |
| `GOOGLE_CLIENT_ID`     | Secrets Manager         | OAuth client (Web application type)                        |
| `GOOGLE_CLIENT_SECRET` | Secrets Manager         | ditto                                                      |
| `JWT_SIGNING_KEY`      | Secrets Manager         | `openssl rand -hex 32`; pin so tokens survive restarts     |
| `TOKEN_ENCRYPTION_KEY` | Secrets Manager         | Optional; encrypts refresh tokens at rest                  |
| `ALLOWED_DOMAINS`      | static                  | `treemetrics.com` (restrict sign-in to our workspace)      |
| `TOKEN_STORE`          | *unset for now*         | See "Token persistence" below                              |

Google OAuth client's authorised redirect URI **must** be `${BASE_URL}/oauth/callback`.

## Hosting options (to decide)

| Option            | Pros                                                    | Cons                                                    |
| ----------------- | ------------------------------------------------------- | ------------------------------------------------------- |
| **ECS Fargate**   | Standard pattern; full control; easy Secrets Manager    | Needs ALB + target group + TLS cert; more moving parts  |
| **App Runner**    | Simplest: HTTPS + autoscale + secrets built-in          | Less network control; VPC connector costs extra         |
| **EC2 + compose** | Cheapest if we already have a box                       | Manual TLS, manual updates, single point of failure     |

Recommendation: **App Runner** for first cut (lowest operational overhead) unless we need VPC-private outbound or fine-grained networking, in which case **ECS Fargate behind an ALB**.

## Token persistence (open question)

Upstream ships a Firestore adapter (`src/firestoreTokenStorage.ts`). AWS options:

1. **In-memory (default)** — simplest; every redeploy forces users to re-auth. Acceptable for a small team while we iterate.
2. **DynamoDB adapter** — mirror `FirestoreTokenStorage` against a single-table DDB design. ~100 LOC. Best fit for AWS.
3. **S3 adapter** — cheapest, but higher latency and weaker consistency; not recommended.

Decision TBD. Ship with in-memory, add DDB adapter once we pick a hosting target.

## Rough deploy sketch (App Runner)

```bash
# 1. Push image to ECR
aws ecr create-repository --repository-name google-docs-mcp
docker buildx build --platform linux/amd64 -t <acct>.dkr.ecr.<region>.amazonaws.com/google-docs-mcp:<sha> --push .

# 2. Secrets
aws secretsmanager create-secret --name google-docs-mcp/google-client-id --secret-string "..."
aws secretsmanager create-secret --name google-docs-mcp/google-client-secret --secret-string "..."
aws secretsmanager create-secret --name google-docs-mcp/jwt-signing-key --secret-string "$(openssl rand -hex 32)"

# 3. App Runner service
#    - Image: the ECR URI above
#    - Port: 8080
#    - Env vars: MCP_TRANSPORT=httpStream, PORT=8080, BASE_URL=<service URL>,
#                ALLOWED_DOMAINS=treemetrics.com
#    - Secret refs: GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET, JWT_SIGNING_KEY
#    - Health check: TCP 8080 (or HTTP GET / — returns 200 with landing page)

# 4. After first deploy, update Google OAuth client's redirect URI to
#    https://<service-url>/oauth/callback and set BASE_URL to the same host.
```

## Health check

No dedicated `/health` endpoint. Use `GET /` — the landing page returns 200 in remote mode. Adding a cheap `/healthz` endpoint is a small follow-up if needed for deeper probes.
