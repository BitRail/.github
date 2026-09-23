<div align="center">

<br />

# Bitrail

### Risk rails for productive Bitcoin on Stacks.

[![Built on Stacks](https://img.shields.io/badge/Built%20on-Stacks%20L2-orange?style=for-the-badge&logo=bitcoin)](https://www.stacks.co/)
[![sBTC](https://img.shields.io/badge/Asset-sBTC%20%2F%20stBTC-f7931a?style=for-the-badge)](https://stacks.co/sbtc)
[![Protocols](https://img.shields.io/badge/Protocols-Zest%20%2B%20StackingDAO-blueviolet?style=for-the-badge)](#integrations)
[![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](LICENSE)

<br />

**See the risk. Move capital only inside your limits.**

Bitrail is a Stacks-native risk intelligence and guarded capital-routing layer for productive Bitcoin — sBTC, stBTC, Zest lending positions, and Bitflow LP. It gives users and protocols a shared cross-protocol view of health, liquidation distance, and safe rebalancing so Bitcoin capital can move deeper into the Stacks ecosystem without opaque risk.

<br />

[Dashboard](#) · [SDK Docs](#sdk) · [Risk Model](RISK_MODEL.md) · [Contract Addresses](CONTRACT_ADDRESSES.md)

<br />

</div>

---

## The Problem

Bitcoin on Stacks can now earn and be deployed across multiple protocols simultaneously — staked as stBTC on StackingDAO, supplied as collateral on Zest, providing liquidity on Bitflow. Risk is no longer single-protocol. It is fragmented, cross-protocol, and invisible to the user.

A user with stBTC collateral borrowing sBTC on Zest, while their stBTC yield fluctuates with Bitcoin Staking rewards, has no single view of their true liquidation distance. They are flying blind.

```
User's actual exposure:
  stBTC collateral on Zest   →   liquidation threshold: 80% LTV
  sBTC debt on Zest          →   accruing interest
  Bitflow LP position        →   impermanent loss not factored in
  sBTC/stBTC ratio shifting  →   collateral value changing silently

Existing tools show each of these in isolation.
Bitrail shows the net risk picture — and acts on it safely.
```

---

## What Bitrail Is

Bitrail is **infrastructure** — not a DEX, not a lending protocol, not a yield farm, not another portfolio tracker.

It is three things:

**1. Risk Intelligence**
Aggregates positions across Zest, StackingDAO, and Bitflow. Computes a transparent health factor using a versioned, documented model (`bitrail-risk-v0.1`). Shows liquidation distance in %, USD, and BTC. Labels every assumption.

**2. Alert Engine**
Users set thresholds — `health < 1.3`, `liquidation within 10%`. Bitrail monitors every 5 minutes and fires webhooks or in-app alerts before danger.

**3. Guarded Routing**
Optional execution layer. The Bitrail router contract executes a capital action — repay debt, reduce borrow — only if the simulated post-action health factor meets the user's registered policy minimum. If the health check fails, the transaction does not execute. Fails closed, always.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Browser                            │
│              Leather / Xverse wallet connect                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Bitrail Web App                             │
│                    (Next.js · lava/)                            │
│                                                                 │
│  Portfolio ─ Health Score ─ Positions ─ Alerts ─ Actions       │
└────────────────────────────┬────────────────────────────────────┘
                             │  proxied API calls
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Bitrail Indexer + API                          │
│                   (Node.js · sbtc-pay/)                         │
│                                                                 │
│  /v1/bitrail/positions/:address                                 │
│  /v1/bitrail/health/:address                                    │
│  /v1/bitrail/alerts                                             │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Zest Adapter │  │ StackingDAO  │  │   Risk Engine v0.1   │  │
│  │              │  │   Adapter    │  │  healthFactor formula │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────────────────┘  │
└─────────┼────────────────┼──────────────────────────────────────┘
          │                │
          ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Stacks Mainnet                               │
│                                                                 │
│  Zest pool-borrow-v2-3      sBTC / stBTC token contracts        │
│  StackingDAO data-stbtc-v1  Bitflow univ2-core                  │
│  Hiro API                   Bitrail policy + router contracts   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Health Score Model

Model ID: `bitrail-risk-v0.1` — versioned, transparent, auditable.

```
healthFactor = totalCollateralValueUSD / (totalDebtValueUSD / liquidationThreshold)

status:
  healthFactor > 1.5   →  SAFE    (green)
  healthFactor 1.0–1.5 →  WATCH   (yellow)
  healthFactor < 1.0   →  DANGER  (red, liquidatable)

liquidationDistancePct = ((healthFactor - 1.0) / healthFactor) × 100
liquidationDistanceBTC = (totalCollateralUSD - totalDebtUSD / threshold) / btcPriceUSD
```

Every score includes a visible `assumptions[]` array — users see exactly what was assumed, including price source, model limitations, and data freshness. See [RISK_MODEL.md](RISK_MODEL.md).

---

## SDK

Any protocol can consume Bitrail risk data in one line:

```typescript
import { getBitrailHealth } from '@bitrail/sdk';

const health = await getBitrailHealth('SP2VCQJGH7PHP2DJK7Z0V48AGBHQAW3R3ZW1QF4N');

console.log(health.healthFactor);        // 2.41
console.log(health.status);              // 'safe'
console.log(health.liquidationDistancePct); // 58.5
console.log(health.modelId);             // 'bitrail-risk-v0.1'
console.log(health.assumptions);         // ['spot price used', 'no slippage modeled', ...]
```

Full SDK reference:

```typescript
const client = new BitrailClient('https://api.bitrail.xyz');

// Get health score
const health = await client.getHealth(address);

// List normalized positions across all protocols
const positions = await client.listPositions(address);

// Register an alert webhook
await client.subscribeAlert({
  address,
  alertType: 'health_below',
  threshold: 1.3,
  webhookUrl: 'https://yourapp.com/bitrail-alert',
});
```

---

## Integrations

| Protocol | Data Read | Status |
|---|---|---|
| **Zest Protocol** | Supply, borrow, LTV, liquidation threshold | ✅ MVP |
| **StackingDAO** | stBTC balance, sBTC/stBTC ratio | ✅ MVP |
| **Bitflow** | LP positions (sBTC/USDCx, STX/USDCx) | 🔄 Post-MVP |
| **Native sBTC** | Wallet balance | ✅ MVP |

---

## Smart Contracts

Non-custodial by default. Minimal surface area.

### `bitrail-policy.clar`
User self-registers risk limits on-chain. Stores `max-ltv-bps` and `min-health-bps` per principal. No token approvals. No custody.

### `bitrail-router.clar`
Executes a capital action (e.g. repay Zest debt) only if the caller-provided post-action health factor meets the user's registered policy minimum. **Fails closed**: if the health check fails, no action executes — the transaction aborts.

```clarity
;; Router will NOT execute if expected post-health < your registered minimum
(define-public (guarded-repay (market <market-trait>) (ft <ft-trait>) (amount uint) (expected-health-bps uint))
  ...
  (asserts! (>= expected-health-bps min-health) (err err-health-check-failed))
  (try! (contract-call? market repay ft amount none))
  (ok true))
```

---

## Repo Structure

```
STACKS GRANT/
├── lava/                   # Bitrail web app (Next.js 15)
│   ├── app/dashboard/      # Portfolio, health, positions, alerts, actions
│   ├── components/bitrail/ # HealthScoreWidget, PositionsTable, AlertCard
│   └── stores/             # Wallet state (Leather/Xverse)
│
├── sbtc-pay/               # Bitrail indexer + API (Node.js)
│   ├── src/lib/bitrail/    # Adapters, risk engine, alert engine
│   ├── src/app/api/v1/bitrail/ # REST endpoints
│   ├── contracts/          # bitrail-policy.clar, bitrail-router.clar
│   └── packages/bitrail-sdk/   # @bitrail/sdk TypeScript package
│
└── StacksMCPServer/        # Protocol data layer (Zest, sBTC, Bitflow plugins)
```

---

## Security

- Never requests seed phrases or private keys
- No unlimited token approvals in MVP
- Router fails closed — health check failure aborts action
- Smart contracts recommended for audit before large allowances
- All APIs rate-limited
- Clear disclaimers: not financial advice; scores can be wrong; verify on-chain

---


## Disclaimer

> Bitrail health scores are computed from on-chain data using a documented model (`bitrail-risk-v0.1`). Scores can be wrong. Prices can be stale. Model assumptions are visible in every response. This is not financial advice. Always verify your position on-chain before taking action. Smart contracts have not been audited — use limited allowances and exercise caution.

---

<div align="center">

Built on Stacks · Secured by Bitcoin · Not financial advice

</div>
