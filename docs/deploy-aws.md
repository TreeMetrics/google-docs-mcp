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

## Google OAuth client and consent screen

The server uses **MCP OAuth 2.1** with Google as the upstream identity provider. Getting this right once up front avoids the most common deployment failures.

### Client type

Create an OAuth 2.0 Client ID of type **Web application** (not Desktop). Desktop clients do not accept configurable redirect URIs and won't work with `MCP_TRANSPORT=httpStream`.

### Authorised redirect URIs

Must be exactly `${BASE_URL}/oauth/callback`. Add one entry per deployment target — you can list multiple on a single client:

- `http://localhost:8088/oauth/callback` — local Docker
- `https://<tunnel-hostname>/oauth/callback` — tunnel (see Local testing below)
- `https://<aws-url>/oauth/callback` — production

Two-client variant (recommended for production): one client for dev (localhost + tunnel URIs), a separate one for prod (prod URI only). A leaked dev secret then can't touch prod.

### Consent screen: keep it Internal

For a Google Workspace domain (e.g. `treemetrics.com`), set **User type = Internal**:

- Only `@treemetrics.com` accounts can sign in
- No Google app-verification review required, even for sensitive scopes
- No per-user test user list to maintain

Avoid **External + Production** — that opens the app to any Google account worldwide and triggers verification review (multi-week process) for Drive/Gmail scopes. **External + Testing** works for a handful of users but caps at 100 and each must be added as a test user manually.

If the "Internal" option is greyed out, the GCP project isn't owned by the Workspace org — create a new project inside the Workspace org and move the client there.

### Scopes to authorise on the consent screen

```
openid
email
https://www.googleapis.com/auth/documents
https://www.googleapis.com/auth/spreadsheets
https://www.googleapis.com/auth/drive
https://www.googleapis.com/auth/script.external_request
https://www.googleapis.com/auth/gmail.modify
https://www.googleapis.com/auth/calendar.events
```

All must be listed on the consent screen; missing scopes cause opaque "Something went wrong" errors during sign-in.

### Enable the APIs

On the GCP project, enable: Google Docs API, Google Sheets API, Google Drive API, Gmail API, Google Calendar API, Apps Script API. Missing APIs also surface as generic sign-in errors.

### Defence in depth: `ALLOWED_DOMAINS`

Set `ALLOWED_DOMAINS=treemetrics.com` even with Internal consent screen. Server-side enforcement means a misconfigured consent screen can't accidentally let outside accounts in.

## Local testing with a public URL

Claude Desktop and Claude.ai route custom connectors through Anthropic infrastructure — they cannot reach `http://localhost:8088` directly. To test the full connector flow before deploying to AWS, front the local server with a tunnel that gives a public HTTPS URL.

### Option A — Ephemeral `trycloudflare.com` (no auth)

```bash
cloudflared --config /dev/null tunnel --url http://localhost:8088
```

- No Cloudflare account needed, no DNS, no config
- URL is random (e.g. `https://<random>.trycloudflare.com`) and changes on every restart
- Each restart requires re-registering the callback URL in Google Cloud and updating `BASE_URL` in `.env`
- Fine for a one-off smoke test; painful for ongoing iteration

The explicit `--config /dev/null` is required if a `~/.cloudflared/config.yml` exists for other tunnels — otherwise cloudflared mixes settings and routing silently breaks.

### Option B — Named Cloudflare tunnel (stable hostname)

```bash
cloudflared login                                                    # browser → authorise zone
cloudflared tunnel create google-docs-mcp                            # creates tunnel + credentials
cloudflared tunnel route dns google-docs-mcp google-docs-mcp.treemetrics.dev
cloudflared tunnel run google-docs-mcp                               # in its own config
```

Stable URL, matches prod shape, but requires the logged-in Cloudflare account to have the **Cloudflare One Connector: cloudflared Write** permission on the target zone. Without that role, `tunnel route dns` fails with `Authentication error (code 10000)`.

### Option C — Skip tunnels, go straight to AWS

The final prod deployment gives you a stable HTTPS URL anyway (App Runner / ALB hostname). If you don't need to iterate on server code locally against a real MCP client, deploying directly is the cleanest path.

## MCP client integration (coworker-facing)

Once a public HTTPS URL exists, each coworker onboards themselves — no shared secret, no local install, no config file on Claude Desktop.

**Claude Desktop:** Settings → Integrations → **Add custom connector** → paste `https://<url>/mcp` → Add. First tool call triggers Google sign-in.

**Claude.ai (web):** Settings → Connectors → same flow.

**Cursor / Windsurf:** paste the following into their MCP settings (this is the JSON from upstream README and works natively there):
```json
{
  "mcpServers": {
    "google-docs": {
      "type": "streamableHttp",
      "url": "https://<url>/mcp"
    }
  }
}
```

**Claude Code CLI:**
```
claude mcp add google-docs --transport http https://<url>/mcp
```

Note: `claude_desktop_config.json` is **stdio-only** — pasting the JSON above into that file does not work for Claude Desktop. Use the Custom Connectors UI instead.

## Troubleshooting cheatsheet

| Symptom during sign-in               | Cause                                                 | Fix                                                       |
| ------------------------------------ | ----------------------------------------------------- | --------------------------------------------------------- |
| `Error 400: redirect_uri_mismatch`   | Callback URL not in client's Authorised redirect URIs | Add exact `${BASE_URL}/oauth/callback`                    |
| "Access blocked: app not verified"   | External + Production consent screen, sensitive scope | Switch to Internal (Workspace) or add account as test user |
| "Something went wrong" (generic)     | Missing scopes on consent screen OR APIs not enabled  | Authorise all scopes; enable Docs/Sheets/Drive/Gmail/Cal  |
| `No sessionId` at `/mcp` in browser  | Not an error — MCP needs a proper session init        | Ignore; use an MCP client                                 |
| Sessions drop across server restart  | `JWT_SIGNING_KEY` auto-generated per boot             | Pin a fixed key via env var / Secrets Manager             |
