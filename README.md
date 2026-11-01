# Solana Detective

A dark, responsive Solana token-scanner starter built with Next.js App Router. It streams new Pump.fun mints through PumpPortal, enriches tokens from DexScreener and Solana RPC, and keeps missing risk signals explicitly unverified.

> **Important:** this tool is an information dashboard, not an audit or a trading system. It does not submit transactions, hold keys, or guarantee that a token is safe. Risk checks are incomplete when provider data is unavailable or not configured.

## Requirements

- Node.js 18.17+ and npm
- Optional: a Solana RPC provider, Telegram bot, X API bearer token, and a FOMO Family JSON endpoint

## Local setup

```powershell
npm install
Copy-Item .env.example .env.local
# Add any provider credentials to .env.local
npm run dev
```

Open [http://localhost:3000](http://localhost:3000). Without credentials, PumpPortal and the public Solana RPC are used where possible. Telegram alerts and X search stay disabled until configured.

## Data sources and coverage

| Capability | Provider | Notes |
|---|---|---|
| Live new-token events | PumpPortal WebSocket | Browser subscribes to Pump.fun token creation and recent token trades. This requires an active browser tab. |
| Initial token list | Pump.fun live-token endpoint | Used as a bootstrap and polled every 30 seconds; the WebSocket is the live path. Provider availability/rate limits may change. |
| SOL pair price, liquidity, volume and 1h change | DexScreener | Only Solana pairs quoted in SOL are considered. Detail data is fetched lazily for recent tokens. |
| Mint/freeze authorities | Solana JSON-RPC | Parsed from the SPL mint account when RPC data is available. Set `SOLANA_RPC_URL` to a reliable provider for production. |
| Whale wallet qualification | Solana RPC + CoinGecko SOL/USD | Checks the wallet's **native SOL balance**, not total token/NFT portfolio value. Buy events are observed for subscribed recent tokens; whale observations live in browser memory. |
| FOMO Family | Configured API endpoint | Optional. Set `FOMO_FAMILY_API_URL`; the endpoint must return a token array or `{ "tokens": [...] }`. Each entry must include a Solana chain, SOL quote symbol, and address/mint. |
| X mentions | X API v2 recent search | Requires `X_BEARER_TOKEN` and an X API plan with recent-search access. Sentiment is a simple keyword estimate, not a trained model. Accounts with at least 10K followers are marked in the detail view. |
| Telegram notifications | Telegram Bot API | Requires `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID`; notifications are sent only when the user enables alerts in the dashboard. |

Several requested risk checks need additional indexed data that is not available from the current free/provider-neutral integrations: top-holder aggregation, verified liquidity locks/burns, developer/rug history, coordinated launch wallets, and connected-wallet attribution. These checks remain **Unverified**. The app deliberately keeps such tokens in “Needs review”; it will not classify a token as “Verified clear” unless every configured risk signal is checked and clear.

Whale qualification uses a $20,000 native SOL balance threshold, with a 30-second server-process cache. Three distinct qualifying wallets buying the same watched token within five minutes produce a whale alert. Serverless instance restarts and multiple instances do not share this ephemeral cache.

## Environment variables

Copy `.env.example` to `.env.local` for local work and configure values in your host's encrypted environment-variable settings for deployment.

| Variable | Required | Purpose |
|---|---:|---|
| `SOLANA_RPC_URL` | No | HTTPS JSON-RPC URL; defaults to Solana's public mainnet endpoint |
| `TELEGRAM_BOT_TOKEN` | For Telegram | Secret bot token from `@BotFather` |
| `TELEGRAM_CHAT_ID` | For Telegram | Destination chat or group ID |
| `X_BEARER_TOKEN` | For X search | X API v2 bearer token |
| `FOMO_FAMILY_API_URL` | No | Trusted HTTPS endpoint returning the documented token-array shape |

Never put private keys, bot credentials, or API bearer tokens in frontend variables (for example `NEXT_PUBLIC_*`).

## API

All routes are Next.js route handlers. Responses use JSON. Limits are process-local best-effort protections; configure platform-level rate limiting or a shared store for multi-instance production.

### `GET /api/tokens/new`

Returns `{ tokens, status, errors }`. Polls Pump.fun's live-token feed and, if configured, the FOMO Family endpoint. Only valid Solana addresses and SOL-quoted FOMO pairs are included.

### `GET /api/tokens/{address}`

Validates the address, then returns the best-liquidity SOL-quoted DexScreener pair, available market metrics, mint/freeze authority signals, and a risk assessment. Provider failures produce an explicit error or unverified signal.

### `GET /api/whales/tracking?wallet={address}`

Checks the wallet's native SOL balance in USD using Solana RPC and CoinGecko. Returns `{ wallet, balanceUsd, cached }`. The current deployment model does not persist trade history server-side; recent watched-token whale observations are held in the browser session.

### `GET /api/twitter/mentions?q={term}`

Searches recent X posts. Requires `X_BEARER_TOKEN`; responds with `{ mentions, count }`. Recent-search availability depends on the X API plan.

### `POST /api/alerts/telegram`

Sends a notification using server-side Telegram credentials. JSON body:

```json
{
  "type": "whale",
  "name": "Example",
  "symbol": "EX",
  "address": "valid-base58-mint-address",
  "riskLevel": "medium",
  "score": 50,
  "whaleCount": 3
}
```

Supported `type` values: `no-risk`, `whale`, `risk-change`. The route validates the mint and escapes token text before sending HTML.

## Deploy to Vercel

1. Push the project to a Git provider and import it into Vercel.
2. Select the Next.js preset and leave the build command as `npm run build`.
3. Add the desired variables from `.env.example` in Project Settings → Environment Variables. Use an RPC provider for consistent production data.
4. Deploy. The interface is served as a Next.js app; its API routes run as serverless functions.

Netlify can also host the project using its Next.js runtime/plugin. Configure the same environment variables and build command. Live WebSocket events connect from the user's browser; serverless functions are not expected to keep a permanent socket open.

For production scale, add a shared rate limiter/cache (for example a host-provided KV/Redis), durable whale-event storage, monitoring, and indexed/on-chain data providers for holder and developer analysis.

## Development checks

```powershell
npm run typecheck
npm run lint
npm run build
```

The interface contains no wallet connection or transaction signing functionality.
