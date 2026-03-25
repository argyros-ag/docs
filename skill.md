---
name: argyros-swap
description: Integrate Argyros swap API on Fogo. Use for getting quotes, building swap transactions, getting raw instructions, and handling errors across Vortex, Fluxbeam, and Fogo.fun bonding curves. Solana support coming soon.
version: "1.1.0"
tags:
  - argyros
  - fogo
  - swap
  - aggregator
  - defi
  - quote
  - routing
  - multi-hop
---

# Argyros Swap API

**Base URL**: `https://api.argyros.xyz`
**OpenAPI spec**: `https://argyros.xyz/api-reference/openapi.json`
**Full reference**: `https://argyros.xyz/llms-full.txt`

## Chain Support

Currently: **Fogo** (5 max hops, Vortex/Fluxbeam/Fogo.fun DEXs).

**Coming soon**: Solana with P-Token (7 max hops, lower compute, Raydium/Orca/Meteora). When available, a `chain` query parameter will be added to all endpoints.

## Use / Do Not Use

Use when:
- The task involves swapping tokens on Fogo via Argyros.
- The user needs a quote, swap transaction, or raw instructions.
- The user needs help debugging Argyros API errors.

Do not use when:
- The task is generic Solana development with no Argyros API usage.
- The task involves other aggregators (Jupiter, etc.).

**Triggers**: `swap`, `quote`, `best route`, `token swap`, `argyros`, `multi-hop`, `price impact`, `slippage`, `build transaction`, `raw instructions`, `fogo`

## Quick Reference

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/quote` | GET | Get best route and estimated output |
| `/api/v1/swap` | POST | Build unsigned swap transaction |
| `/api/v1/instructions` | POST | Get raw instructions for custom tx |
| `/health` | GET | Service health check |

## Intent → Endpoint Decision Tree

### "I want to swap Token A for Token B"

```
1. GET /api/v1/quote
   params: inputMint, outputMint, amount, swapMode=ExactIn, slippageBps=50
   
2. Check response:
   - success=false → handle error (see Error Recovery)
   - priceImpactSeverity=high|extreme → warn user
   
3. POST /api/v1/swap
   body: { userWallet, inputMint, outputMint, amount, swapMode, slippageBps }
   
4. Check simulation:
   - simulation.insufficientFunds=true → show balance error
   - simulation.slippageExceeded=true → retry with fresh quote
   
5. Sign transaction (base64 decode → VersionedTransaction.deserialize → sign)
6. Submit (sendRawTransaction → confirmTransaction)
```

### "I want to get a price quote only"

```
1. GET /api/v1/quote
   params: inputMint, outputMint, amount, swapMode=ExactIn
   
2. Use response fields:
   - amountOut: estimated output
   - priceImpactPercent: price impact
   - feeBps: total fees
   - hopCount: number of hops (up to 5)
   - routes[]: detailed route info per hop
```

### "I want to compose swap with other instructions"

```
1. POST /api/v1/instructions
   body: { userWallet, inputMint, outputMint, amount, swapMode, slippageBps }
   
2. Build v0 transaction:
   - Fetch addressLookupTableAddresses via getAddressLookupTable()
   - Convert instructions[] to TransactionInstruction[]
   - Add your custom instructions before/after
   - Compile to V0Message with ALTs
   - Sign and submit
```

### "I want to buy exactly X amount of a token"

```
Same as swap flow, but use swapMode=ExactOut:
- amount = desired output amount in smallest units
- Response amountIn = calculated input needed
- Response maxAmountIn = maximum you'll spend (slippage-adjusted)
```

## Parameter Reference

### GET /api/v1/quote

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| inputMint | string | yes | — | Input token mint (base58) |
| outputMint | string | yes | — | Output token mint (base58) |
| amount | string | yes | — | Amount in smallest units |
| swapMode | string | yes | — | `ExactIn` or `ExactOut` |
| slippageBps | integer | no | 50 | Slippage tolerance (1 bps = 0.01%) |

### POST /api/v1/swap

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| userWallet | string | yes | — | Signer wallet address (base58) |
| inputMint | string | yes | — | Input token mint |
| outputMint | string | yes | — | Output token mint |
| amount | string | yes | — | Amount in smallest units |
| swapMode | string | yes | — | `ExactIn` or `ExactOut` |
| slippageBps | integer | no | 50 | Slippage tolerance |
| skipSimulation | boolean | no | false | Skip pre-flight simulation |

### POST /api/v1/instructions

Same as `/swap` without `skipSimulation`.

## Response Schemas

### QuoteResponse (GET /quote → data)

```json
{
  "inputMint": "string",
  "outputMint": "string",
  "amountIn": "string",
  "amountOut": "string",
  "priceImpactBps": 0,
  "priceImpactPercent": "string",
  "priceImpactSeverity": "none|low|moderate|high|extreme",
  "priceImpactWarning": "string",
  "feeBps": 0,
  "routes": [{ "poolAddress": "string", "poolType": "string", "percent": 0, "inputMint": "string", "outputMint": "string" }],
  "routePath": ["string"],
  "hopCount": 0,
  "otherAmountThreshold": "string"
}
```

### SwapResponse (POST /swap → data)

```json
{
  "transaction": "base64-string",
  "lastValidBlockHeight": 0,
  "amountIn": "string",
  "amountOut": "string",
  "minAmountOut": "string (ExactIn only)",
  "maxAmountIn": "string (ExactOut only)",
  "feeAmount": "string",
  "simulation": { "success": true, "logs": [], "computeUnitsConsumed": 0, "error": "", "insufficientFunds": false, "slippageExceeded": false },
  "computeUnitsEstimate": 0,
  "route": ["string"],
  "hopCount": 0,
  "pools": ["string"],
  "isSplitRoute": false,
  "splitPercents": [0]
}
```

### InstructionsResponse (POST /instructions → data)

```json
{
  "amountIn": "string",
  "amountOut": "string",
  "otherAmountThreshold": "string",
  "feeAmount": "string",
  "route": ["string"],
  "hopCount": 0,
  "pools": ["string"],
  "instructions": [{ "programId": "base58", "data": "base64", "accounts": [{ "publicKey": "base58", "isSigner": false, "isWritable": false }] }],
  "addressLookupTableAddresses": ["string"]
}
```

## Error Recovery

| Error | HTTP | Recovery |
|-------|------|----------|
| `no route found` | 404 | Try smaller amount, different slippage, or check token support |
| `no pool found` | 404 | Token may not have listed pools |
| `insufficient liquidity` | 400 | Reduce swap amount |
| `invalid inputMint` | 400 | Verify base58 public key format |
| `invalid user wallet` | 400 | Verify wallet address |
| `simulation failed` | 500 | Check simulation.logs, verify balance |
| Rate limited | 429 | Exponential backoff: 1s, 2s, 4s. Max 3 retries |
| `SlippageExceeded` (on-chain) | — | Get fresh quote, rebuild transaction |
| `BlockhashExpired` (on-chain) | — | Rebuild transaction from scratch |

## Common Token Mints

| Token | Mint | Decimals |
|-------|------|----------|
| SOL | `So11111111111111111111111111111111111111112` | 9 |
| USDC | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` | 6 |

## Rate Limits

Default: 60 requests/minute, 10 requests/second burst.
