# Tx Templates, Per-wallet Config, Custody, Operator

Everything not covered in the other reference files.

---

## Transaction Templates

Templates are a DSL for declaratively specifying a transaction. See [hathor-wallet-lib instructions](https://github.com/HathorNetwork/hathor-wallet-lib/blob/master/src/template/transaction/instructions.ts) for the full instruction set. Both endpoints require `x-wallet-id`.

### `POST /wallet/tx-template/run` — Execute and push a template

Builds, signs, and broadcasts a transaction described by a template instruction array. Use when you want the headless to handle the whole build → sign → push flow in one call.

**Query:** `debug` (bool, optional) — enable debug logs.

**Body:** an array of instructions (template DSL).

Example body — build and push a token creation:

```json
[
  { "type": "input/utxo", "fill": 100, "token": "00" },
  { "type": "token/create", "name": "TestTok", "symbol": "TT", "amount": 10000 },
  { "type": "output/token", "amount": 10000, "token": "{{TOKEN}}", "address": "H..." }
]
```

**Response:**

```json
{
  "success": true,
  "inputs": [...],
  "outputs": [...],
  "hash": "00005d39b210f230...",
  "name": "TestTok",
  "symbol": "TT"
}
```

---

### `POST /wallet/tx-template/build` — Build without pushing

Same as `run` but stops after building the transaction — you get a `txHex` back that you can inspect, sign externally, or broadcast later. Use this for offline-signing or review-before-broadcast flows.

**Query:**
- `debug` (bool, optional)
- `sign` (bool, optional) — if true, sign the tx; otherwise leave it unsigned

**Body:** same as `run`.

**Response:** `{ "success": true, "txHex": "00020101000006e698..." }`

Broadcast separately with `POST /push-tx`.

---

## Config (per-wallet, runtime)

For wallets started with `scanPolicy: index-limit`, these endpoints adjust address loading at runtime. All require `x-wallet-id`.

### `GET /wallet/config/last-loaded-address-index` — Last loaded index

Returns the highest BIP32 derivation index currently loaded by the wallet.

**Response:** `{ "success": true, "index": 74 }`

---

### `POST /wallet/config/index-limit/load-more-addresses` — Load N more addresses

Extends an `index-limit` wallet by loading `count` more addresses past the current end.

**Body:** `count` (int, required, ≥ 1).

**Response:** `{ "success": true }`

---

### `POST /wallet/config/index-limit/last-index` — Set the end index

Sets the upper bound of the `index-limit` address range.

**Body:** `index` (int, required, ≥ 0).

**Response:** `{ "success": true }`

---

## Custody integrations (enterprise / operator)

> **Do not suggest HSM or Fireblocks flows unless the user has explicitly told you they have one configured in `config.js`.** These endpoints depend on external custodial infrastructure and are used only by enterprise deployments. A regular wallet is started via `POST /start` (see `endpoints-core.md`).
>
> HSM support in wallet-headless is built against Dinamo Networks HSMs — see <https://dinamonetworks.com/en/>. Fireblocks requires Fireblocks API credentials.

### `POST /hsm/start` — Start a read-only wallet backed by an HSM

The HSM holds the BIP32 xPriv; this wallet is read-only from the headless' perspective (for signing workflows, the HSM is invoked separately).

**Body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `wallet-id` | string | yes | |
| `hsm-key` | string | yes | Key name on the HSM device that holds the BIP32 xPriv |

**Response:** `{ "success": true }`

Requires HSM configuration in `config.js`. See `docs/hsm.md` in the wallet-headless repo.

---

### `POST /fireblocks/start` — Start a Fireblocks-custodied wallet

Starts a read-only wallet whose signing is delegated to Fireblocks.

**Body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `wallet-id` | string | yes | |
| `xpub` | string | yes | Fireblocks xpub derived to account path `m/44/280/0` |

**Response:** `{ "success": true }`

Requires Fireblocks credentials in `config.js`. See `docs/fireblocks.md` in the wallet-headless repo.

---

## Operator endpoints (avoid by default)

> **Do not call this unless the user explicitly asked you to.** It mutates service-wide state and can restart the process.

### `POST /reload-config` — Reload `config.js`

Re-reads the service config file. A non-recoverable change (seeds, network, etc.) will trigger a service shutdown. This is an operator action, not part of normal wallet workflows.

**Response:** `{ "success": true }`
