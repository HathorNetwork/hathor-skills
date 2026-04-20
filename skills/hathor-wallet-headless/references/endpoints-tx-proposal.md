# Transaction Proposals & P2SH

Offline-signing flows. Build an unsigned transaction, collect signatures (potentially from multiple parties or a separate signer device), attach them, then push. All `/wallet/*` endpoints require `x-wallet-id`.

The two families:

- **`/wallet/tx-proposal/*`** — single-key wallets, offline signing.
- **`/wallet/p2sh/tx-proposal/*`** — multisig (P2SH) wallets, multi-party signing.

After either flow, use `POST /push-tx` (root) to broadcast.

---

## Regular tx-proposal (single-key)

### `POST /wallet/tx-proposal` — Build an unsigned tx

**Body:** same `outputs` / `inputs` / `change_address` shape as `/wallet/send-tx` (see [endpoints-core.md](endpoints-core.md)).

**Response:** `{ "success": true, "txHex": "0123...", "dataToSignHash": "abcd..." }`

`dataToSignHash` is what an external signer signs per input.

---

### `GET /wallet/tx-proposal/get-wallet-inputs` — Which inputs belong to this wallet

**Query:** `txHex` (string, required).

**Response:**

```json
{
  "inputs": [
    { "inputIndex": 0, "addressIndex": 1, "addressPath": "m/44'/280'/0'/0/1" }
  ]
}
```

Use when delegating signing to the wallet for just some of the inputs.

---

### `POST /wallet/tx-proposal/add-signatures` — Attach collected signatures

**Body:**

```json
{
  "txHex": "0123...",
  "signatures": [
    { "index": 0, "data": "abc123..." },
    { "index": 1, "data": "def456..." }
  ]
}
```

**Response:** `{ "success": true, "txHex": "..." }` (signed hex, ready for `/push-tx`).

---

### `POST /wallet/tx-proposal/input-data` — Build input-data from raw signatures

Converts ECDSA signatures into the wire-format input-data blob.

**Body (P2PKH):**

```json
{ "index": 1, "signature": "abc..." }
```

- `index` — BIP32 path index of the signing address.
- `signature` — P2PKH: ECDSA sig, little-endian DER hex.

**Body (P2SH):**

```json
{
  "signatures": {
    "xpub1...": "abc...",
    "xpub2...": "def..."
  }
}
```

**Response:** `{ "success": true, "inputData": "..." }`

---

## P2SH tx-proposal (multisig)

A P2SH wallet is started with multiple participant xpubs. A transaction needs signatures from a threshold of them before it can be broadcast.

### `POST /wallet/p2sh/tx-proposal` — Build unsigned P2SH tx

**Body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `outputs` | object[] | yes | Each: `{ address, value, token? }` |
| `inputs` | object[] | no | Explicit `{ txId, index }` entries |
| `change_address` | string | no | |
| `mark_inputs_as_used` | bool | no | Lock UTXOs while signing |

**Response:** `{ "success": true, "txHex": "..." }`

Specialized variants build common token operations as P2SH proposals:

- `POST /wallet/p2sh/tx-proposal/create-token` — body mirrors `/wallet/create-token`.
- `POST /wallet/p2sh/tx-proposal/mint-tokens` — mirrors `/wallet/mint-tokens`.
- `POST /wallet/p2sh/tx-proposal/melt-tokens` — mirrors `/wallet/melt-tokens`.

All return an unsigned `txHex`.

---

### `POST /wallet/p2sh/tx-proposal/get-my-signatures` — Sign with this wallet's key

**Body:** `txHex` (required).

**Response:** `{ "success": true, "signatures": "..." }` — a serialized blob to share with other participants.

---

### `POST /wallet/p2sh/tx-proposal/sign` — Apply collected signatures

**Body:** `txHex` (required), `signatures` (array of per-participant signature blobs, optional).

**Response:** `{ "success": true, "txHex": "..." }` (fully signed hex).

---

### `POST /wallet/p2sh/tx-proposal/sign-and-push` — Sign and broadcast

**Body:** `txHex` (required), `signatures` (array, optional).

**Response:** `{ "success": true, "hash": "...", ... }`

Convenience endpoint that combines `sign` + `push-tx`. Use when all signatures are already gathered.

---

## Typical offline-signing flow

1. Build: `POST /wallet/tx-proposal` → get `txHex`, `dataToSignHash`.
2. Identify which inputs are yours: `GET /wallet/tx-proposal/get-wallet-inputs`.
3. Sign externally (hardware wallet, air-gapped machine, etc.).
4. Convert raw signatures into input-data: `POST /wallet/tx-proposal/input-data`.
5. Attach: `POST /wallet/tx-proposal/add-signatures`.
6. Broadcast: `POST /push-tx`.

## Typical P2SH flow

1. Each participant starts the same multisig wallet with their own headless instance.
2. One participant builds: `POST /wallet/p2sh/tx-proposal` → shares `txHex`.
3. Each participant signs: `POST /wallet/p2sh/tx-proposal/get-my-signatures` → shares the signature blob back.
4. Aggregator calls `POST /wallet/p2sh/tx-proposal/sign-and-push` with the collected signatures.
