# End-to-end Examples

Copy-pasteable flows, all using `curl` and `jq`. Set `HEADLESS` and `WALLET` first:

```bash
export HEADLESS=http://127.0.0.1:8000
export WALLET=my-wallet
# If API key auth is configured:
# export APIKEY_HEADER='X-API-KEY: your-key'
```

All examples below omit the optional API-key header for brevity; add `-H "$APIKEY_HEADER"` to every request if your server requires it.

---

## 1. Start a wallet and wait until it is ready

```bash
# Start (using a seed key defined in config.js as "default")
curl -sS -X POST "$HEADLESS/start" \
  -H "Content-Type: application/json" \
  -d "{\"wallet-id\":\"$WALLET\",\"seedKey\":\"default\"}" | jq

# Poll until ready
while :; do
  status=$(curl -sS -H "x-wallet-id: $WALLET" "$HEADLESS/wallet/status" | jq -r '.statusCode')
  echo "status=$status"
  [ "$status" = "3" ] && break
  sleep 2
done
echo "wallet ready"
```

---

## 2. Get an address and the balance

```bash
ADDR=$(curl -sS -H "x-wallet-id: $WALLET" "$HEADLESS/wallet/address" | jq -r '.address')
echo "address: $ADDR"

curl -sS -H "x-wallet-id: $WALLET" "$HEADLESS/wallet/balance" | jq
# → {"available":200,"locked":0}   (2.00 HTR available)
```

---

## 3. Send 1.23 HTR

```bash
curl -sS -X POST "$HEADLESS/wallet/simple-send-tx" \
  -H "x-wallet-id: $WALLET" -H "Content-Type: application/json" \
  -d '{"address":"HNnK9wgUVL6Cjzs1K3jpoGgqQTXCqpAnW8","value":123}' | jq
```

Remember: `value` is in cents. `123` = 1.23 HTR.

---

## 4. Send to multiple outputs in one tx

```bash
curl -sS -X POST "$HEADLESS/wallet/send-tx" \
  -H "x-wallet-id: $WALLET" -H "Content-Type: application/json" \
  -d '{
    "outputs": [
      { "address": "H8bt9nYhUNJHg7szF32CWWi1eB8PyYZnbt", "value": 100 },
      { "address": "HPxB4dKccUWbECh1XMWPEgZVZP2EC34BbB", "value": 50  },
      { "type": "data", "data": "note" }
    ]
  }' | jq
```

---

## 5. Create a custom token, mint more, melt some

```bash
# Create 1000.00 MY tokens
TOKEN_TX=$(curl -sS -X POST "$HEADLESS/wallet/create-token" \
  -H "x-wallet-id: $WALLET" -H "Content-Type: application/json" \
  -d '{"name":"MyToken","symbol":"MY","amount":100000}')
TOKEN_UID=$(echo "$TOKEN_TX" | jq -r '.hash')
echo "token uid: $TOKEN_UID"

# Mint 50.00 more
curl -sS -X POST "$HEADLESS/wallet/mint-tokens" \
  -H "x-wallet-id: $WALLET" -H "Content-Type: application/json" \
  -d "{\"token\":\"$TOKEN_UID\",\"amount\":5000,\"address\":\"$ADDR\"}" | jq

# Melt 10.00 back to HTR
curl -sS -X POST "$HEADLESS/wallet/melt-tokens" \
  -H "x-wallet-id: $WALLET" -H "Content-Type: application/json" \
  -d "{\"token\":\"$TOKEN_UID\",\"amount\":1000}" | jq
```

---

## 6. Create an NFT

```bash
curl -sS -X POST "$HEADLESS/wallet/create-nft" \
  -H "x-wallet-id: $WALLET" -H "Content-Type: application/json" \
  -d '{
    "name": "MyNFT",
    "symbol": "NFT",
    "amount": 1,
    "data": "ipfs://QmExampleHashHere"
  }' | jq
```

---

## 7. Query and consolidate UTXOs

```bash
# See what's available
curl -sS -H "x-wallet-id: $WALLET" \
  "$HEADLESS/wallet/utxo-filter?only_available_utxos=true" | jq

# Merge them all to one address
curl -sS -X POST "$HEADLESS/wallet/utxo-consolidation" \
  -H "x-wallet-id: $WALLET" -H "Content-Type: application/json" \
  -d "{\"destination_address\":\"$ADDR\"}" | jq
```

---

## 8. Create a nano contract and read its state

```bash
# Create an instance of a blueprint
NC=$(curl -sS -X POST "$HEADLESS/wallet/nano-contracts/create" \
  -H "x-wallet-id: $WALLET" -H "Content-Type: application/json" \
  -d "{
    \"blueprint_id\": \"<BLUEPRINT_HASH>\",
    \"address\": \"$ADDR\",
    \"data\": {
      \"args\": [\"arg1\", 42],
      \"actions\": [ { \"type\": \"deposit\", \"token\": \"00\", \"amount\": 1000 } ]
    }
  }")
NC_ID=$(echo "$NC" | jq -r '.hash')
echo "contract id: $NC_ID"

# Read some fields
curl -sS -G -H "x-wallet-id: $WALLET" "$HEADLESS/wallet/nano-contracts/state" \
  --data-urlencode "id=$NC_ID" \
  --data-urlencode "fields[]=total" \
  --data-urlencode "balances[]=00" | jq

# Execute a method
curl -sS -X POST "$HEADLESS/wallet/nano-contracts/execute" \
  -H "x-wallet-id: $WALLET" -H "Content-Type: application/json" \
  -d "{
    \"nc_id\": \"$NC_ID\",
    \"method\": \"bet\",
    \"address\": \"$ADDR\",
    \"data\": {
      \"args\": [\"1x0\"],
      \"actions\": [ { \"type\": \"deposit\", \"token\": \"00\", \"amount\": 500 } ]
    }
  }" | jq
```

---

## 9. Publish an on-chain blueprint from source

```bash
CODE=$(cat my_blueprint.py | jq -Rs .)   # JSON-escape the source
curl -sS -X POST "$HEADLESS/wallet/nano-contracts/create-on-chain-blueprint" \
  -H "x-wallet-id: $WALLET" -H "Content-Type: application/json" \
  -d "{\"code\": $CODE, \"address\": \"$ADDR\"}" | jq
```

The response's `hash` is the new blueprint's ID.

---

## 10. Atomic swap (peer-to-peer, no service)

Alice wants 5.00 MY tokens and will send 1.00 HTR. Bob is the other side.

```bash
# --- Alice's side ---
ALICE=$(curl -sS -X POST "$HEADLESS/wallet/atomic-swap/tx-proposal" \
  -H "x-wallet-id: alice-wallet" -H "Content-Type: application/json" \
  -d "{
    \"receive\": { \"tokens\": [{ \"value\": 500, \"token\": \"$TOKEN_UID\" }] },
    \"send\":    { \"tokens\": [{ \"value\": 100, \"token\": \"00\" }] }
  }")
PARTIAL=$(echo "$ALICE" | jq -r '.data')
# Send $PARTIAL to Bob over any channel.

# --- Bob extends the proposal with his side ---
BOB=$(curl -sS -X POST "$HEADLESS/wallet/atomic-swap/tx-proposal" \
  -H "x-wallet-id: bob-wallet" -H "Content-Type: application/json" \
  -d "{
    \"partial_tx\": \"$PARTIAL\",
    \"receive\":    { \"tokens\": [{ \"value\": 100, \"token\": \"00\" }] },
    \"send\":       { \"tokens\": [{ \"value\": 500, \"token\": \"$TOKEN_UID\" }] }
  }")
PARTIAL=$(echo "$BOB" | jq -r '.data')
# isComplete should now be true.

# --- Each signs ---
ALICE_SIG=$(curl -sS -X POST "$HEADLESS/wallet/atomic-swap/tx-proposal/get-my-signatures" \
  -H "x-wallet-id: alice-wallet" -H "Content-Type: application/json" \
  -d "{\"partial_tx\": \"$PARTIAL\"}" | jq -r '.signatures')

BOB_SIG=$(curl -sS -X POST "$HEADLESS/wallet/atomic-swap/tx-proposal/get-my-signatures" \
  -H "x-wallet-id: bob-wallet" -H "Content-Type: application/json" \
  -d "{\"partial_tx\": \"$PARTIAL\"}" | jq -r '.signatures')

# --- Someone pushes ---
curl -sS -X POST "$HEADLESS/wallet/atomic-swap/tx-proposal/sign-and-push" \
  -H "x-wallet-id: alice-wallet" -H "Content-Type: application/json" \
  -d "{
    \"partial_tx\": \"$PARTIAL\",
    \"signatures\": [\"$ALICE_SIG\", \"$BOB_SIG\"]
  }" | jq
```

---

## 11. Offline-signing flow (single-key)

```bash
# Build
BUILT=$(curl -sS -X POST "$HEADLESS/wallet/tx-proposal" \
  -H "x-wallet-id: $WALLET" -H "Content-Type: application/json" \
  -d '{"outputs":[{"address":"H...","value":100}]}')
TXHEX=$(echo "$BUILT" | jq -r '.txHex')

# Which inputs are mine?
curl -sS -G -H "x-wallet-id: $WALLET" "$HEADLESS/wallet/tx-proposal/get-wallet-inputs" \
  --data-urlencode "txHex=$TXHEX" | jq

# ... sign externally, get raw signatures per input ...

# Convert raw sig to input-data
INPUT_DATA=$(curl -sS -X POST "$HEADLESS/wallet/tx-proposal/input-data" \
  -H "x-wallet-id: $WALLET" -H "Content-Type: application/json" \
  -d '{"index":0,"signature":"<hex-sig>"}' | jq -r '.inputData')

# Attach
SIGNED=$(curl -sS -X POST "$HEADLESS/wallet/tx-proposal/add-signatures" \
  -H "x-wallet-id: $WALLET" -H "Content-Type: application/json" \
  -d "{\"txHex\":\"$TXHEX\",\"signatures\":[{\"index\":0,\"data\":\"$INPUT_DATA\"}]}" | jq -r '.txHex')

# Broadcast
curl -sS -X POST "$HEADLESS/push-tx" \
  -H "Content-Type: application/json" \
  -d "{\"txHex\":\"$SIGNED\"}" | jq
```

---

## 12. Clean up

```bash
curl -sS -X POST "$HEADLESS/wallet/stop" -H "x-wallet-id: $WALLET"
```
