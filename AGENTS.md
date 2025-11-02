# AGENTS.md

Agent-focused documentation for the Azure Voice Live Proxy project.

## Project Overview

**Purpose:** Secure WebSocket proxy server for Azure Voice Live API (Voice, Avatar, Agent Service). Keeps credentials server-side, enforces security (CORS, rate limits, headers), and forwards messages transparently.

**Tech Stack:**

- TypeScript (strict mode)
- Node.js 18+
- Express with WebSocket support
- Azure Application Insights (optional)
- Docker + docker-compose

**Related Projects:**

- Frontend React library: <https://github.com/iLoveAgents/azure-voice-live-react>

## Setup Commands

```bash
# Install dependencies
npm install

# Development (build + watch)
npm run dev

# Production build
npm run build

# Start production server
npm start

# Type checking only
npm run type-check

# Clean build artifacts
npm run clean
```

## Environment Configuration

Required env vars (copy from `.env.example`):

```bash
AZURE_AI_FOUNDRY_RESOURCE=your-resource-name  # Required
AZURE_SPEECH_KEY=your-api-key                 # Optional if using MSAL
AGENT_ID=your-agent-id                        # Optional for Agent mode
PROJECT_NAME=your-project-name                # Optional for Agent mode
```

Security and telemetry (optional):

```bash
ALLOWED_ORIGINS=http://localhost:3000
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX_REQUESTS=100
MAX_CONNECTIONS=1000
APPLICATIONINSIGHTS_CONNECTION_STRING=...     # Optional telemetry
```

## Code Style Guidelines

**TypeScript:**

- Strict mode enabled (`tsconfig.json`)
- Full type safety - no `any` types
- Use interfaces for shape definitions
- Prefer `const` over `let`, avoid `var`

**Formatting:**

- Use the existing ESM module format (`import`/`export`)
- Maintain existing indentation (2 spaces)
- Keep functions focused and single-purpose

**Architecture:**

- `src/index.ts`: Main server, routes, middleware
- `src/types.ts`: All TypeScript interfaces and types
- Keep security middleware at the top of the request chain

## Build Instructions

The project uses TypeScript compilation:

1. **Development:** `npm run dev` - builds and watches for changes
2. **Production:** `npm run build` - outputs to `dist/` folder
3. **Output:** Compiled JS and declaration files in `dist/`

**Important:** The `dist/` folder is gitignored. Always rebuild after pulling changes.

## Testing Instructions

Currently no automated tests. When adding tests:

- Use a test framework like Vitest or Jest
- Place tests in `__tests__/` or co-located as `*.test.ts`
- Mock WebSocket connections
- Test authentication flows (API key vs MSAL)
- Test CORS, rate limiting, connection limits

## Docker

**Local build:**

```bash
npm run docker:build
npm run docker:run
```

**docker-compose (recommended):**

```bash
cp .env.example .env
# Edit .env with your values
docker-compose up -d
docker-compose logs -f
docker-compose down
```

**Image details:**

- Multi-stage build (build stage + runtime stage)
- Alpine-based for minimal size
- Non-root user for security
- Health check on `/health` endpoint

## Security Considerations

**Critical:**

- Never commit `.env` files (already in `.gitignore`)
- API keys must stay server-side only
- Use MSAL tokens for per-user auth in production

**Built-in protections:**

- CORS origin validation
- IP-based rate limiting
- Helmet.js security headers
- Connection limits to prevent DoS

**Production checklist:**

- Set `ALLOWED_ORIGINS` to specific domains (no `*`)
- Use WSS (TLS) via reverse proxy
- Enable Application Insights for monitoring
- Rotate API keys regularly if using shared key mode

## Authentication Modes

**Standard Mode (Voice/Avatar):**

- Option 1: API key in `.env` (shared access)
- Option 2: MSAL token from client (per-user auth)
- Scope: `https://ai.azure.com/.default`

**Agent Mode:**

- MSAL token required (no API key option)
- Requires `AGENT_ID` and `PROJECT_NAME` in env
- Scope: `https://ai.azure.com/.default`

## Deployment

**Azure Container Apps:**

```bash
az containerapp create \
  --name azure-voice-proxy \
  --resource-group myRG \
  --environment myEnv \
  --image your-image:latest \
  --target-port 8080 \
  --ingress external \
  --env-vars AZURE_AI_FOUNDRY_RESOURCE=... AZURE_SPEECH_KEY=...
```

**Azure App Service (Linux Container):**

```bash
az webapp create \
  --resource-group myRG \
  --plan myPlan \
  --name azure-voice-proxy \
  --deployment-container-image-name your-image:latest
```

**PM2 (Node.js):**

```bash
npm install -g pm2
pm2 start dist/index.js --name azure-voice-proxy
pm2 save
pm2 startup
```

## Common Issues

**Build fails:**

- Ensure Node.js 18+ is installed
- Run `npm clean-install` to reset dependencies
- Check `tsconfig.json` for syntax errors

**WebSocket connection fails:**

- Verify proxy is running: `curl http://localhost:8080/health`
- Check `.env` has correct `AZURE_AI_FOUNDRY_RESOURCE`
- Review browser console for CORS errors
- Ensure origin is in `ALLOWED_ORIGINS`

**"Missing token parameter":**

- Agent mode requires MSAL token in query string
- Standard mode can use API key OR MSAL token

**Rate limit errors:**

- Adjust `RATE_LIMIT_MAX_REQUESTS` or `RATE_LIMIT_WINDOW_MS`
- Default: 100 requests per 60 seconds per IP

## File Structure

```plaintext
.
├── src/
│   ├── index.ts          # Main server, routes, WebSocket proxy
│   └── types.ts          # TypeScript type definitions
├── dist/                 # Compiled output (gitignored)
├── .env.example          # Environment template
├── Dockerfile            # Multi-stage Docker build
├── docker-compose.yml    # Docker compose config
├── tsconfig.json         # TypeScript configuration
├── package.json          # Dependencies and scripts
└── README.md             # Human-readable documentation
```

## Additional Notes

**Logging:**

- High-frequency messages (audio buffers, deltas) are filtered from console logs
- Application Insights captures all events if configured
- Console logs include direction markers: `Browser → Azure`, `Azure → Browser`

**Performance:**

- Built for high concurrency (default: 1000 concurrent connections)
- Uses native WebSocket forwarding (minimal overhead)
- Express async handlers for non-blocking I/O

**Monitoring:**

- `/health` endpoint returns active connections and server status
- Application Insights tracks: `ServerStarted`, `WebSocketConnected`, `WebSocketError`, `activeConnections` metric

---

Made with 💜 [iLoveAgents](https://iloveagents.ai)
