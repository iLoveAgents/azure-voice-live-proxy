# Azure Voice Live Proxy

**Secure WebSocket proxy server for Azure Voice Live API with comprehensive authentication support.**

Generic proxy that handles Voice, Avatar, and Agent Service scenarios with both API key and MSAL token authentication.

## Why Use This Proxy?

### Security Problem: API Keys in Browser

**Never embed API keys in browser code:**
- API keys in `.env` files get bundled into browser JavaScript
- Anyone can inspect network tab and steal your keys
- Keys can be extracted from source code

### Solution: Backend Proxy

This proxy secures your API keys server-side and supports multiple authentication methods:

| Authentication Method | Security | Auditing | Use Case |
|----------------------|----------|----------|----------|
| API keys in browser | Insecure | No | Quick demos only |
| Backend Proxy + API Key | Good | No | Production (shared access) |
| Backend Proxy + MSAL | Best | Yes | Enterprise (user-level auth) |

## Features

- **All Azure Voice Live Scenarios**: Voice chat, Avatar video, Agent Service
- **Multiple Auth Methods**: API key (shared) or MSAL tokens (per-user)
- **Zero Configuration**: Works out of the box with environment variables
- **Transparent Proxy**: Bidirectional WebSocket message forwarding
- **Production Ready**: Clean error handling and logging
- **Lightweight**: Only 148 lines of code, 2 dependencies

## Installation

```bash
npm install @iloveagents/azure-voice-live-proxy
```

Or run directly with npx:

```bash
npx @iloveagents/azure-voice-live-proxy
```

## Quick Start

### 1. Configure Environment

Create `.env` file:

```bash
# Required
AZURE_AI_FOUNDRY_RESOURCE=your-resource-name

# For Voice/Avatar with API key
AZURE_SPEECH_KEY=your-api-key

# For Agent Service (optional)
AGENT_ID=your-agent-id
PROJECT_NAME=your-project-name

# Server config (optional)
PORT=8080
API_VERSION=2025-10-01
```

### 2. Start Proxy Server

```bash
# Option 1: Using npx
npx @iloveagents/azure-voice-live-proxy

# Option 2: After npm install
npm start

# Option 3: Development mode with auto-reload
npm run dev
```

### 3. Connect from Browser

**Voice/Avatar with API Key (Anonymous):**
```typescript
import { useVoiceLive, createVoiceLiveConfig } from '@iloveagents/azure-voice-live-react';

const config = createVoiceLiveConfig('default', {
  connection: {
    customWebSocketUrl: 'ws://localhost:8080?mode=standard&model=gpt-realtime',
  }
});

const { connect, videoStream } = useVoiceLive(config);
```

**Voice/Avatar with MSAL Token (User-level):**
```typescript
const token = await msalInstance.acquireTokenSilent({
  scopes: ['https://cognitiveservices.azure.com/.default']
});

const config = createVoiceLiveConfig('default', {
  connection: {
    customWebSocketUrl: `ws://localhost:8080?mode=standard&model=gpt-realtime&token=${token.accessToken}`,
  }
});
```

**Agent Service with MSAL Token:**
```typescript
const token = await msalInstance.acquireTokenSilent({
  scopes: ['https://ai.azure.com/.default']
});

const config = createVoiceLiveConfig('agent', {
  connection: {
    customWebSocketUrl: `ws://localhost:8080?mode=agent&token=${token.accessToken}`,
  }
});
```

## Usage Scenarios

### Voice & Avatar - Option 1: API Key (Shared Access)

**Use when:** Quick demos, internal tools, shared access OK

**Frontend:**
```typescript
customWebSocketUrl: 'ws://localhost:8080?mode=standard&model=gpt-realtime'
```

**Backend (.env):**
```bash
AZURE_AI_FOUNDRY_RESOURCE=your-resource
AZURE_SPEECH_KEY=your-api-key  # Secured server-side
```

**Benefits:**
- Simple setup
- No user authentication needed
- Good for internal applications

### Voice & Avatar - Option 2: MSAL Token (User-Level Auth)

**Use when:** Enterprise apps, need per-user auditing, SSO integration

**Frontend:**
```typescript
const token = await msalInstance.acquireTokenSilent({
  scopes: ['https://cognitiveservices.azure.com/.default']
});

customWebSocketUrl: `ws://localhost:8080?mode=standard&model=gpt-realtime&token=${token.accessToken}`
```

**Backend (.env):**
```bash
AZURE_AI_FOUNDRY_RESOURCE=your-resource
# No API key needed - uses user's MSAL token
```

**Benefits:**
- No API keys stored anywhere
- Each user authenticated individually
- Tokens auto-expire (1 hour)
- Works with Conditional Access policies
- Enterprise SSO support

**Setup required:**
1. Azure App Registration with scope: `https://cognitiveservices.azure.com/.default`
2. Assign "Cognitive Services User" role on AI Foundry resource
3. Install `@azure/msal-react` and configure MsalProvider

### Agent Service (MSAL Required)

**Frontend:**
```typescript
const token = await msalInstance.acquireTokenSilent({
  scopes: ['https://ai.azure.com/.default']
});

customWebSocketUrl: `ws://localhost:8080?mode=agent&token=${token.accessToken}`
```

**Backend (.env):**
```bash
AZURE_AI_FOUNDRY_RESOURCE=your-resource
AGENT_ID=your-agent-id
PROJECT_NAME=your-project
```

**Note:** Agent Service ONLY supports MSAL authentication (no API key option)

## Production Deployment

### Option 1: PM2 (Node.js Process Manager)

```bash
npm install -g pm2
pm2 start index.js --name azure-voice-proxy
pm2 save
pm2 startup
```

### Option 2: Docker

Create `Dockerfile`:

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
EXPOSE 8080
CMD ["node", "index.js"]
```

Build and run:

```bash
docker build -t azure-voice-proxy .
docker run -p 8080:8080 --env-file .env azure-voice-proxy
```

### Option 3: Azure Container Apps

```bash
az containerapp create \
  --name azure-voice-proxy \
  --resource-group myResourceGroup \
  --environment myEnvironment \
  --image ghcr.io/iloveagents/azure-voice-live-proxy:latest \
  --target-port 8080 \
  --ingress external \
  --env-vars \
    AZURE_AI_FOUNDRY_RESOURCE=your-resource \
    AZURE_SPEECH_KEY=your-key
```

### Option 4: Cloudflare Workers

Not recommended for this proxy due to WebSocket limitations. Use Node.js-based deployment instead.

## Security Best Practices

1. **Never commit `.env`** - add to `.gitignore`
2. **Use WSS in production** - secure WebSocket (TLS/SSL)
3. **Add rate limiting** - prevent abuse
4. **Validate tokens** - especially for agent mode
5. **Use environment-specific configs** - dev/staging/prod
6. **Enable CORS restrictions** - limit allowed origins
7. **Monitor usage** - track API consumption
8. **Rotate keys regularly** - automated key rotation

## Configuration Reference

### Environment Variables

| Variable | Required | Description | Default |
|----------|----------|-------------|---------|
| `PORT` | No | Server port | 8080 |
| `API_VERSION` | No | Azure API version | 2025-10-01 |
| `AZURE_AI_FOUNDRY_RESOURCE` | Yes | Azure resource name | - |
| `AZURE_SPEECH_KEY` | Conditional | API key for Voice/Avatar | - |
| `AGENT_ID` | No | Agent Service ID | - |
| `PROJECT_NAME` | No | Agent project name | - |

### Query Parameters

| Parameter | Required | Description | Example |
|-----------|----------|-------------|---------|
| `mode` | No | Connection mode | `standard` or `agent` |
| `model` | No | Model name | `gpt-realtime` |
| `token` | Conditional | MSAL access token | From Azure AD |
| `agentId` | No | Override AGENT_ID from env | Custom agent |
| `projectName` | No | Override PROJECT_NAME from env | Custom project |

## Troubleshooting

**Connection fails:**
- Verify `.env` has correct values
- Check proxy is running: `curl http://localhost:8080`
- Check backend logs for errors

**"Missing token parameter":**
- Agent mode requires MSAL token in query param
- Standard mode doesn't need token if AZURE_SPEECH_KEY is set

**"AZURE_SPEECH_KEY required":**
- Standard mode needs API key in backend .env OR MSAL token from client
- Make sure you copied .env.example to .env

**Double audio playback:**
- This is a client-side issue, not proxy related
- Check `@iloveagents/azure-voice-live-react` documentation

## Examples

See the [@iloveagents/azure-voice-live-react](https://github.com/iLoveAgents/azure-voice-live-react) library for complete frontend examples:

- Voice Chat - Simple
- Voice Chat - Advanced
- Voice Chat - Secure Proxy (API Key)
- Voice Chat - Secure Proxy (MSAL)
- Avatar - Simple
- Avatar - Advanced
- Avatar - Secure Proxy (API Key)
- Avatar - Secure Proxy (MSAL)
- Agent Service with MSAL

## Contributing

Contributions welcome! Please see [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines.

## License

MIT © [iloveagents](https://github.com/iloveagents)

## Support

- **Issues**: [GitHub Issues](https://github.com/iLoveAgents/azure-voice-live-proxy/issues)
- **React Library**: [@iloveagents/azure-voice-live-react](https://github.com/iLoveAgents/azure-voice-live-react)
- **Azure Docs**: [Azure Voice Live](https://learn.microsoft.com/azure/ai-services/speech-service/voice-live)

---

**Built by [iloveagents](https://github.com/iloveagents)**
