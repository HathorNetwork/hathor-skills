# Core Wallet Endpoints

Root + `/wallet/*` lifecycle, queries, and basic send operations. All `/wallet/*` endpoints require the `x-wallet-id` header. Amounts are integers in cents.

## Root

### `POST /start` — Create and start a wallet

Starts a wallet in memory and assigns it an ID for subsequent requests.

**Body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `wallet-id` | string | yes | Identifier used on `x-wallet-id` header afterwards |
| `seedKey` | string | conditional | Key of seed defined in `config.js` (use this OR `seed` OR `xpubkey`) |
| `seed` | string | conditional | 24 words separated by spaces. Avoid in prod |
| `xpubkey` | string | conditional | Account-level xpub for read-only wallet |
| `passphrase` | string | no | Optional BIP39 passphrase |
| `multisig` | boolean | no | Start as multisig; requires multisig config |
| `multisigKey` | string | no | Key of multisig config entry (if different from `seedKey`) |
| `scanPolicy` | enum | no | `gap-limit` (default) or `index-limit` |
| `gapLimit` | int | no | Gap limit for gap-limit policy |
| `policyStartIndex` | int | no | Start loading addresses from this index (default 0) |
| `policyEndIndex` | int | no | Stop at this index (index-limit policy) |
| `historySyncMode` | enum | no | `polling_http_api` (default), `xpub_stream_ws`, `manual_stream_ws` |

**Mutual exclusion (enforced by the service):**

- Exactly one of `seedKey`, `seed`, or `xpubkey` is required.
- `seedKey` and `seed` cannot both be present.
- `xpubkey` cannot be combined with `seedKey` or `seed` (read-only vs. signing).

**Response:** `{ "success": true }`

**Errors:**
- `WALLET_ALREADY_STARTED` (errorCode) — `wallet-id` is already in use; stop the wallet first.
- `"Parameter 'seedKey', 'seed' or 'xpubkey' is required."` — none provided.
- `"You can't have both 'seedKey' and 'seed' in the body."`
- `"You can't start a readonly wallet and send a seed in the same request."` — `xpubkey` mixed with `seed`/`seedKey`.
- `"<key> is not configured for multisig."` — `multisig: true` but no matching entry in `config.multisig`.

---

#### Start from a configured seed (recommended)

The seed is held server-side in `config.js` under a named key — you only send the key.

```bash
curl -X POST http://localhost:8000/start \
  -H 'Content-Type: application/json' \
  -d '{ "wallet-id": "my-wallet", "seedKey": "default" }'
```

#### Start from a raw seed (dev/testing only — avoid in prod)

```bash
curl -X POST http://localhost:8000/start \
  -H 'Content-Type: application/json' \
  -d '{
    "wallet-id": "my-wallet",
    "seed": "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about ... (24 words)"
  }'
```

#### Start a read-only wallet from an xpub

No signing key on the server — it can read balances and history but cannot send transactions.

```bash
curl -X POST http://localhost:8000/start \
  -H 'Content-Type: application/json' \
  -d '{
    "wallet-id": "watch-only",
    "xpubkey": "xpub6C...<account-level xpub>"
  }'
```

#### Start a multisig wallet

Requires `config.multisig` in `config.js` (pubkeys of all participants + `numSignatures`). `multisig: true` tells the service to derive MultiSig addresses; `multisigKey` selects which multisig entry to use (defaults to `seedKey`).

```bash
curl -X POST http://localhost:8000/start \
  -H 'Content-Type: application/json' \
  -d '{
    "wallet-id": "treasury",
    "seedKey": "treasury-signer",
    "multisig": true
  }'
```

---

**Bad-request examples (these will return `success: false`):**

```jsonc
// 1) Sending both seedKey and seed — pick one
{ "wallet-id": "w", "seedKey": "default", "seed": "abandon ..." }

// 2) Mixing xpubkey with a seed — read-only and signing are mutually exclusive
{ "wallet-id": "w", "xpubkey": "xpub6C...", "seedKey": "default" }

// 3) None of seedKey / seed / xpubkey
{ "wallet-id": "w" }

// 4) Multisig requested but no matching config entry
{ "wallet-id": "w", "seedKey": "not-in-config-multisig", "multisig": true }
```

---

### `POST /multisig-pubkey` — Get MultiSig xpub for a seed

Derives the MultiSig xpub for the seed referenced by `seedKey`. Share this xpub with the other participants during multisig setup.

**Body:** `seedKey` (string, required), `passphrase` (string, optional)

**Response:** `{ "success": true, "xpubkey": "xpub..." }`

---

### `POST /push-tx` — Push a signed transaction hex

Broadcasts a fully-signed transaction hex to the network. Use after `tx-proposal` / `p2sh/tx-proposal` / atomic-swap flows, or any time you've signed a transaction externally.

**Body:** `txHex` (string, required)

**Response:**

```json
{ "success": true, "tx": { "hash": "...", ... } }
```

Use with the tx-proposal / p2sh / atomic-swap flows to broadcast a fully signed transaction.

---

### `GET /configuration-string` — Get token configuration string

Returns the canonical `[Name:SYM:uid:checksum]` identifier for a token — the shareable form used to register tokens in wallets.

**Query:** `token` (string, required) — token uid.

**Response:** `{ "success": true, "configurationString": "[Name:SYM:hash:checksum]" }`

---

### `GET /health` — Service health

Liveness / readiness check covering one or more components. Returns 200 if all requested components pass, 503 if any fail.

**Query — at least one is required** (a call with no params returns `"At least one component must be included in the health check"`):
- `wallet_ids` — comma-separated wallet IDs to check
- `include_fullnode=true` — include fullnode health
- `include_tx_mining=true` — include tx-mining-service health

**Response (200 on pass, 503 on fail):**

```json
{
  "status": "pass",
  "description": "Wallet-headless health",
  "checks": {
    "Wallet <id>": [{ "status": "pass", "componentType": "internal", "output": "Wallet is ready" }],
    "fullnode": [{ "status": "pass", "componentType": "fullnode", "output": "Fullnode is responding" }]
  }
}
```

---

## Wallet Status & Addresses

### `GET /wallet/status` — Wallet status

Current sync state, network, and fullnode info. Poll until `statusCode === 3` (Ready) before sending transactions.

**Response:**

```json
{
  "statusCode": 3,
  "statusMessage": "Ready",
  "network": "mainnet",
  "serverUrl": "https://node2.mainnet.hathor.network/v1a/",
  "serverInfo": {
    "version": "0.29.0",
    "network": "mainnet",
    "min_weight": 14,
    "token_deposit_percentage": 0.01
  }
}
```

Status codes: `0` Closed, `1` Connecting, `2` Syncing, `3` Ready.

---

### `GET /wallet/balance` — Balance

Balance for a single token on this wallet. Values are in cents.

**Query:** `token` (string, optional) — token uid; defaults to HTR (`"00"`).

**Response:** `{ "available": 200, "locked": 0 }` (values in cents — 200 = 2.00).

---

### `GET /wallet/address` — Current receiving address

Returns the wallet's current receiving address. Pass `mark_as_used=true` to advance the pointer so the next call returns a fresh address.

**Query (optional):**
- `mark_as_used` (bool) — mark this address as used so the next call returns a fresh one
- `index` (int) — return the address at a specific BIP32 derivation index

**Response:** `{ "address": "H8bt9nYhUNJHg7szF32CWWi1eB8PyYZnbt" }`

---

### `GET /wallet/address-index` — Index of an address

Returns the BIP32 derivation index of an address belonging to this wallet. Useful for offline-signing flows.

**Query:** `address` (string, required)

**Response:** `{ "success": true, "index": 2 }`

Errors if the address does not belong to this wallet.

---

### `GET /wallet/addresses` — All generated addresses

Every address the wallet has generated so far (up to the current scan limit).

**Response:** `{ "addresses": ["H8b...", "HPx...", ...] }`

---

### `GET /wallet/address-info` — Stats for an address

Lifetime totals for one address — received, sent, available, locked — for a given token.

**Query:** `address` (string, required), `token` (string, optional — defaults to HTR).

**Response:**

```json
{
  "success": true,
  "total_amount_received": 9299200,
  "total_amount_sent": 6400,
  "total_amount_available": 9292800,
  "total_amount_locked": 0,
  "token": "00",
  "index": 0
}
```

---

## Transactions — read

### `GET /wallet/transaction` — One transaction from the wallet's history

Full transaction object for a tx already in this wallet's history. Errors if the hash isn't known to the wallet.

**Query:** `id` (string, required) — tx hash.

**Response:**

```json
{
  "tx_id": "0000340349f9...",
  "version": 1,
  "timestamp": 1578430704,
  "is_voided": false,
  "inputs": [...],
  "outputs": [...],
  "parents": [...]
}
```

---

### `GET /wallet/tx-confirmation-blocks` — Confirmations

Number of blocks confirming a transaction — higher = safer. Returns 0 until a block references the tx.

**Query:** `id` (string, required).

**Response:** `{ "success": true, "confirmationNumber": 15 }`

---

### `GET /wallet/tx-history` — History of the wallet

Full transaction history for this wallet, newest first.

**Query:** `limit` (int, optional) — most recent N transactions, sorted by timestamp descending.

**Response:** array of tx objects.

```json
[
  { "tx_id": "0000340349f9...", "version": 1, "timestamp": 1578430704, "is_voided": false, "inputs": [...], "outputs": [...], "parents": [...] },
  { "tx_id": "0000276ec98...",  "version": 1, ... }
]
```

---

## Transactions — send

### `POST /wallet/simple-send-tx` — Single-output send

One-output send (HTR or a custom token). For multi-output, data outputs, or custom input selection, use `/wallet/send-tx`.

**Body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `address` | string | yes | Destination |
| `value` | int | yes | Amount in cents |
| `token` | string \| object | no | Token uid (string) or deprecated `{uid, name, symbol}` object; omit for HTR |
| `change_address` | string | no | Address for change output |

**Response — fields are at the top level, not nested under a `tx` key:**

```json
{
  "success": true,
  "hash": "00001bc7...",
  "nonce": 33440807,
  "timestamp": 1579656120,
  "version": 1,
  "weight": 16.83,
  "signalBits": 0,
  "parents": ["...", "..."],
  "inputs":  [ { "tx_id": "...", "index": 0, "data": { "type": "Buffer", "data": [...] } } ],
  "outputs": [ { "value": 100, "tokenData": 0, "script": { "type": "Buffer", "data": [...] }, "decodedScript": { "address": { "base58": "..." }, "timelock": null }, "token_data": 0 } ],
  "tokens": [],
  "headers": [],
  "_dataToSignCache": { "type": "Buffer", "data": [...] }
}
```

The tx hash is `.hash` (top level). Scripts come back as Node Buffer JSON; `.decodedScript.address.base58` gives the human-readable address.

---

### `POST /wallet/send-tx` — Multi-output / custom-input send

Send to multiple recipients in one transaction, optionally with data outputs or explicit input selection.

**Body:**

- `outputs` (array, required) — each item is one of:
  - P2PKH/P2SH: `{ "address": "...", "value": 100, "token": "00"?, "timelock": 12345? }`
  - Data output: `{ "type": "data", "data": "string" }` (no `address`/`value`)
- `inputs` (array, optional) — either explicit `{ "hash": "...", "index": 0 }` or query-mode `{ "type": "query", "token": "...", "filter_address": "...", "amount_smaller_than": N, "amount_bigger_than": N, "max_utxos": N }`
- `change_address` (string, optional)
- `debug` (bool, optional)

Example:

```json
{
  "outputs": [
    { "address": "H8bt...", "value": 100, "token": "00" },
    { "address": "HPxB...", "value": 50,  "token": "00" },
    { "type": "data", "data": "note attached to this tx" }
  ],
  "change_address": "HC...",
  "inputs": [ { "type": "query", "token": "00", "amount_bigger_than": 1000 } ]
}
```

**Response:** same flat shape as `simple-send-tx` — `success`, `hash`, `inputs`, `outputs`, `parents`, `tokens`, etc. all at top level.

Data outputs and P2PKH/P2SH outputs cannot be mixed in a single output object (but can coexist as separate outputs).

---

### `POST /wallet/decode` — Decode a tx hex

Parses a transaction hex (full or partial) into a structured view, flagging which inputs and outputs belong to this wallet (`mine: true`).

**Body:** `txHex` (string) **or** `partial_tx` (string). At least one required.

**Response:**

```json
{
  "success": true,
  "tx": {
    "completeSignatures": false,
    "tokens": [],
    "outputs": [{ "decoded": {...}, "token": "00", "value": 100, "mine": true }],
    "inputs":  [{ "decoded": {...}, "txId": "...", "index": 0, "mine": true }]
  },
  "balance": { "00": { "tokens": {...}, "authorities": {...} } }
}
```

---

### `PUT /wallet/utxos-selected-as-input` — Lock/unlock UTXOs referenced by a txHex

Marks the UTXOs used as inputs in `txHex` as locked (or unlocked with `mark_as_used: false`), so other API calls won't re-use them during multi-step signing.

**Body:**

- `txHex` (string, required) — hex of a transaction
- `mark_as_used` (bool, optional) — default true; false to unlock
- `ttl` (int, optional) — lock duration in seconds

**Response:** `{ "success": true }`

Useful when orchestrating multi-step signing to prevent other API calls from re-using the same UTXOs.

---

### `POST /wallet/stop` — Stop the wallet

Unloads the wallet from memory. After stopping, requests with this `x-wallet-id` return 404 until the wallet is started again.

**Response:** `{ "success": true }`
