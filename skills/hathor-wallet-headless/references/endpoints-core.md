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
| `multisigKey` | string | no | Key of multisig config entry |
| `scanPolicy` | enum | no | `gap-limit` (default) or `index-limit` |
| `gapLimit` | int | no | Gap limit for gap-limit policy |
| `policyStartIndex` | int | no | Start loading addresses from this index (default 0) |
| `policyEndIndex` | int | no | Stop at this index (index-limit policy) |
| `historySyncMode` | enum | no | `polling_http_api` (default), `xpub_stream_ws`, `manual_stream_ws` |

**Response:** `{ "success": true }`

**Errors:** `WALLET_ALREADY_STARTED` (errorCode) if `wallet-id` collides; seed/seedKey not found.

---

### `POST /multisig-pubkey` — Get MultiSig xpub for a seed

**Body:** `seedKey` (string, required), `passphrase` (string, optional)

**Response:** `{ "success": true, "xpubkey": "xpub..." }`

---

### `POST /push-tx` — Push a signed transaction hex

**Body:** `txHex` (string, required)

**Response:**

```json
{ "success": true, "tx": { "hash": "...", ... } }
```

Use with the tx-proposal / p2sh / atomic-swap flows to broadcast a fully signed transaction.

---

### `GET /configuration-string` — Get token configuration string

**Query:** `token` (string, required) — token uid.

**Response:** `{ "success": true, "configurationString": "[Name:SYM:hash:checksum]" }`

---

### `POST /reload-config` — Reload config file

**Response:** `{ "success": true }`

A non-recoverable change will trigger a service shutdown.

---

### `GET /health` — Service health

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

**Query:** `token` (string, optional) — token uid; defaults to HTR (`"00"`).

**Response:** `{ "available": 200, "locked": 0 }` (values in cents — 200 = 2.00).

---

### `GET /wallet/address` — Current receiving address

**Query (optional):**
- `mark_as_used` (bool) — mark this address as used so the next call returns a fresh one
- `index` (int) — return the address at a specific BIP32 derivation index

**Response:** `{ "address": "H8bt9nYhUNJHg7szF32CWWi1eB8PyYZnbt" }`

---

### `GET /wallet/address-index` — Index of an address

**Query:** `address` (string, required)

**Response:** `{ "success": true, "index": 2 }`

Errors if the address does not belong to this wallet.

---

### `GET /wallet/addresses` — All generated addresses

**Response:** `{ "addresses": ["H8b...", "HPx...", ...] }`

---

### `GET /wallet/address-info` — Stats for an address

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

**Query:** `id` (string, required).

**Response:** `{ "success": true, "confirmationNumber": 15 }`

---

### `GET /wallet/tx-history` — History of the wallet

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

**Body:**

- `txHex` (string, required) — hex of a transaction
- `mark_as_used` (bool, optional) — default true; false to unlock
- `ttl` (int, optional) — lock duration in seconds

**Response:** `{ "success": true }`

Useful when orchestrating multi-step signing to prevent other API calls from re-using the same UTXOs.

---

### `POST /wallet/stop` — Stop the wallet

**Response:** `{ "success": true }`

After stopping, `x-wallet-id` for that ID becomes invalid until started again.
