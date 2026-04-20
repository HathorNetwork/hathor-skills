---
name: hathor-wallet-headless
description: Interact with the Hathor wallet-headless HTTP API — create and manage wallets, query balances, send transactions, create tokens and NFTs, run nano contracts, atomic swaps, multisig, and more. Use when the user mentions the headless wallet, wallet-headless, hathor wallet API, or wants to programmatically manage Hathor wallets via curl/HTTP.
---

# Hathor Wallet Headless

The **wallet-headless** is the official headless wallet of Hathor Network — a daemon that exposes wallet operations via HTTP. It runs a Hathor wallet in memory and lets any HTTP client create wallets, send transactions, interact with tokens and nano contracts, etc.

Default local port: `8000`. Base URL in this doc: `$HEADLESS` (export to your URL, e.g. `http://127.0.0.1:8000`).

## Core Concepts

### The `x-wallet-id` header

Almost every `/wallet/*` endpoint requires an `x-wallet-id` header. That header routes the request to a specific wallet that was previously started with `POST /start`. Without it, the middleware rejects the request.

```bash
curl -H "x-wallet-id: my-wallet" "$HEADLESS/wallet/balance"
```

The root endpoints (`/start`, `/multisig-pubkey`, `/push-tx`, `/configuration-string`, `/reload-config`, `/health`) do **not** require it.

### Optional API key

If the server was configured with `http_api_key`, add `X-API-KEY: <key>` to every request. This is a global gate, applied to all endpoints including the root ones.

### Amounts are integers in cents

Every `value` / `amount` field is an **integer in the smallest unit** (cents). `1 HTR = 100`. The API rejects floats. Convert user-facing amounts before calling.

### Wallet readiness: `statusCode: 3`

After `POST /start`, the wallet takes time to sync. Poll `GET /wallet/status`:

- `0` = Closed
- `1` = Connecting
- `2` = Syncing
- `3` = Ready ← only now can you send transactions

Do not attempt send/balance queries before `3`.

### Tokens

- HTR (native token) uid is `"00"`.
- Custom tokens are referenced by their creation-tx hash (a hex string).

## Quick Reference: Most Common Flow

```bash
export HEADLESS=http://127.0.0.1:8000
WALLET=my-wallet

# 1. Start a wallet (uses seedKey from config.js)
curl -X POST "$HEADLESS/start" \
  -H "Content-Type: application/json" \
  -d "{\"wallet-id\":\"$WALLET\",\"seedKey\":\"default\"}"

# 2. Wait until ready (statusCode: 3)
while true; do
  code=$(curl -s -H "x-wallet-id: $WALLET" "$HEADLESS/wallet/status" | jq -r '.statusCode')
  [ "$code" = "3" ] && break
  sleep 2
done

# 3. Get an address
curl -H "x-wallet-id: $WALLET" "$HEADLESS/wallet/address"
# → {"address":"H8bt9nYhUNJHg7szF32CWWi1eB8PyYZnbt"}

# 4. Check balance (HTR)
curl -H "x-wallet-id: $WALLET" "$HEADLESS/wallet/balance"
# → {"available":200,"locked":0}   (= 2.00 HTR)

# 5. Send 1.23 HTR to an address
curl -X POST "$HEADLESS/wallet/simple-send-tx" \
  -H "x-wallet-id: $WALLET" \
  -H "Content-Type: application/json" \
  -d '{"address":"HNnK9...","value":123}'

# 6. Stop when done
curl -X POST "$HEADLESS/wallet/stop" -H "x-wallet-id: $WALLET"
```

## Endpoint Map

| Area | Highlights | Reference |
|------|------------|-----------|
| **Lifecycle & core wallet** | `/start`, `/wallet/status`, `/wallet/balance`, `/wallet/address`, `/wallet/addresses`, `/wallet/simple-send-tx`, `/wallet/send-tx`, `/wallet/stop`, history, decode | [endpoints-core.md](references/endpoints-core.md) |
| **Tokens, NFTs, UTXOs** | `/wallet/create-token`, `/wallet/mint-tokens`, `/wallet/melt-tokens`, `/wallet/create-nft`, `/wallet/utxo-filter`, `/wallet/utxo-consolidation` | [endpoints-tokens.md](references/endpoints-tokens.md) |
| **Nano contracts** | `/wallet/nano-contracts/create`, `/execute`, `/state`, `/history`, `/create-on-chain-blueprint`, oracle helpers | [endpoints-nano.md](references/endpoints-nano.md) |
| **Offline signing & P2SH** | `/wallet/tx-proposal/*`, `/wallet/p2sh/tx-proposal/*`, `/push-tx` | [endpoints-tx-proposal.md](references/endpoints-tx-proposal.md) |
| **Atomic swap** | `/wallet/atomic-swap/tx-proposal/*` | [endpoints-atomic-swap.md](references/endpoints-atomic-swap.md) |
| **Tx templates, HSM, Fireblocks, health, config** | `/wallet/tx-template/*`, `/hsm/start`, `/fireblocks/start`, `/health`, `/wallet/config/*` | [endpoints-misc.md](references/endpoints-misc.md) |
| **End-to-end recipes** | Create wallet, fund, send, create token, mint, NFT, nano contract, swap | [examples.md](references/examples.md) |

## Error Shape

All errors follow:

```json
{ "success": false, "message": "Error description" }
```

Some endpoints add `state` (wallet state code) or `errorCode` (e.g. `WALLET_ALREADY_STARTED`). Always check `success` before trusting the rest of the payload.

## Starting a Wallet: Your Options

`POST /start` has several modes. Pick one:

| Mode | Required fields | Use when |
|------|-----------------|----------|
| **Seed from config** | `wallet-id`, `seedKey` | Normal case — seed is in `config.js` keyed by name (e.g. `default`) |
| **Seed inline** | `wallet-id`, `seed` (24 words) | Testing; avoid in production (leaves seed in logs) |
| **Read-only (xpub)** | `wallet-id`, `xpubkey` | Watch-only wallet — can query, cannot sign |
| **MultiSig** | `wallet-id`, `multisig: true`, `multisigKey` | Multisig wallet, participants defined in config |
| **HSM** | call `POST /hsm/start` instead | Keys live on an HSM device |
| **Fireblocks** | call `POST /fireblocks/start` instead | Keys in Fireblocks custody |

Optional tuning: `passphrase`, `gapLimit`, `scanPolicy` (`gap-limit` default, or `index-limit`), `policyStartIndex`, `policyEndIndex`, `historySyncMode` (`polling_http_api` default, `xpub_stream_ws`, `manual_stream_ws`).

## Gotchas Checklist

- [ ] `x-wallet-id` header set on every `/wallet/*` call.
- [ ] `value` / `amount` is an **integer in cents** (no floats).
- [ ] Poll `/wallet/status` until `statusCode === 3` before sending.
- [ ] HTR token uid is `"00"`; omit `token` on balance/send to default to HTR.
- [ ] `allow_external_*_authority_address` flags default to `false` — if you pass an address outside the wallet, opt in.
- [ ] `/wallet/send-tx` output `type: "data"` is mutually exclusive with `address`/`value`.
- [ ] Nano contract dict fields: access with `dictName.a'<base58-address>'` syntax in `fields[]` queries.
- [ ] Atomic swap passwords must be ≥ 3 characters when using the Atomic Swap Service.

## Security

- **Never log seeds.** When using `seed` directly on `/start`, ensure logs are scrubbed.
- Protect the headless endpoint — anyone who can hit `/wallet/*` owns the wallet. Use `http_api_key`, network ACLs, and/or a reverse proxy.
- Starting a wallet in read-only mode (`xpubkey`) is safer for monitoring use-cases.
