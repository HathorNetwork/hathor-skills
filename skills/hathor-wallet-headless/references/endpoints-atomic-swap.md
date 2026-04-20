# Atomic Swap

Two parties exchange tokens in a single transaction — no counterparty risk. The Hathor Atomic Swap Service (optional) coordinates proposal discovery and persistence; you can also swap fully peer-to-peer by shipping the `partial_tx` blob over any channel.

All endpoints below require `x-wallet-id`.

## Flow at a glance

1. **Alice builds a proposal** — `POST /wallet/atomic-swap/tx-proposal` with what she wants to receive and what she will send. Gets back a `partial_tx` blob.
2. **Bob reviews and counter-builds** — same endpoint, but passes `partial_tx` to update it with his own side. The response's `isComplete` becomes `true` once both sides balance.
3. **Each party signs** — `POST /wallet/atomic-swap/tx-proposal/get-my-signatures` → signature blob.
4. **Someone aggregates & pushes** — `POST /wallet/atomic-swap/tx-proposal/sign-and-push` (or `sign` + `/push-tx`).

## Endpoints

### `POST /wallet/atomic-swap/tx-proposal` — Build or update a proposal

**Body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `partial_tx` | string | conditional | Existing proposal to extend. Omit for a new one |
| `receive.tokens` | object[] | no | What this caller wants to receive |
| `receive.tokens[].value` | int | yes | In cents |
| `receive.tokens[].token` | string | no | Default HTR |
| `receive.tokens[].address` | string | no | Optional receive address |
| `send.tokens` | object[] | no | What this caller offers |
| `send.tokens[].value` | int | yes | In cents |
| `send.tokens[].token` | string | no | |
| `send.utxos` | object[] | no | Force specific UTXOs: `{ txId, index }` |
| `lock` | bool | no | Lock UTXOs referenced in `send` (default true) |
| `change_address` | string | no | |
| `service.is_new` | bool | no | Create a new proposal in the Atomic Swap Service |
| `service.proposal_id` | string | no | Existing service proposal ID (for updates) |
| `service.password` | string | conditional | Required for service proposals; ≥ 3 chars |
| `service.version` | int | no | Proposal version (for optimistic concurrency) |

**Response:**

```json
{
  "success": true,
  "data": "PartialTx|...",
  "isComplete": false,
  "createdProposalId": "b11948c7-48..."
}
```

`isComplete: true` means the inputs and outputs balance — ready to sign.

`createdProposalId` only present when `service.is_new: true`.

---

### `POST /wallet/atomic-swap/tx-proposal/get-my-signatures` — Sign

**Body:** `partial_tx` (required).

**Response:** `{ "success": true, "signatures": "PartialTxInputData|..." }`

Share this blob with the other party (or aggregator).

---

### `POST /wallet/atomic-swap/tx-proposal/sign` — Attach collected signatures

**Body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `partial_tx` | string | yes | |
| `signatures` | string[] | yes | All participants' signature blobs |
| `service.proposal_id` | string | no | For service-tracked proposals |
| `service.version` | int | no | |

**Response:** `{ "success": true, "txHex": "..." }` — broadcast with `/push-tx`.

---

### `POST /wallet/atomic-swap/tx-proposal/sign-and-push` — Sign and broadcast

**Body:** `partial_tx` (required), `signatures` (string[], required).

**Response:** `{ "success": true, "tx": { ... } }`

---

### `POST /wallet/atomic-swap/tx-proposal/unlock` — Free UTXOs

Release UTXOs that were locked by `lock: true` on the proposal build.

**Body:** `partial_tx` (required).

**Response:** `{ "success": true }`

---

### `GET /wallet/atomic-swap/tx-proposal/get-locked-utxos` — Inspect locks

**Response:**

```json
{
  "success": true,
  "locked_utxos": [
    { "tx_id": "006e18f3...", "outputs": [0, 1] }
  ]
}
```

---

### `POST /wallet/atomic-swap/tx-proposal/get-input-data` — Extract signatures from a signed hex

Given a fully-signed `txHex`, return the `signatures` blob representing this wallet's contribution — useful for rewinding to the signature-exchange stage.

**Body:** `txHex` (required).

**Response:** `{ "success": true, "signatures": "PartialTxInputData|..." }`

---

## Atomic Swap Service endpoints

Used when the optional service is configured (backend that stores proposals so parties can sync asynchronously).

### `POST /wallet/atomic-swap/tx-proposal/register/{proposalId}` — Register a proposal

**Body:** `password` (string, required).

**Response:** `{ "success": true }`

---

### `GET /wallet/atomic-swap/tx-proposal/fetch/{proposalId}` — Fetch from service

**Response:**

```json
{
  "success": true,
  "proposal": {
    "proposalId": "1a574e6c-73...",
    "version": 1,
    "timestamp": "Fri Mar 10 2023...",
    "partialTx": "PartialTx|000100010...",
    "signatures": null,
    "history": []
  }
}
```

---

### `GET /wallet/atomic-swap/tx-proposal/list` — List registered proposals

**Response:** `{ "success": true, "proposals": ["1a574e6c-...", "85585de5-..."] }`

---

### `DELETE /wallet/atomic-swap/tx-proposal/delete/{proposalId}` — Unregister

**Response:** `{ "success": true }`

---

## Gotchas

- **Password length.** Service proposals require a password ≥ 3 chars — shorter strings error.
- **Lock defaults to true.** If you build a proposal and abandon it, remember to `/unlock` or the UTXOs stay held.
- **`isComplete` check.** Don't attempt to sign a proposal while `isComplete` is false — the transaction is not balanced yet.
- **Version drift.** When updating a service proposal, pass `service.version` matching what you fetched to avoid clobbering concurrent edits.
