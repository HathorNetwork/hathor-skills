# Tokens, NFTs, and UTXOs

Create custom tokens, mint/melt, create NFTs, and manage UTXOs. All endpoints require `x-wallet-id`. Amounts in cents.

## Tokens

### `POST /wallet/create-token` — Create a new token

Creates a new custom token. The top-level `hash` in the response is the new token's uid — save it for subsequent mint/melt/send calls. `version: 1` is a DEPOSIT token (default), `version: 2` is a FEE token.

**Body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | yes | Token name |
| `symbol` | string | yes | Token symbol |
| `amount` | int | yes | Initial mint amount (in cents) |
| `address` | string | no | Destination of minted tokens (default: wallet address) |
| `change_address` | string | no | Change address for HTR deposit |
| `create_mint` | bool | no | Create mint authority (default: true) |
| `mint_authority_address` | string | no | Recipient of mint authority |
| `allow_external_mint_authority_address` | bool | no | Allow address outside wallet (default: false) |
| `create_melt` | bool | no | Create melt authority (default: true) |
| `melt_authority_address` | string | no | Recipient of melt authority |
| `allow_external_melt_authority_address` | bool | no | Allow address outside wallet (default: false) |
| `data` | string[] | no | UTF-8 data outputs to attach |
| `version` | int | no | `1` = DEPOSIT token (default), `2` = FEE token |

**Response — flat transaction object plus token metadata at the top level:**

```json
{
  "success": true,
  "hash": "00000c7aedb78f26...",
  "nonce": 310758,
  "timestamp": 1776719293,
  "version": 2,
  "weight": 18.02,
  "signalBits": 0,
  "parents": [...],
  "inputs":  [...],
  "outputs": [...],
  "tokens": [],
  "headers": [],
  "name": "MyToken",
  "symbol": "MY",
  "tokenVersion": 1,
  "configurationString": "[MyToken:MY:00000c7a...:9b08c6a7]"
}
```

The token's uid is `.hash` (top level) — save it for subsequent mint/melt/send operations. `.configurationString` is the shareable `[Name:SYM:uid:checksum]` identifier.

---

### `POST /wallet/mint-tokens` — Mint more of a token

Requires this wallet to hold the mint authority for `token`.

**Body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `token` | string | yes | Token uid |
| `amount` | int | yes | Cents to mint |
| `address` | string | no | Destination (default: wallet address) |
| `change_address` | string | no | |
| `mint_authority_address` | string | no | New mint authority recipient |
| `allow_external_mint_authority_address` | bool | no | Default false |
| `unshift_data` | bool | no | Prepend data outputs (default: true) |
| `data` | string[] | no | Data outputs |

**Response:** flat transaction object — `success`, `hash`, `inputs`, `outputs`, `tokens`, `parents`, etc. at the top level (same shape as `create-token` but without `name` / `symbol` / `configurationString`).

---

### `POST /wallet/melt-tokens` — Burn tokens back to HTR

Burns `amount` of `token`, refunding the HTR deposit to the wallet (or to `deposit_address`). Requires this wallet to hold the melt authority for the token.

**Body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `token` | string | yes | Token uid |
| `amount` | int | yes | Cents to melt |
| `change_address` | string | no | Token change address |
| `deposit_address` | string | no | Where the refunded HTR goes |
| `melt_authority_address` | string | no | New melt authority recipient |
| `allow_external_melt_authority_address` | bool | no | Default false |
| `unshift_data` | bool | no | Prepend data outputs (default: true) |
| `data` | string[] | no | Data outputs |

**Response:** flat transaction object — `success`, `hash`, `inputs`, `outputs`, `tokens`, `parents`, etc. at the top level (same shape as `create-token` but without `name` / `symbol` / `configurationString`).

---

## NFTs

### `POST /wallet/create-nft` — Create an NFT

NFTs are tokens with a data field (usually an IPFS hash). Typical amount is `1`.

**Body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | yes | |
| `symbol` | string | yes | |
| `amount` | int | yes | Usually `1` |
| `data` | string | yes | Payload (e.g. `"ipfs://Qm..."`) |
| `address` | string | no | Destination |
| `change_address` | string | no | |
| `create_mint` | bool | no | Default **false** for NFTs |
| `mint_authority_address` | string | no | |
| `allow_external_mint_authority_address` | bool | no | Default false |
| `create_melt` | bool | no | Default **false** for NFTs |
| `melt_authority_address` | string | no | |
| `allow_external_melt_authority_address` | bool | no | Default false |

**Response:** same flat shape as `create-token` (top-level `hash`, `configurationString`, `name`, `symbol`, `tokenVersion`, etc.) — the NFT `data` lives as a data output inside the `outputs` array.

NFTs differ from `create-token` mainly in the `data` field and the defaulted-off authorities.

---

## UTXOs

### `GET /wallet/utxo-filter` — List UTXOs matching filters

Lists UTXOs owned by this wallet, filtered by token, address, and/or amount range. Use to inspect holdings before building a custom `/wallet/send-tx` with explicit inputs.

**Query (all optional):**

| Param | Type | Notes |
|-------|------|-------|
| `max_utxos` | int | Default 255 |
| `token` | string | Default HTR |
| `filter_address` | string | Only UTXOs on this address |
| `amount_smaller_than` | int | Upper bound (cents) |
| `amount_bigger_than` | int | Lower bound (cents) |
| `maximum_amount` | int | Stop once sum reaches this |
| `only_available_utxos` | bool | Exclude locked UTXOs |

**Response:**

```json
{
  "total_amount_available": 12000,
  "total_utxos_available": 2,
  "total_amount_locked": 6000,
  "total_utxos_locked": 1,
  "utxos": [
    {
      "address": "HNnK9wgUVL6Cjzs1K3jpoGgqQTXCqpAnW8",
      "amount": 6000,
      "tx_id": "00fff7a3...",
      "locked": false,
      "index": 0
    }
  ]
}
```

---

### `POST /wallet/utxo-consolidation` — Merge UTXOs to one address

Useful for reducing fragmentation / dust on a wallet.

**Body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `destination_address` | string | yes | Where consolidated UTXOs go |
| `max_utxos` | int | no | Default 255 |
| `token` | string | no | Default HTR |
| `filter_address` | string | no | |
| `amount_smaller_than` | int | no | |
| `amount_bigger_than` | int | no | |
| `maximum_amount` | int | no | |

**Response:**

```json
{
  "success": true,
  "txId": "0000002b...",
  "total_utxos_consolidated": 8,
  "total_amount": 140800,
  "utxos": [ ... ]
}
```
