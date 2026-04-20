# Tokens, NFTs, and UTXOs

Create custom tokens, mint/melt, create NFTs, and manage UTXOs. All endpoints require `x-wallet-id`. Amounts in cents.

## Tokens

### `POST /wallet/create-token` — Create a new token

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

**Response:** Full transaction object plus token metadata:

```json
{
  "hash": "00c9b977...",
  "nonce": 200,
  "timestamp": 1610730485,
  "version": 2,
  "weight": 8.0,
  "parents": [...],
  "inputs": [...],
  "outputs": [...],
  "tokens": [],
  "token_name": "Test",
  "token_symbol": "TST",
  "configurationString": "[Test:TST:00c9b977...:a233sac]"
}
```

The token's uid is the `hash` field — save it for subsequent mint/melt/send operations.

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

**Response:** Full transaction object.

---

### `POST /wallet/melt-tokens` — Burn tokens back to HTR

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

**Response:** Full transaction object.

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

**Response:** Transaction with the data embedded in the first output.

NFTs differ from `create-token` mainly in the `data` field and the defaulted-off authorities.

---

## UTXOs

### `GET /wallet/utxo-filter` — List UTXOs matching filters

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
