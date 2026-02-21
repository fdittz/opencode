# External Integrations

**Analysis Date:** 2026-02-21

## AI/LLM Providers

All providers are integrated via the Vercel AI SDK v5 (`ai` package) with provider-specific adapter packages. Configuration and initialization happens in `packages/opencode/src/provider/provider.ts`.

**Anthropic:**

- SDK: `@ai-sdk/anthropic`
- Auth: `ANTHROPIC_API_KEY` env var
- Features: Claude models, prompt caching, PDF support, thinking/extended-thinking

**OpenAI:**

- SDK: `@ai-sdk/openai`
- Auth: `OPENAI_API_KEY` env var
- Features: GPT/o-series models, reasoning effort, web search tool

**Azure OpenAI:**

- SDK: `@ai-sdk/azure`
- Auth: `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_RESOURCE_NAME` env vars
- Features: Azure-hosted OpenAI models

**Google (Gemini):**

- SDK: `@ai-sdk/google`
- Auth: `GOOGLE_GENERATIVE_AI_API_KEY` or `GEMINI_API_KEY` env vars
- Features: Gemini models, thinking, grounding via Google Search

**Google Vertex AI:**

- SDK: `@ai-sdk/google-vertex`
- Auth: Google Cloud ADC (Application Default Credentials), `GOOGLE_VERTEX_PROJECT`/`GOOGLE_VERTEX_LOCATION` env vars
- Features: Vertex-hosted Gemini models

**AWS Bedrock:**

- SDK: `@ai-sdk/amazon-bedrock`
- Auth: AWS credentials (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`) or SSO profile
- Features: Bedrock-hosted models (Anthropic, etc.), thinking support

**xAI (Grok):**

- SDK: `@ai-sdk/xai`
- Auth: `XAI_API_KEY` env var

**Mistral:**

- SDK: `@ai-sdk/mistral`
- Auth: `MISTRAL_API_KEY` env var

**Groq:**

- SDK: `@ai-sdk/groq`
- Auth: `GROQ_API_KEY` env var

**DeepInfra:**

- SDK: `@ai-sdk/openai` (OpenAI-compatible)
- Auth: `DEEPINFRA_API_KEY` env var
- Base URL: `https://api.deepinfra.com/v1/openai`

**Cerebras:**

- SDK: `@ai-sdk/cerebras`
- Auth: `CEREBRAS_API_KEY` env var

**Cohere:**

- SDK: `@ai-sdk/cohere`
- Auth: `COHERE_API_KEY` env var

**TogetherAI:**

- SDK: `@ai-sdk/togetherai`
- Auth: `TOGETHER_API_KEY` env var

**Perplexity:**

- SDK: `@ai-sdk/perplexity`
- Auth: `PERPLEXITY_API_KEY` env var

**OpenRouter:**

- SDK: `@ai-sdk/openai` (OpenAI-compatible)
- Auth: `OPENROUTER_API_KEY` env var
- Base URL: `https://openrouter.ai/api/v1`

**Vercel:**

- SDK: `@ai-sdk/openai` (OpenAI-compatible)
- Auth: `VERCEL_API_KEY` env var
- Base URL: `https://api.vercel.ai/v1/`

**GitHub Copilot:**

- SDK: `@ai-sdk/openai` (OpenAI-compatible)
- Auth: `GITHUB_COPILOT_TOKEN` (obtained via device flow OAuth or VS Code token)
- Base URL: `https://api.githubcopilot.com`
- Special handling: Device auth flow in `packages/opencode/src/provider/copilot.ts`

**GitHub Copilot Enterprise:**

- SDK: `@ai-sdk/openai` (OpenAI-compatible)
- Auth: Same as Copilot
- Base URL: `https://api.githubcopilot.com` with `/chat/completions` endpoint

**GitLab:**

- SDK: `@ai-sdk/openai` (OpenAI-compatible)
- Auth: `GITLAB_API_KEY` env var
- Base URL: `https://gitlab.com/api/v4/ai/proxy/v1/`

**Cloudflare Workers AI:**

- SDK: `@ai-sdk/openai` (OpenAI-compatible)
- Auth: `CLOUDFLARE_API_KEY` env var
- Base URL: `https://api.cloudflare.com/client/v4/accounts/{CLOUDFLARE_ACCOUNT_ID}/ai/v1`

**Cloudflare AI Gateway:**

- SDK: `@ai-sdk/openai` (OpenAI-compatible)
- Auth: `CLOUDFLARE_AI_GATEWAY_API_KEY` env var
- Base URL: `https://gateway.ai.cloudflare.com/v1/{CLOUDFLARE_ACCOUNT_ID}/{CLOUDFLARE_AI_GATEWAY_ID}`

**SAP AI Core:**

- SDK: `@ai-sdk/openai` (OpenAI-compatible)
- Auth: OAuth2 client credentials (`SAP_AI_CORE_CLIENT_ID`, `SAP_AI_CORE_CLIENT_SECRET`, `SAP_AI_CORE_TOKEN_URL`)
- Base URL: `SAP_AI_CORE_BASE_URL` + `/v2/inference/deployments/{SAP_AI_CORE_DEPLOYMENT_ID}`

**Custom OpenAI-Compatible:**

- SDK: `@ai-sdk/openai`
- Auth: Configurable via `baseURL` and `apiKey` in user config
- Allows any OpenAI-compatible endpoint

## Protocol Integrations

**MCP (Model Context Protocol):**

- Client: `@modelcontextprotocol/sdk` 1.12
- Implementation: `packages/opencode/src/mcp/index.ts`
- Transports: stdio, SSE (Server-Sent Events), StreamableHTTP
- OAuth support for authenticated MCP servers
- Manages tool discovery, lifecycle, reconnection

**ACP (Agent Client Protocol):**

- Client: `@agentclientprotocol/sdk`
- Implementation: `packages/opencode/src/acp/index.ts`
- Purpose: Agent-to-agent communication protocol

**LSP (Language Server Protocol):**

- Client: `vscode-jsonrpc` (JSON-RPC over stdio)
- Implementation: `packages/opencode/src/lsp/`
- Purpose: Language intelligence (diagnostics, hover, completions) for TUI
- Connects to user's language servers defined in config

## Data Storage

**Databases:**

- SQLite (CLI local storage)
  - Driver: `bun:sqlite` (built into Bun runtime)
  - ORM: `drizzle-orm` with `drizzle-orm/bun-sqlite`
  - Config: `packages/opencode/drizzle.config.ts`
  - Schema files: `packages/opencode/src/**/*.sql.ts` (session, message, part, todo, permission, project, share, control tables)
  - Mode: WAL (Write-Ahead Logging), `journal_mode=WAL`, `synchronous=NORMAL`
  - Location: `~/.local/share/opencode/{project-hash}/db.sqlite`
  - Implementation: `packages/opencode/src/storage/db.ts`

- PlanetScale MySQL (console/cloud)
  - Driver: `@planetscale/database`
  - ORM: `drizzle-orm` with `drizzle-orm/planetscale-serverless`
  - Config: `packages/console/core/drizzle.config.ts`
  - Schema files: `packages/console/core/src/**/*.sql.ts`
  - Connection: `DATABASE_URL` env var (provided by SST PlanetScale component)
  - Branching: One database branch per SST stage
  - Implementation: `packages/console/core/src/drizzle/index.ts`

**File Storage:**

- Cloudflare R2
  - Used for: Session sharing, enterprise file storage
  - Buckets defined in `infra/app.ts` (share bucket) and `infra/enterprise.ts` (storage bucket)
  - Access: Via Cloudflare Workers bindings or S3-compatible API
  - Secret keys: `infra/secret.ts` (`R2AccessKeyId`, `R2SecretAccessKey`)

**Caching:**

- Cloudflare KV
  - Used for: Auth session storage (OpenAuth), configuration
  - Namespaces defined in `infra/console.ts` (AuthKV)
  - Bound to workers via SST linkable resources

**Durable Objects:**

- Cloudflare Durable Objects
  - Used for: Real-time sync (SyncServer)
  - Implementation: `packages/function/src/api.ts` (SyncServer class)
  - Purpose: Coordinates state between CLI clients and cloud services

## Authentication & Identity

**Auth Provider (Cloud):**

- OpenAuth (`@openauthjs/openauth`)
  - Implementation: `packages/console/function/src/auth.ts`
  - Subjects: "account" (with accountID, email properties, Zod-validated)
  - OAuth providers:
    - GitHub OAuth (`GitHubAdapter`) - `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`
    - Google OIDC (`GoogleOidcAdapter`) - `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`
  - Session storage: Cloudflare KV (DynamoStorage adapter with KV)
  - Deployed as Cloudflare Worker via SST

**Auth (CLI):**

- Implementation: `packages/opencode/src/auth/index.ts`
  - OAuth flow for cloud services (OpenAuth client)
  - API key authentication
  - WellKnown discovery for auth endpoints
  - Device flow for GitHub Copilot tokens

**GitHub App Auth:**

- JWT creation via `jose` library
  - Implementation: `packages/function/src/api.ts`
  - Signs JWTs with GitHub App private key for installation access tokens
  - Used for GitHub API operations on behalf of users

## APIs & External Services

**GitHub:**

- REST API: `@octokit/rest` (`packages/function`)
- GraphQL API: `@octokit/graphql` (`packages/function`)
- Auth: GitHub App (JWT + installation tokens) and user OAuth tokens
- Used for: Repository operations, PR management, issue management, code search
- Webhooks: Incoming GitHub webhooks processed in `packages/function/src/api.ts`
- Implementation: `packages/function/src/github.ts`

**Stripe:**

- SDK: `stripe` 18.x (`packages/console/core`)
- Auth: `STRIPE_SECRET_KEY` (SST secret)
- Used for: Subscription management, pricing, billing
- Products/prices defined in SST config: `infra/console.ts`
- Webhooks: Incoming Stripe webhooks for subscription events
- Implementation: `packages/console/core/src/stripe/index.ts`

**Discord:**

- REST API (direct HTTP, no SDK)
- Auth: Bot token and webhook tokens (`DISCORD_BOT_TOKEN`, per-channel webhook URLs)
- Used for: Support ticket creation, notification forwarding
- Secrets: `infra/app.ts` (DiscordBotToken, DiscordSupportChannelWebhook, etc.)

**Feishu/Lark:**

- REST API (direct HTTP)
- Auth: Tenant access token via `FEISHU_APP_ID`, `FEISHU_APP_SECRET`
- Used for: Message forwarding, support integration
- Secrets: `infra/app.ts` (FeishuAppId, FeishuAppSecret, FeishuGroupChatId)

**Slack:**

- SDK: `@slack/bolt` 4.x (`packages/slack`)
- Auth: Bot token, signing secret, app token
- Used for: Slack bot integration
- Implementation: `packages/slack/src/index.ts`

**EmailOctopus:**

- REST API (direct HTTP)
- Auth: API key (`EMAILOCTOPUS_API_KEY`)
- Used for: Newsletter subscriptions
- Secrets: `infra/app.ts`

**AWS SES:**

- SDK: `@aws-sdk/client-sesv2` (`packages/console/mail`)
- Auth: AWS credentials (via SST/Cloudflare Worker environment)
- Used for: Transactional email sending
- Implementation: `packages/console/mail/src/index.ts`

**models.dev:**

- REST API (direct HTTP fetch)
- URL: `https://models.dev/data/models.json`
- Auth: None (public API)
- Used for: AI model registry/metadata, fetched at build time and embedded in CLI binary
- Implementation: `packages/opencode/script/build.ts`

## Monitoring & Observability

**Error Tracking:**

- None detected (no Sentry, Bugsnag, etc.)

**Logs:**

- Cloudflare Workers: Log tailing via Cloudflare Logpush
  - Log processor worker defined in `infra/app.ts` (TailConsumer)
  - Processes logs from API worker and forwards to Honeycomb
- CLI: Console-based logging, structured output in TUI
- Honeycomb integration for cloud log analysis (`infra/app.ts` → HoneycombApiKey secret)

**Analytics:**

- Not detected in codebase

## CI/CD & Deployment

**Hosting:**

- Cloudflare (primary cloud platform)
  - Workers: API, Auth, Console, Enterprise, Functions, Log Processor, Mail
  - Static Sites / Pages: Web app, Docs site
  - R2: File storage
  - KV: Session/config storage
  - Durable Objects: Sync server

**Infrastructure-as-Code:**

- SST v3 (Ion) with Cloudflare home
  - Config: `sst.config.ts`
  - Infra modules: `infra/app.ts`, `infra/console.ts`, `infra/enterprise.ts`, `infra/secret.ts`, `infra/stage.ts`
  - Providers: Cloudflare, Stripe, PlanetScale
  - Stage-based deployment (production, dev, PR-based stages)

**CI Pipeline:**

- Not directly visible in repo (no `.github/workflows` explored), but SST supports CLI-driven deploys
- Turborepo for build orchestration (`turbo build`, `turbo typecheck`)

**CLI Distribution:**

- Standalone Bun binaries compiled for 11 platform targets
- Build script: `packages/opencode/script/build.ts`
- Targets: linux-x64-glibc, linux-x64-musl, linux-x64-glibc-baseline, linux-arm64-glibc, linux-arm64-musl, darwin-x64, darwin-arm64, win32-x64, win32-x64-baseline, win32-arm64
- Nix package also available via `flake.nix`

## Environment Configuration

**Required env vars (CLI):**

- At least one AI provider API key (e.g., `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`)
- No other env vars strictly required for local CLI usage

**Required env vars (Cloud/Console):**

- `DATABASE_URL` - PlanetScale connection string (via SST)
- `STRIPE_SECRET_KEY` - Stripe API key (via SST secret)
- `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET` - OAuth for console login
- `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` - OIDC for console login
- `GITHUB_APP_ID`, `GITHUB_APP_PRIVATE_KEY` - GitHub App for API operations
- R2 access keys via `infra/secret.ts`

**SST Secrets (defined in infra/):**

- `infra/console.ts`: StripeSecretKey, GithubClientId, GithubClientSecret, GoogleClientId, GoogleClientSecret, GithubAppId, GithubAppPrivateKey, GithubAppClientId, GithubAppClientSecret
- `infra/app.ts`: GithubAppId, GithubAppPrivateKey, DiscordBotToken, DiscordSupportChannelWebhook, DiscordSupportThreadWebhook, DiscordSalesChannelWebhook, FeishuAppId, FeishuAppSecret, FeishuGroupChatId, HoneycombApiKey, EmailOctopusApiKey

**Secrets location:**

- SST secrets (encrypted, stored in SST state)
- Cloudflare Worker environment bindings (injected at deploy time)
- Local `.env` files for development (not committed)

## Webhooks & Callbacks

**Incoming:**

- GitHub App webhooks → `packages/function/src/api.ts` (repository events, PR events, etc.)
- Stripe webhooks → `packages/console/core/src/stripe/index.ts` (subscription lifecycle events)
- Cloudflare Worker tail logs → Log processor worker in `infra/app.ts`

**Outgoing:**

- Discord webhook notifications (support tickets, sales alerts) → `infra/app.ts` webhook URLs
- Feishu/Lark message forwarding → Feishu REST API
- EmailOctopus newsletter subscription calls

---

_Integration audit: 2026-02-21_
