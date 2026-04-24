# Nano Contracts

All endpoints under `/wallet/nano-contracts/` require `x-wallet-id`. Nano contracts are created from *blueprints* — either built-in or an on-chain Python blueprint uploaded via `/create-on-chain-blueprint`. For authoring blueprints, use the companion `hathor-blueprint` skill.

## Read endpoints

### `GET /wallet/nano-contracts/state` — Contract state

Read one or more fields, token balances, or view-method results from a live nano contract.

**Query:**

| Param | Type | Required | Notes |
|-------|------|----------|-------|
| `id` | string | yes | Nano contract ID |
| `fields[]` | string[] | no | Field names to read (e.g. `token_uid`, `total`) |
| `balances[]` | string[] | no | Token uids whose balance to return (`00` for HTR) |
| `calls[]` | string[] | no | Private/view methods to invoke |
| `block_hash` | string | no | Historical state at a block |
| `block_height` | int | no | Historical state at a block height |

**Dict-field access:** for a dict-typed field `withdrawals`, read a specific key with base58 address wrapped in single quotes:

```
fields[]=withdrawals.a'Wi8zvxdXHjaUVAoCJf52t3WovTZYcU9aX6'
```

**Response:**

```json
{
  "success": true,
  "nc_id": "3cb032600bdf7db...",
  "blueprint_name": "Bet",
  "fields": {
    "token_uid": { "value": "00" },
    "total":     { "value": 300 },
    "final_result": { "value": "1x0" },
    "withdrawals.a'Wi8zv...'": { "value": 300 }
  }
}
```

---

### `GET /wallet/nano-contracts/history` — Contract call history

Every transaction that invoked a method on this nano contract, newest first.

**Query:** `id` (required), `count` (optional, default 100), `after` (optional tx hash), `before` (optional tx hash).

**Response:**

```json
{
  "success": true,
  "count": 100,
  "history": [
    {
      "hash": "5c02adea056d7b43...",
      "nonce": 0,
      "timestamp": 1572636346,
      "nc_id": "5c02adea...",
      "nc_method": "initialize",
      "nc_args": "0004313233340001...",
      "nc_pubkey": "033f5d238afaa9e..."
    }
  ]
}
```

---

### `GET /wallet/nano-contracts/oracle-data` — Resolve oracle data

Resolves an oracle parameter (base58 address or raw hex) into the canonical oracle-data hex that blueprints consume.

**Query:** `oracle` (required) — address (base58) or oracle data (hex).

**Response:** `{ "success": true, "oracleData": "12345678" }`

---

### `GET /wallet/nano-contracts/oracle-signed-result` — Get a result signed by the oracle

Produces an oracle-signed result blob that a blueprint can verify on-chain (e.g. to settle a bet).

**Query:**

| Param | Type | Required | Notes |
|-------|------|----------|-------|
| `oracle_data` | string | yes | Address or hex oracle data |
| `contract_id` | string | yes | Nano contract ID (hex) |
| `result` | string/int/number/bool | yes | Result value to sign |
| `type` | string | yes | Result type (`str`, `int`, `bool`, ...) |

**Response:** `{ "success": true, "oracleData": "12345678:1x0:str" }`

---

## Write endpoints

### `POST /wallet/nano-contracts/create` — Instantiate a contract from a blueprint

Creates a new nano contract by calling the blueprint's `initialize` method. The response's `nc_id` (= transaction hash) is the contract ID you'll pass to future `execute` / `state` calls.

**Body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `blueprint_id` | string | yes | Blueprint to instantiate |
| `address` | string | yes | Caller address (must belong to this wallet) |
| `data.args` | array | no | Constructor arguments (for `initialize`) |
| `data.actions` | array | no | Deposit/withdrawal/authority actions (see below) |
| `create_token_options` | object | no | If the blueprint's initialize also creates a token (see below) |
| `max_fee` | int | no | Upper bound on fee (cents) |
| `contract_pays_fees` | bool | no | If true, the contract pays fees instead of the caller (default false) |

### Action shapes

Each action object's required fields depend on `type`:

```json
// Token in/out — uses amount
{ "type": "deposit",    "token": "00", "amount": 1000 }
{ "type": "withdrawal", "token": "00", "amount": 1000, "address": "Hxxx..." }

// Authority in/out — uses authority ("mint" or "melt"), not amount
{ "type": "grant_authority",   "token": "<token-uid>", "authority": "mint",
  "address": "Hxxx...", "authorityAddress": "Hyyy..." }

{ "type": "acquire_authority", "token": "<token-uid>", "authority": "mint",
  "address": "Hxxx..." }
```

- `deposit` — send `amount` of `token` from caller into the contract.
  - `address` (optional): the caller address to pull the UTXO from. If omitted, the wallet picks any of its UTXOs.
  - `changeAddress` (optional): where to send change.
- `withdrawal` — pull `amount` of `token` out of the contract to `address` (required).
- `grant_authority` — hand a mint or melt authority **over `token`** to the contract.
  - `address` (optional): the caller address that holds the authority UTXO being spent. If omitted, the wallet picks one of its own authority UTXOs.
  - `authorityAddress` (optional): if set, creates a **second authority output** at this address so the caller *keeps a copy* of the authority after granting. Must belong to this wallet. Omit to fully transfer the authority to the contract.
  - `changeAddress` (optional): change address for any non-authority value pulled from the input UTXO.
- `acquire_authority` — take a mint or melt authority out of the contract to `address` (required).

### `create_token_options` shape

Used when the blueprint's `initialize` will create a token as part of contract setup. Same fields as `/wallet/create-token` plus:

| Field | Type | Notes |
|-------|------|-------|
| `contract_pays_deposit` | bool | Contract pays the HTR deposit required for token creation |
| `mint_address` | string | Address that receives the minted tokens |
| `is_create_nft` | bool | If true, create as NFT (default false) |

**Response:** transaction object with `nc_id` (new contract ID).

---

### `POST /wallet/nano-contracts/execute` — Call a method on an existing contract

Invokes a public method on a live nano contract. `data.actions` can move tokens or authorities in/out of the contract (see Action shapes above).

**Body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `nc_id` | string | yes | Contract ID |
| `method` | string | yes | Method name |
| `address` | string | yes | Caller address (must belong to this wallet) |
| `data.args` | array | no | Method arguments |
| `data.actions` | array | no | Same shape as `create` |
| `max_fee` | int | no | |
| `contract_pays_fees` | bool | no | |

**Response:** transaction object.

---

### `POST /wallet/nano-contracts/create-on-chain-blueprint` — Publish a blueprint

Uploads Python source code as a new on-chain blueprint, making it available for `create` calls.

**Body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `code` | string | yes | Python source |
| `address` | string | yes | Caller address (must belong to wallet) |

**Response:**

```json
{
  "success": true,
  "hash": "000001b28c9dcffd...",
  "version": 6,
  "weight": 21.99,
  "code": { "kind": "python+gzip", "content": { "type": "Buffer", "data": [...] } }
}
```

The `hash` is the new blueprint's ID.

> For writing the Python blueprint source, see the `hathor-blueprint` skill.
