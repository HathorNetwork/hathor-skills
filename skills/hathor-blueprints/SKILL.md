---
name: hathor-blueprint
description: Create Hathor blockchain blueprints (nano contracts) - Python 3.11 smart contracts. Use when building, debugging, or modifying Hathor nano contracts, or when user mentions blueprints, nano contracts, or Hathor smart contracts.
---

# Hathor Blueprint Reference

Blueprints are Python 3.11 smart contracts for Hathor blockchain.

## Allowed Imports

Only these imports are permitted (sandboxed environment):

```python
# Standard library (only these)
from math import ceil, floor
from typing import Optional, NamedTuple, TypeAlias, Union
from collections import OrderedDict

# Hathor module
from hathor import (
    Address, Amount, Blueprint, BlueprintId, CallerId, Context, ContractId,
    HATHOR_TOKEN_UID, NCAcquireAuthorityAction, NCAction, NCActionType,
    NCArgs, NCDepositAction, NCFail, NCFee, NCGrantAuthorityAction,
    NCParsedArgs, NCRawArgs, NCWithdrawalAction, SignedData, Timestamp,
    TokenUid, TxOutputScript, VertexId, export, fallback, public, sha3,
    verify_ecdsa, view
)
```

Forbidden: `import x`, `hashlib`, `os`, `json`, `datetime`, `requests`, `typing.Dict`, `typing.List`

## Critical Rules

1. **Use `@export` decorator** on Blueprint class
2. **Use `initialize()` not `__init__`** - must be `@public` with `ctx: Context`
3. **Initialize ALL attributes** in `initialize()` - including empty containers
4. **No default parameters** in `@public` or `@view` methods
5. **`@public` methods** require `ctx: Context` first param, can modify state
6. **`@view` methods** have no `ctx`, read-only (raises `NCViewMethodError` if writes)

## Blueprint Template

```python
from hathor import Blueprint, public, view, Context, export, Address, NCFail

@export
class MyContract(Blueprint):
    owner: Address
    counter: int
    balances: dict[Address, int]  # DictContainer, not Python dict!
    history: list[str]            # DequeContainer, not Python list!
    members: set[Address]         # SetContainer, not Python set!

    @public
    def initialize(self, ctx: Context, owner: Address) -> None:
        self.owner = owner
        self.counter = 0
        self.balances = {}    # Must initialize all containers
        self.history = []
        self.members = set()

    @public
    def increment(self, ctx: Context) -> None:
        self.counter += 1

    @view
    def get_counter(self) -> int:
        return self.counter
```

## Container Types (Critical)

Hathor containers are NOT standard Python - they persist to blockchain storage.

| Declared | Actual Type | Iteration | Key Methods |
|----------|-------------|-----------|-------------|
| `dict[K,V]` | DictContainer | ❌ No | `[]`, `.get()`, `in`, `del`, `len`, `.update()` |
| `list[T]` | DequeContainer | ✅ Yes | `[]`, `.append()`, `.appendleft()`, `.pop()`, `.popleft()`, `len` |
| `set[T]` | SetContainer | ❌ No | `.add()`, `.discard()`, `.remove()`, `in`, `len`, `.update()` |

**Not implemented**: `.keys()`, `.values()`, `.items()`, `.copy()`, `.clear()`, `.sort()`, `.insert()`, `.union()`

See [references/containers.md](references/containers.md) for full operation list.

## Decorator Parameters

```python
@public(
    allow_deposit=True,           # Allow token deposits
    allow_withdrawal=True,        # Allow token withdrawals
    allow_grant_authority=True,   # Grant mint/melt authority
    allow_acquire_authority=True, # Acquire mint/melt authority
    allow_reentrancy=False        # Prevent recursive calls (default)
)
def my_method(self, ctx: Context) -> None: ...
```

## Context Object Quick Reference

```python
ctx.caller_id                    # Address | ContractId
ctx.get_caller_address()         # Address or None
ctx.get_caller_contract_id()     # ContractId or None
ctx.actions_list                 # Sequence[NCAction]
ctx.get_single_action(token_uid) # Get exactly one action (raises if != 1)
ctx.block.timestamp              # int (seconds since epoch)
ctx.block.height                 # int
ctx.vertex.hash                  # bytes (32 bytes tx hash)
```

See [references/context.md](references/context.md) for full API.

## Error Handling

```python
from hathor import NCFail

class InsufficientBalance(NCFail):
    pass

@public
def withdraw(self, ctx: Context, amount: int) -> None:
    if amount <= 0:
        raise NCFail("Amount must be positive")
    if self.balances.get(ctx.get_caller_address(), 0) < amount:
        raise InsufficientBalance()
```

## Deposits & Withdrawals

```python
@public(allow_deposit=True)
def deposit(self, ctx: Context) -> None:
    action = ctx.get_single_action(self.token_uid)
    assert isinstance(action, NCDepositAction)
    caller = ctx.get_caller_address()
    self.balances[caller] = self.balances.get(caller, 0) + action.amount

@public(allow_withdrawal=True)
def withdraw(self, ctx: Context) -> None:
    action = ctx.get_single_action(self.token_uid)
    if action.amount > self.balances[ctx.get_caller_address()]:
        raise NCFail("Insufficient balance")
```

## Advanced Features

Access via `self.syscall`:
- `get_contract()` - Interface to call other contracts
- `create_deposit_token()`, `create_fee_token()`, `mint_tokens()`, `melt_tokens()` - Token management
- `get_current_balance()`, `can_mint()`, `can_melt()` - Queries
- `rng` - Built-in deterministic RNG (ChaCha20-based)
- `emit_event()` - Emit custom events

See [references/syscall.md](references/syscall.md) for details.

## Reference Files

- [references/containers.md](references/containers.md) - Full container operations
- [references/context.md](references/context.md) - Complete Context API
- [references/syscall.md](references/syscall.md) - Advanced syscall features & RNG
- [references/examples.md](references/examples.md) - Complete contract examples
