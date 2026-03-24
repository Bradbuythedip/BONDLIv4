# Bondli API Reference

Complete REST API reference for the Bondli platform. Base URL: `https://bondli.fun`

---

## Table of Contents

- [Authentication](#authentication)
- [Health & Status](#health--status)
- [GPU Endpoints](#gpu-endpoints)
- [Nosana / AI Inference](#nosana--ai-inference)
- [Token Radar & Scoring](#token-radar--scoring)
- [Trading](#trading)
- [Auto-Trade Engine](#auto-trade-engine)
- [Session Management](#session-management)
- [Fleet Trading](#fleet-trading)
- [Intelligence](#intelligence)
- [Access & Tiers](#access--tiers)
- [Chat (AI Support)](#chat-ai-support)
- [Utilities](#utilities)
- [Error Handling](#error-handling)
- [Rate Limits](#rate-limits)

---

## Authentication

Bondli uses **wallet-based authentication**. Most endpoints require a Solana wallet address.

### Headers

| Header | Required | Description |
|--------|----------|-------------|
| `X-Wallet` | Yes (most endpoints) | Your Solana wallet address |
| `X-API-Secret` | Admin only | Admin API secret for privileged operations |

### Tier System

All features are available to all users. Tiers affect fee rates only:

| Tier | Cost | Profit Fee | GPU Sweep Fee |
|------|------|-----------|---------------|
| Free | None | 25% of profits | 15% of NOS |
| Pro/VIP | 10 SOL (one-time) | 0% | 5% of NOS |

---

## Health & Status

### GET /api/health

Server health check.

**Response:**
```json
{
  "status": "ok",
  "uptime": 86400
}
```

---

### GET /api/status

Platform status including service availability.

**Response:**
```json
{
  "status": "operational",
  "services": {
    "trading": true,
    "scoring": true,
    "gpu": true,
    "chat": true
  }
}
```

---

### GET /api/config

Public configuration values.

---

### GET /api/sol-price

Current SOL/USD price (cached).

**Response:**
```json
{
  "price": 185.42,
  "timestamp": "2026-03-24T12:00:00Z"
}
```

---

### GET /api/pricing

Fee structure and pricing details.

---

## GPU Endpoints

Manage GPU nodes connected to the Nosana decentralized compute network. See [GPU_NOSANA_GUIDE.md](./GPU_NOSANA_GUIDE.md) for full setup instructions.

### POST /api/gpu/register

Register a new GPU node. Generates a Solana keypair and assigns a Nosana market.

**Request:**
```json
{
  "wallet": "8xYz...your_wallet",
  "gpuModel": "NVIDIA GeForce RTX 4090"
}
```

**Response (200):**
```json
{
  "nodeId": "gpu_abc123",
  "keypair": [/* 64-byte Solana keypair */],
  "market": {
    "address": "97G9NnvBDQ2WpKu6fasoMsAKmfj63C9rhysJnkeWodAf",
    "tier": "RTX 4090",
    "nosPerSec": 0.000097,
    "vram": 24
  }
}
```

**Error (409):** User already has an active node.

---

### POST /api/gpu/reconnect

Reactivate a disconnected node. Preserves original keypair.

**Request:**
```json
{
  "wallet": "8xYz...your_wallet",
  "nodeId": "gpu_abc123"
}
```

---

### POST /api/gpu/deregister

Remove node and trigger final earnings sweep.

**Request:**
```json
{
  "wallet": "8xYz...your_wallet",
  "nodeId": "gpu_abc123"
}
```

**Response (200):**
```json
{
  "status": "deregistered",
  "finalSweep": {
    "amount": 2.45,
    "txSignature": "5xYz..."
  }
}
```

---

### GET /api/gpu/dashboard

Node status, metrics, and earnings history.

**Headers:** `X-Wallet`

**Response (200):**
```json
{
  "node": {
    "nodeId": "gpu_abc123",
    "status": "busy",
    "gpuModel": "NVIDIA GeForce RTX 4090",
    "uptime": 86400,
    "lastHeartbeat": "2026-03-24T12:00:00Z",
    "metrics": {
      "gpu_utilization": 85,
      "gpu_temp": 72,
      "gpu_memory_used": 8192,
      "gpu_memory_total": 24576,
      "power_draw": 320
    }
  },
  "earnings": {
    "totalNOS": 156.78,
    "todayNOS": 8.4,
    "lastSweep": "2026-03-24T11:00:00Z",
    "pendingNOS": 1.2
  }
}
```

---

### POST /api/gpu/heartbeat

Agent sends GPU metrics every 60s. Returns queued commands.

**Request:**
```json
{
  "wallet": "8xYz...",
  "nodeId": "gpu_abc123",
  "metrics": {
    "gpu_utilization": 85,
    "gpu_temp": 72,
    "gpu_memory_used": 8192,
    "gpu_memory_total": 12288,
    "power_draw": 180,
    "fan_speed": 65,
    "clock_speed": 1800
  },
  "status": "busy"
}
```

**Response (200):**
```json
{
  "ack": true,
  "commands": []
}
```

---

### POST /api/gpu/job-completed

Agent reports a completed inference job.

**Request:**
```json
{
  "wallet": "8xYz...",
  "nodeId": "gpu_abc123",
  "jobId": "nosana_job_456",
  "duration": 45.2,
  "success": true
}
```

---

### GET /api/gpu/fleet

Aggregate fleet view across all user nodes.

**Headers:** `X-Wallet`

---

### POST /api/gpu/command

Queue a remote command for the node.

**Request:**
```json
{
  "wallet": "8xYz...",
  "nodeId": "gpu_abc123",
  "command": "restart"
}
```

Available commands: `restart`, `stop`, `logs`, `power`

---

### GET /api/gpu/command-status

Check status of a queued command.

**Query:** `?commandId=cmd_xyz`

---

### GET /api/gpu/node-logs

Retrieve recent Docker logs from the node.

**Headers:** `X-Wallet`

---

## Nosana / AI Inference

Deploy models and submit inference jobs to the Nosana GPU network.

### POST /api/nosana/deploy

Deploy a persistent inference service on Nosana.

**Headers:** `X-Wallet`

**Request:**
```json
{
  "model": "bondli-scorer",
  "market": "RTX 4090",
  "config": {
    "replicas": 1
  }
}
```

---

### POST /api/nosana/job

Submit a one-off inference job.

**Headers:** `X-Wallet`

**Request:**
```json
{
  "type": "inference",
  "model": "bondli-scorer",
  "input": { /* model-specific input */ }
}
```

---

### GET /api/nosana/job/:jobId

Get job status and results.

**Response (200):**
```json
{
  "jobId": "job_abc",
  "status": "completed",
  "result": { /* inference output */ },
  "duration": 12.5
}
```

---

### GET /api/nosana/status

Router status and statistics.

---

### GET /api/nosana/markets

Available GPU markets and pricing.

**Response (200):**
```json
{
  "markets": [
    {
      "tier": "RTX 3060",
      "address": "7AtiXMSH6R1jjBxrcYjehCkkSF7zvYWte63gwEDBcGHq",
      "nosPerSec": 0.000043,
      "vram": 12
    },
    {
      "tier": "RTX 4090",
      "address": "97G9NnvBDQ2WpKu6fasoMsAKmfj63C9rhysJnkeWodAf",
      "nosPerSec": 0.000097,
      "vram": 24
    },
    {
      "tier": "H100",
      "address": "Crop49jpc7prcgAcS82WbWyGHwbN5GgDym3uFbxxCTZg",
      "nosPerSec": 0.000150,
      "vram": 80
    }
  ]
}
```

---

### GET /api/nosana/credits

User's inference credits balance.

**Headers:** `X-Wallet`

---

## Token Radar & Scoring

Real-time token discovery and ML-powered scoring.

### GET /api/radar

Real-time feed of new token launches with entry signals.

**Headers:** `X-Wallet`

**Response (200):**
```json
{
  "tokens": [
    {
      "ca": "pump123...abc",
      "name": "BONK2",
      "symbol": "BONK2",
      "score": 82,
      "signal": "STRONG",
      "rugProbability": 0.05,
      "launchedAt": "2026-03-24T12:00:00Z",
      "marketCap": 45000,
      "holders": 234
    }
  ]
}
```

---

### GET /api/radar/token/:ca

Detailed view for a specific token.

**Parameters:** `ca` — Contract address (mint)

---

### GET /api/score/:ca

ML scoring with 40+ features.

**Parameters:** `ca` — Contract address

**Response (200):**
```json
{
  "ca": "pump123...abc",
  "score": 82,
  "velocity": 3.2,
  "acceleration": -0.1,
  "rugProbability": 0.05,
  "graduationProbability": 0.72,
  "signal": "STRONG",
  "features": {
    "holderDistribution": 0.78,
    "buyPressure": 0.85,
    "walletAge": 0.65,
    "devProfile": 0.90,
    "liquidityDepth": 0.70,
    "socialSentiment": 0.80,
    "artworkOriginality": 0.95
  },
  "rugSignals": {
    "count": 0,
    "signals": []
  }
}
```

**Score Interpretation:**

| Range | Signal | Action |
|-------|--------|--------|
| 80-100 | APE | Auto-enter (configurable) |
| 40-79 | WATCH | Monitor for entry |
| 0-39 | SKIP | Pass |

**Hard Rejection:** 3+ moderate rug signals = auto-skip regardless of score.

---

### GET /api/memetic-score/:ca

Full memetic DNA analysis across 7 modules (artwork, linguistics, cultural timing, archetype matching, absurdity, virality, originality).

**Parameters:** `ca` — Contract address

---

### GET /api/movers

Top performing tokens by price movement.

**Headers:** `X-Wallet`

---

### GET /api/movers/token/:ca

Detailed analytics for a top mover.

---

### GET /api/graduated

Tokens that graduated from pump.fun bonding curve to Raydium AMM.

---

### GET /api/spot/:mint

Current price for a token.

**Parameters:** `mint` — Token mint address

**Response (200):**
```json
{
  "mint": "pump123...abc",
  "price": 0.000045,
  "priceUsd": 0.0083
}
```

---

### POST /api/spot/batch

Batch price lookup for multiple tokens.

**Request:**
```json
{
  "mints": ["pump123...abc", "pump456...def"]
}
```

---

### GET /api/candles/:mint

OHLCV candlestick data.

**Parameters:** `mint` — Token mint address

**Query:** `?interval=1m&limit=100`

---

### GET /api/pool/:mint

Liquidity pool data (Raydium, Meteora).

---

### GET /api/chart/:ca

Chart pattern analysis (staircase detection, momentum classification).

---

## Trading

Execute trades on Solana (pump.fun, Raydium, Bags.fm).

### POST /api/quick-ape

Quick buy a token.

**Headers:** `X-Wallet`

**Request:**
```json
{
  "ca": "pump123...abc",
  "amountSol": 1.0,
  "slippageBps": 500
}
```

**Response (200):**
```json
{
  "txSignature": "3abc...xyz",
  "amountSol": 1.0,
  "tokensReceived": 1250000,
  "price": 0.0000008
}
```

---

### POST /api/quick-sell

Quick sell a token position.

**Headers:** `X-Wallet`

**Request:**
```json
{
  "ca": "pump123...abc",
  "percentage": 100,
  "slippageBps": 500
}
```

---

### GET /api/positions/:wallet

All open positions for a wallet.

**Response (200):**
```json
{
  "positions": [
    {
      "ca": "pump123...abc",
      "symbol": "BONK2",
      "entryPrice": 0.0000008,
      "currentPrice": 0.0000024,
      "pnlPercent": 200,
      "amountTokens": 1250000,
      "valueSol": 3.0
    }
  ]
}
```

---

## Auto-Trade Engine

Autonomous trading with ML-driven entry/exit.

### POST /api/auto-trade/start

Start an autonomous trading session.

**Headers:** `X-Wallet`

**Request:**
```json
{
  "config": {
    "maxPositionSol": 2.0,
    "scoreThreshold": 80,
    "takeProfitMultiple": 3.0,
    "stopLossPercent": -50,
    "trailingStopPercent": 80
  }
}
```

**Response (200):**
```json
{
  "sessionId": "auto_abc123",
  "status": "running",
  "config": { /* echoed config */ }
}
```

### 3-Layer Exit System

1. **Score Derivatives** — Exit when momentum (velocity) deteriorates or acceleration turns negative. Detects trends ~2 cycles before price drops.
2. **Absolute Thresholds** — Take profit at configured multiple (default 3x), stop loss at configured percentage (default -50%).
3. **Trailing Stop** — Lock in gains at configured percentage of peak price (default 80%).

---

### POST /api/auto-trade/stop

Stop the auto-trading session.

**Headers:** `X-Wallet`

---

### POST /api/auto-trade/exit-position

Manually exit a specific position.

**Headers:** `X-Wallet`

**Request:**
```json
{
  "ca": "pump123...abc"
}
```

---

### GET /api/auto-trade/status

Current session status and active positions.

**Headers:** `X-Wallet`

---

## Session Management

Manage isolated trading sessions with dedicated wallets.

### POST /api/session/create

Create a new trading session.

**Headers:** `X-Wallet`

---

### POST /api/session/fund

Fund a session wallet with SOL.

**Headers:** `X-Wallet`

**Request:**
```json
{
  "sessionId": "sess_abc",
  "amountSol": 5.0
}
```

---

### POST /api/session/launch

Launch/deploy a token via a session.

---

### GET /api/session/balance

Get session wallet balance.

**Headers:** `X-Wallet`

---

### POST /api/session/command

Send a control command to a session.

---

### POST /api/session/force-recover

Recover a stuck session.

**Headers:** `X-Wallet`

---

### GET /api/sessions

List all sessions for the authenticated wallet.

**Headers:** `X-Wallet`

---

### POST /api/sessions/clear

Clear all sessions.

**Headers:** `X-Wallet`

---

## Fleet Trading

Coordinated multi-wallet trading operations.

### GET /api/fleet

Fleet status overview.

**Headers:** `X-Wallet`

---

### GET /api/fleet/wallets/:ca

Wallets in the fleet holding a specific token.

---

### POST /api/fleet/wallet-sell

Coordinated sell across multiple fleet wallets.

**Headers:** `X-Wallet`

**Request:**
```json
{
  "ca": "pump123...abc",
  "percentage": 100
}
```

---

### GET /api/fleet/balances/:ca

Token balances across the fleet for a given token.

---

## Intelligence

ML-powered analytics and market intelligence.

### GET /api/intel/stats

Scoring model statistics and accuracy metrics.

---

### GET /api/intel/x-stats

Twitter/X signal aggregation statistics.

---

### GET /api/intel/x-analyze/:ca

Social intelligence analysis for a token (Twitter sentiment, KOL mentions, engagement metrics).

**Parameters:** `ca` — Contract address

---

### GET /api/intel/artwork/:ca

Artwork forensic analysis — perceptual hashing to detect duplicated/recycled token artwork.

**Parameters:** `ca` — Contract address

**Response (200):**
```json
{
  "ca": "pump123...abc",
  "isOriginal": true,
  "duplicates": [],
  "hashDistance": null
}
```

---

### GET /api/intel/artwork-stats

Global artwork duplication statistics.

---

### GET /api/intel/survivor-stats

Survivorship bias analysis metrics.

---

### GET /api/intel/survivor-check/:ca

Check if a token matches known successful archetypes.

---

### GET /api/intel/smart-money

Track wallets with consistently profitable trades.

**Headers:** `X-Wallet`

---

### GET /api/intel/smart-money/signals

Real-time buy signals from tracked smart money wallets.

**Headers:** `X-Wallet`

---

### GET /api/intel/dev-wallets

Dev wallet profiling — identify serial ruggers and trusted developers.

---

### GET /api/intel/dev-wallets/:addr

Profile a specific developer wallet.

**Parameters:** `addr` — Solana wallet address

**Response (200):**
```json
{
  "address": "Dev123...abc",
  "tokensLaunched": 12,
  "rugCount": 0,
  "avgTokenLifespan": "4.5 days",
  "trustScore": 0.92,
  "history": [/* token launch history */]
}
```

---

## Access & Tiers

### GET /api/access/check

Check wallet's current tier and access status.

**Headers:** `X-Wallet`

**Response (200):**
```json
{
  "wallet": "8xYz...",
  "tier": "free",
  "features": "all",
  "profitFee": 0.25,
  "gpuFee": 0.15
}
```

---

### POST /api/access/activate-free

Activate free tier (no payment required).

**Request:**
```json
{
  "wallet": "8xYz..."
}
```

---

### POST /api/access/activate-pro

Activate Pro tier with 10 SOL payment.

**Request:**
```json
{
  "wallet": "8xYz...",
  "txSignature": "5abc...payment_tx"
}
```

Payment is verified on-chain before activation.

---

## Chat (AI Support)

Built-in AI assistant powered by xAI Grok, trained on Nosana architecture, GPU setup, and Solana trading.

### POST /api/chat

Send a message to the AI assistant.

**Headers:** `X-Wallet`

**Request:**
```json
{
  "messages": [
    { "role": "user", "content": "How do I fix my node showing offline?" }
  ],
  "wallet": "8xYz..."
}
```

**Response (200):**
```json
{
  "reply": "Check if the Docker container is running with: docker ps | grep nosana. If it's not listed, try: docker restart nosana-node",
  "usage": {
    "prompt_tokens": 150,
    "completion_tokens": 45
  }
}
```

**Rate Limit:** 10 messages per user per minute.

---

### GET /api/chat/status

Chat service availability.

---

## Utilities

### POST /api/username

Set a display username for your wallet.

**Request:**
```json
{
  "wallet": "8xYz...",
  "username": "degen_trader"
}
```

---

### GET /api/username/:wallet

Get username for a wallet address.

---

### GET /api/referral/:wallet

Referral statistics and earnings.

---

### POST /api/referral/apply

Apply a referral code.

**Request:**
```json
{
  "wallet": "8xYz...",
  "code": "REF_ABC123"
}
```

---

### GET /api/bags/status

Bags.fm (Meteora DBC) integration status.

---

### GET /api/bags/feed

Bags.fm token feed.

---

## Error Handling

All errors follow a consistent format:

```json
{
  "error": "Description of what went wrong",
  "code": "ERROR_CODE"
}
```

### HTTP Status Codes

| Code | Meaning |
|------|---------|
| 200 | Success |
| 400 | Bad request — missing or invalid parameters |
| 401 | Unauthorized — missing X-Wallet header |
| 403 | Forbidden — insufficient tier or permissions |
| 404 | Not found — resource doesn't exist |
| 409 | Conflict — resource already exists (e.g., duplicate GPU registration) |
| 429 | Rate limited — too many requests |
| 500 | Server error |

### Common Error Codes

| Code | Description |
|------|-------------|
| `WALLET_REQUIRED` | X-Wallet header missing |
| `NODE_EXISTS` | User already has an active GPU node |
| `NODE_NOT_FOUND` | GPU node not found for this wallet |
| `INSUFFICIENT_BALANCE` | Not enough SOL for the operation |
| `TOKEN_NOT_FOUND` | Contract address not found |
| `SESSION_NOT_FOUND` | Trading session doesn't exist |
| `RATE_LIMITED` | Too many requests, try again later |

---

## Rate Limits

| Endpoint Category | Limit |
|-------------------|-------|
| General API | 60 requests/min per wallet |
| GPU Heartbeat | 120 requests/min per node |
| Trading (buy/sell) | 10 requests/min per wallet |
| Chat | 10 messages/min per wallet |
| Scoring | 30 requests/min per wallet |

Rate limit headers are included in responses:

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1711281600
```
