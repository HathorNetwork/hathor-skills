# Context Object Reference

The `Context` object is passed to all `@public` methods. It provides immutable access to transaction and block data.

## Caller Information

```python
ctx.caller_id                      # Address | ContractId
ctx.get_caller_address()           # Address or None (if called by wallet)
ctx.get_caller_contract_id()       # ContractId or None (if called by contract)
```

Usage pattern:
```python
if caller := ctx.get_caller_address():
    # Called by wallet
    pass
elif contract := ctx.get_caller_contract_id():
    # Called by another contract
    pass
```

## Actions

```python
ctx.actions_list                   # Sequence[NCAction] - all actions
ctx.actions                        # MappingProxyType[TokenUid, tuple[NCAction, ...]]
ctx.get_single_action(token_uid)   # Get exactly one action (raises NCFail if != 1)
```

Action types:
```python
from hathor import NCDepositAction, NCWithdrawalAction, NCGrantAuthorityAction, NCAcquireAuthorityAction

for action in ctx.actions_list:
    if isinstance(action, NCDepositAction):
        token_uid = action.token_uid
        amount = action.amount
    elif isinstance(action, NCWithdrawalAction):
        token_uid = action.token_uid
        amount = action.amount
    elif isinstance(action, NCGrantAuthorityAction):
        token_uid = action.token_uid
        mint = action.mint   # bool
        melt = action.melt   # bool
    elif isinstance(action, NCAcquireAuthorityAction):
        token_uid = action.token_uid
        mint = action.mint
        melt = action.melt
```

## Block Data

```python
ctx.block.timestamp    # int - seconds since epoch (when tx confirmed)
ctx.block.height       # int - blockchain position
ctx.block.hash         # VertexId (bytes)
ctx.timestamp          # DEPRECATED - use ctx.block.timestamp
```

## Transaction (Vertex) Data

```python
ctx.vertex.weight      # float
ctx.vertex.nonce       # int
ctx.vertex.version     # TxVersion
ctx.vertex.parents     # tuple[VertexId, ...]
ctx.vertex.tokens      # tuple[TokenUid, ...] - custom tokens involved
```

### Transaction Inputs
```python
for tx_input in ctx.vertex.inputs:
    tx_input.tx_id     # VertexId - previous tx
    tx_input.index     # int - output index
    tx_input.data      # bytes - signature data
    if tx_input.info:
        tx_input.info.value        # int
        tx_input.info.raw_script   # bytes
        tx_input.info.token_data   # int
```

### Transaction Outputs
```python
for tx_output in ctx.vertex.outputs:
    tx_output.value        # int - amount
    tx_output.raw_script   # bytes
    tx_output.token_data   # int
    if tx_output.parsed_script:
        tx_output.parsed_script.type      # "P2PKH" or "MultiSig"
        tx_output.parsed_script.address   # str (base58)
        tx_output.parsed_script.timelock  # int | None
```

## Common Patterns

### Owner-only access
```python
@public
def admin_only(self, ctx: Context) -> None:
    if ctx.caller_id != self.owner:
        raise NCFail("Unauthorized")
```

### Time-based constraints
```python
@public
def before_deadline(self, ctx: Context) -> None:
    if ctx.block.timestamp > self.deadline:
        raise NCFail("Too late")
```

### Deposit handling
```python
@public(allow_deposit=True)
def deposit(self, ctx: Context) -> None:
    action = ctx.get_single_action(self.token_uid)
    assert isinstance(action, NCDepositAction)
    caller = ctx.get_caller_address()
    self.balances[caller] = self.balances.get(caller, 0) + action.amount
```

## Important Notes

- Context is **immutable** - cannot modify any properties
- `ctx.block` only available after transaction is confirmed
- Actions are validated against method permissions before execution
- Use `get_caller_address()` / `get_caller_contract_id()` for type-safe access
