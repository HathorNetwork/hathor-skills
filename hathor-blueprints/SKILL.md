---
name: hathor-blueprint
description: Create Hathor blockchain blueprints (nano contracts) - Python 3.11 smart contracts. Use when building, debugging, or modifying Hathor nano contracts, or when user mentions blueprints, nano contracts, or Hathor smart contracts. Covers class structure, decorators (@public/@view), initialization patterns, container field rules, and the validate-compile-test workflow.
---

# Hathor Blueprint Reference

Blueprints are Python 3.11 smart contracts for the Hathor blockchain. This reference covers the complete API, patterns, and restrictions.

## Critical Import Restrictions

### Only These Imports Are Allowed

**NEVER use any imports other than these exact imports:**

#### Standard Python Imports (ONLY these specific ones)
```python
# Math functions
from math import ceil, floor

# Typing utilities
from typing import Optional, NamedTuple, TypeAlias, Union

# Collections
from collections import OrderedDict
```

#### Hathor-Specific Imports (ONLY from hathor module)
```python
from hathor import (
    Address,
    Amount,
    Blueprint,
    BlueprintId,
    CallerId,
    Context,
    ContractId,
    HATHOR_TOKEN_UID,
    NCAcquireAuthorityAction,
    NCAction,
    NCActionType,
    NCArgs,
    NCDepositAction,
    NCFail,
    NCFee,
    NCGrantAuthorityAction,
    NCParsedArgs,
    NCRawArgs,
    NCWithdrawalAction,
    SignedData,
    Timestamp,
    TokenUid,
    TxOutputScript,
    VertexId,
    export,
    fallback,
    public,
    sha3,
    verify_ecdsa,
    view,
)
```

### Import Rules Enforcement

❌ **FORBIDDEN**:
```python
import hashlib          # NO - use from x import y syntax only
import os              # NO - not in allowed list
import json            # NO - not in allowed list
import requests        # NO - not in allowed list
import datetime        # NO - not in allowed list
from datetime import datetime  # NO - not in allowed list
from hashlib import sha256     # NO - not in allowed list
from typing import Dict, List  # NO - only Optional, NamedTuple, TypeAlias, Union allowed
```

✅ **ALLOWED**:
```python
from math import ceil, floor
from typing import Optional, NamedTuple, TypeAlias, Union
from collections import OrderedDict
from hathor import Blueprint, public, view, Context, export, NCFail
```

### Why These Restrictions Exist

- **Security**: Prevents access to dangerous system functions
- **Determinism**: Ensures contracts behave identically across all nodes
- **Consensus**: Maintains blockchain state consistency
- **Sandboxing**: Limits contract capabilities to safe, audited functions

**IMPORTANT**: If user code contains forbidden imports, you MUST point out the violation and provide the corrected imports using only the allowed list above.

---

## Critical Rules

### Rule 1: All Attributes Must Be Explicitly Initialized
❌ WRONG:
```python
class MyContract(Blueprint):
    balances: dict[Address, int]
    total_supply: int

@public
def initialize(self, ctx: Context):
    # Missing initialization - attributes not set!
    pass
```

✅ CORRECT:
```python
class MyContract(Blueprint):
    balances: dict[Address, int]
    total_supply: int

@public
def initialize(self, ctx: Context):
    self.balances = {}  # Must initialize all attributes
    self.total_supply = 0
```

### Rule 2: Use initialize(), NOT __init__
❌ WRONG: `def __init__(self):`
✅ CORRECT: `def initialize(self, ctx: Context, ...):`

### Rule 3: Always Export Blueprint
❌ WRONG: Missing @export decorator
✅ CORRECT: Use `@export` decorator on your class

### Rule 4: Decorators Are Required
❌ WRONG: Method without @public or @view
✅ CORRECT:
- `@public` for state-changing methods (requires `ctx: Context`)
- `@view` for read-only methods (no `ctx` parameter)

### Rule 5: No Default Values in @public/@view Methods
❌ WRONG:
```python
@public
def initialize(self, ctx: Context, initial_value: int = 0) -> None:
    # Default values not allowed in public methods!

@view
def get_balance(self, address: Address = None) -> int:
    # Default values not allowed in view methods!
```

✅ CORRECT:
```python
@public
def initialize(self, ctx: Context, initial_value: int) -> None:
    # No default values - caller must provide all arguments

@view
def get_balance(self, address: Address) -> int:
    # No default values - all parameters are required

# Internal methods CAN have defaults:
def _internal_helper(self, value: int = 0) -> int:
    return value * 2
```

**Why**: Default values can cause ABI compatibility issues and make contract calls ambiguous.

### Rule 6: ALL Attributes Must Be Initialized
❌ WRONG:
```python
class MyContract(Blueprint):
    owner: Address
    balances: dict[Address, int]
    total_supply: int

    @public
    def initialize(self, ctx: Context, owner: Address) -> None:
        self.owner = owner
        # Missing initialization of other attributes!
```

✅ CORRECT:
```python
class MyContract(Blueprint):
    owner: Address
    balances: dict[Address, int]
    total_supply: int

    @public
    def initialize(self, ctx: Context, owner: Address) -> None:
        self.owner = owner
        self.balances = {}  # Initialize even if empty
        self.total_supply = 0  # Initialize all attributes
```

**Why**: All declared attributes must be explicitly initialized in the `initialize` method, even if they start as empty containers or zero values.

---

## Blueprint Fundamentals

### Blueprint Structure

```python
from hathor import Blueprint, public, view, Context, export

@export
class MyBlueprint(Blueprint):
    # 1. FIELD DECLARATIONS (type annotations only)
    counter: int
    owner: Address
    balances: dict[Address, int]

    # 2. INITIALIZE METHOD (required, must be @public)
    @public
    def initialize(self, ctx: Context, initial_value: int) -> None:
        self.counter = initial_value
        self.owner = ctx.get_caller_address()
        self.balances = {}  # Initialize all declared attributes!

    # 3. PUBLIC METHODS (state-changing, require ctx)
    @public
    def increment(self, ctx: Context) -> None:
        self.counter += 1

    # 4. VIEW METHODS (read-only, no ctx)
    @view
    def get_count(self) -> int:
        return self.counter
```


### Supported Field Types

#### Primitive Types
- `int`: Signed integer (arbitrary precision)
- `bool`: Boolean (True/False)
- `str`: String (UTF-8)
- `bytes`: Byte array

#### Hathor Types
- `Address`: Wallet address (25 bytes)
- `ContractId`: Contract identifier (32 bytes)
- `TokenUid`: Token unique ID (32 bytes)
- `TxOutputScript`: Script for validation
- `SignedData[T]`: Cryptographically signed data

#### Container Types (Special Hathor Implementations!)
- `dict[K, V]`: Key-value mapping (becomes **DictContainer**)
- `set[T]`: Unique values (becomes **SetContainer**)
- `list[T]`: Ordered values (becomes **DequeContainer**)
- `tuple[...]`: Immutable sequence
- `Optional[T]`: Value or None

#### Complex Types
- `tuple[A, B, C]`: Fixed-size tuple
- `NamedTuple`: Named tuple fields
- Dataclasses (with @dataclass)

---

## Container Types (Critical)

### Important: dict ≠ dict, list ≠ list, set ≠ set

**In Hathor Blueprints, when you declare `dict`, `list`, or `set` types, they are NOT standard Python containers. They are special Hathor container types that persist to blockchain storage:**

- `dict[K, V]` → **DictContainer** (not Python dict)
- `list[T]` → **DequeContainer** (not Python list)
- `set[T]` → **SetContainer** (not Python set)

### Key Differences from Standard Python

#### DictContainer vs dict
```python
# Declaration (looks like dict, but it's DictContainer)
class MyContract(Blueprint):
    balances: dict[Address, int]  # This is DictContainer!

    @public
    def initialize(self, ctx: Context) -> None:
        self.balances = {}  # Initialize as empty DictContainer

# Usage differences:
@public
def example(self, ctx: Context) -> None:
    # ✅ WORKS: Basic dict operations
    self.balances[address] = 100
    balance = self.balances.get(address, 0)
    del self.balances[address]

    # ✅ WORKS: membership
    if address in self.balances:
        pass

    # ❌ NOT IMPLEMENTED: Some dict methods
    # self.balances.keys()      # May not work
    # self.balances.values()    # May not work
    # self.balances.items()     # May not work
    # self.balances.copy()      # Not implemented

    # ❌ NOT IMPLEMENTED: Iteration does not work!!
    # for address in self.balances: # <-- WILL NOT WORK
    #    pass
```

#### DequeContainer vs list
```python
# Declaration (looks like list, but it's DequeContainer)
class MyContract(Blueprint):
    items: list[str]  # This is DequeContainer!

    @public
    def initialize(self, ctx: Context) -> None:
        self.items = []  # Initialize as empty DequeContainer

# Usage differences:
@public
def example(self, ctx: Context) -> None:
    # ✅ WORKS: Basic sequence operations
    self.items.append("item")
    self.items.extend(["a", "b", "c"])
    item = self.items[0]        # Access by index
    self.items[0] = "new_item"  # Set by index
    length = len(self.items)

    # ✅ WORKS: Deque-specific operations
    self.items.appendleft("first")
    self.items.extendleft(["x", "y"])
    first = self.items.popleft()
    last = self.items.pop()
    self.items.reverse()

    # ✅ WORKS: Iteration
    for item in self.items:
        pass

    # ❌ NOT IMPLEMENTED: Some list methods
    # self.items.insert(1, "item")  # Not implemented
    # self.items.remove("item")     # Not implemented
    # self.items.sort()             # Not implemented
    # self.items.index("item")      # Not implemented
```

#### SetContainer vs set
```python
# Declaration (looks like set, but it's SetContainer)
class MyContract(Blueprint):
    members: set[Address]  # This is SetContainer!
    members2: set[Address]

    @public
    def initialize(self, ctx: Context) -> None:
        self.members = set()  # Initialize as empty SetContainer

# Usage differences:
@public
def example(self, ctx: Context) -> None:
    # ✅ WORKS: Basic set operations
    self.members.add(address)
    self.members.discard(address)
    self.members.remove(address)  # Raises KeyError if not found
    self.members.update([addr1, addr2])

    # ✅ WORKS: Membership and size
    if address in self.members:
        pass
    size = len(self.members)

    # ✅ WORKS: Some set operations
    other_set: set = set()
    is_disjoint = self.members.isdisjoint(other_set)
    is_superset = self.members.issuperset(other_set)
    intersection = self.members.intersection(other_set)

    # ❌ NOT IMPLEMENTED: set operations with other containers
    is_disjoint = self.members.isdisjoint(self.members2)
    is_superset = self.members.issuperset(self.members2)
    intersection = self.members.intersection(self.members2)

    # ❌ NOT IMPLEMENTED: Many set methods
    # self.members.copy()                    # Not implemented
    # self.members.union(other)              # Not implemented
    # self.members.difference(other)         # Not implemented
    # self.members.symmetric_difference()    # Not implemented
```

### Why These Special Containers Exist

1. **Blockchain Storage**: Standard Python containers can't persist to blockchain storage
2. **Merkle Patricia Trie**: Data is stored in a trie structure for efficient blockchain operations

### Container Initialization Patterns

```python
class TokenContract(Blueprint):
    balances: dict[Address, int]           # DictContainer
    transaction_history: list[str]         # DequeContainer
    authorized_minters: set[Address]       # SetContainer

    @public
    def initialize(self, ctx: Context, owner: Address) -> None:
        # ✅ CORRECT: Initialize all containers
        self.balances = {}                 # Empty DictContainer
        self.transaction_history = []      # Empty DequeContainer
        self.authorized_minters = set()    # Empty SetContainer

        # ✅ ALSO CORRECT: initialize containers with values
        # self.balances = { owner: 1000 }
        # self.transaction_history = [f"Contract created by {owner.hex()}"]
        # self.authorized_minters = {owner}
```
---

## Container Operations Reference

### DictContainer Operations

```python
@export
class ExampleContract(Blueprint):
    balances: dict[Address, int]  # This is DictContainer!

    @public
    def dict_operations(self, ctx: Context) -> None:
        address = ctx.get_caller_address()

        # ✅ THESE WORK - Basic dict operations
        self.balances[address] = 100              # __setitem__
        balance = self.balances[address]          # __getitem__
        balance = self.balances.get(address, 0)   # get() with default
        exists = address in self.balances         # __contains__
        del self.balances[address]                # __delitem__
        length = len(self.balances)               # __len__
        self.balances.update({addr1: 50, addr2: 75})  # update()

        # ❌ THESE FAIL - Will raise NotImplementedError or AttributeError
        keys = self.balances.keys()               # NOT IMPLEMENTED!
        values = self.balances.values()           # NOT IMPLEMENTED!
        items = self.balances.items()             # NOT IMPLEMENTED!
        copy_dict = self.balances.copy()          # NOT IMPLEMENTED!
        popped = self.balances.pop(address)       # NOT IMPLEMENTED!
        item = self.balances.popitem()            # NOT IMPLEMENTED!
        self.balances.clear()                     # NOT IMPLEMENTED!
        default_val = self.balances.setdefault(address, 0)  # NOT IMPLEMENTED!

        # ❌ THESE FAIL - Trying to convert to Python dict
        python_dict = dict(self.balances)         # WILL FAIL!
        key_list = list(self.balances.keys())     # WILL FAIL!

        # ✅ WORKAROUNDS - Use supported operations instead
        # Instead of .keys(), use membership testing:
        for potential_key in known_addresses:
            if potential_key in self.balances:
                value = self.balances[potential_key]
```

### DequeContainer Operations

```python
@export
class ExampleContract(Blueprint):
    history: list[str]  # This is DequeContainer!

    @public
    def list_operations(self, ctx: Context) -> None:
        # ✅ THESE WORK - Basic sequence operations
        self.history.append("event")              # append()
        self.history.extend(["a", "b", "c"])      # extend()
        item = self.history[0]                    # __getitem__
        self.history[0] = "new_item"              # __setitem__
        length = len(self.history)                # __len__

        # ✅ THESE WORK - Deque-specific operations
        self.history.appendleft("first")          # appendleft()
        self.history.extendleft(["x", "y"])       # extendleft()
        first = self.history.popleft()            # popleft()
        last = self.history.pop()                 # pop()
        self.history.reverse()                    # reverse()

        # ✅ THESE WORK - Iteration
        for item in self.history:                 # __iter__
            pass

        # ❌ THESE FAIL - Will raise NotImplementedError
        self.history.insert(1, "item")            # NOT IMPLEMENTED!
        self.history.remove("item")               # NOT IMPLEMENTED!
        self.history.sort()                       # NOT IMPLEMENTED!
        index = self.history.index("item")        # NOT IMPLEMENTED!
        count = self.history.count("item")        # NOT IMPLEMENTED!
        copy_list = self.history.copy()           # NOT IMPLEMENTED!

        # ❌ THESE FAIL - Standard list methods
        self.history.clear()                      # NOT IMPLEMENTED!

        # ❌ THESE FAIL - Trying to convert to Python list
        python_list = list(self.history)          # MAY FAIL!

        # ✅ WORKAROUNDS - Use supported operations instead
        # Instead of .index(), iterate manually:
        for i, item in enumerate(self.history):
            if item == "target":
                found_index = i
                break
```

### SetContainer Operations

```python
@export
class ExampleContract(Blueprint):
    members: set[Address]  # This is SetContainer!

    @public
    def set_operations(self, ctx: Context) -> None:
        address = ctx.get_caller_address()
        other_addresses = [addr1, addr2, addr3]

        # ✅ THESE WORK - Basic set operations
        self.members.add(address)                 # add()
        self.members.discard(address)             # discard() (no error if missing)
        self.members.remove(address)              # remove() (raises KeyError if missing)
        self.members.update(other_addresses)      # update()
        exists = address in self.members          # __contains__
        length = len(self.members)                # __len__

        # ✅ THESE WORK - Some set operations
        is_empty = self.members.isdisjoint(other_addresses)  # isdisjoint()
        is_superset = self.members.issuperset(other_addresses)  # issuperset()
        intersection = self.members.intersection(other_addresses)  # intersection()
        self.members.difference_update(other_addresses)  # difference_update()

        # ❌ THESE FAIL - Will raise NotImplementedError
        copy_set = self.members.copy()            # NOT IMPLEMENTED!
        union_set = self.members.union(other_addresses)  # NOT IMPLEMENTED!
        diff_set = self.members.difference(other_addresses)  # NOT IMPLEMENTED!
        sym_diff = self.members.symmetric_difference(other_addresses)  # NOT IMPLEMENTED!
        subset = self.members.issubset(other_addresses)  # NOT IMPLEMENTED!
        popped = self.members.pop()               # NOT IMPLEMENTED!
        self.members.clear()                      # NOT IMPLEMENTED!
        self.members.symmetric_difference_update(other_addresses)  # NOT IMPLEMENTED!
        self.members.intersection_update(other_addresses)  # NOT IMPLEMENTED!

        # ❌ THESE FAIL - Set operators
        union_op = self.members | other_set       # NOT IMPLEMENTED!
        intersect_op = self.members & other_set   # NOT IMPLEMENTED!
        diff_op = self.members - other_set        # NOT IMPLEMENTED!

        # ❌ THESE FAIL - Trying to convert to Python set
        python_set = set(self.members)            # MAY FAIL!

        # ✅ WORKAROUNDS - Use supported operations instead
        # Instead of .union(), add items manually:
        for item in other_addresses:
            self.members.add(item)
```

### Common Failure Patterns

```python
@export
class BadExample(Blueprint):
    data: dict[str, int]
    items: list[str]
    tags: set[str]

    @public
    def bad_operations(self, ctx: Context) -> None:
        # ❌ ALL OF THESE WILL FAIL!

        # Dict failures
        for key in self.data.keys():              # FAIL: .keys() not implemented
            pass
        all_values = list(self.data.values())     # FAIL: .values() not implemented
        dict_copy = self.data.copy()              # FAIL: .copy() not implemented

        # List failures
        self.items.sort()                         # FAIL: .sort() not implemented
        self.items.insert(0, "first")             # FAIL: .insert() not implemented
        self.items.remove("item")                 # FAIL: .remove() not implemented

        # Set failures
        new_set = self.tags.copy()                # FAIL: .copy() not implemented
        union_result = self.tags.union({"a"})     # FAIL: .union() not implemented

        # Conversion failures
        py_dict = dict(self.data)                 # FAIL: Can't convert to Python dict
        py_list = list(self.items)                # FAIL: May not work
        py_set = set(self.tags)                   # FAIL: May not work

@export
class GoodExample(Blueprint):
    data: dict[str, int]
    items: list[str]
    tags: set[str]

    @public
    def good_operations(self, ctx: Context) -> None:
        # ✅ ALL OF THESE WORK!

        # Dict operations
        self.data["key"] = 100
        value = self.data.get("key", 0)
        if "key" in self.data:
            del self.data["key"]

        # List operations
        self.items.append("new")
        self.items.appendleft("first")
        last = self.items.pop()
        first = self.items.popleft()

        # Set operations
        self.tags.add("new_tag")
        self.tags.discard("old_tag")
        if "tag" in self.tags:
            self.tags.remove("tag")
```


### Decorators Explained

Hathor provides three method decorators with fine-grained permission control:

#### @public - State-Changing Methods

The `@public` decorator marks methods that can modify contract state. These methods:
- **MUST** have `ctx: Context` as the first parameter (after `self`)
- Can read and write state
- Can interact with other contracts
- Can perform token operations (if allowed)

**Available Parameters**:
```python
@public(
    # Token Actions (individual flags)
    allow_deposit=True,              # Allow HTR/token deposits
    allow_withdrawal=True,           # Allow HTR/token withdrawals
    allow_grant_authority=True,      # Allow granting mint/melt authorities
    allow_acquire_authority=True,    # Allow acquiring mint/melt authorities

    # OR use allow_actions for multiple types
    allow_actions=[NCActionType.DEPOSIT, NCActionType.WITHDRAWAL],

    # Reentrancy Control
    allow_reentrancy=False           # Prevent recursive calls (default: False)
)
def my_method(self, ctx: Context, arg1: str) -> int:
    # Can modify state
    self.counter += 1

    # Can access context
    caller = ctx.get_caller_address()

    # Can access actions
    for action in ctx.actions_list:
        if isinstance(action, NCDepositAction):
            # Handle deposit
            pass

    return self.counter
```

**Note**: Use either individual flags (`allow_deposit`, `allow_withdrawal`, etc.) OR `allow_actions` list, not both.

#### @view - Read-Only Methods

The `@view` decorator marks methods that can only read state, never modify it:
- **NO** `ctx` parameter (view methods don't have access to context)
- Can only read state
- Cannot call other contracts
- Cannot modify any fields
- Will raise `NCViewMethodError` if you try to write

```python
@view
def get_balance(self, address: Address) -> int:
    # NO ctx parameter!
    # Can only read state, never modify
    return self.balances.get(address, 0)

@view
def get_total_supply(self) -> int:
    return sum(self.balances.values())
```

#### @fallback - Catch-All Method

The `@fallback` decorator creates a catch-all method for undefined method calls. This is an advanced pattern:
- Method **MUST** be named `fallback`
- Has same permission parameters as `@public`
- Special signature: `def fallback(self, ctx: Context, method_name: str, nc_args: NCArgs)`

```python
from hathor import NCArgs

@fallback(allow_deposit=True)
def fallback(self, ctx: Context, method_name: str, nc_args: NCArgs) -> None:
    # Called when undefined method is invoked
    # method_name: the name that was called
    # nc_args: the arguments passed

    if method_name.startswith("custom_"):
        # Handle custom methods dynamically
        pass
    else:
        raise NCFail(f"Unknown method: {method_name}")
```

#### Action Types Reference

```python
from hathor import NCActionType

# Available action types:
NCActionType.DEPOSIT           # Token deposits into contract
NCActionType.WITHDRAWAL        # Token withdrawals from contract
NCActionType.GRANT_AUTHORITY   # Grant mint/melt authority to another contract
NCActionType.ACQUIRE_AUTHORITY # Acquire mint/melt authority from transaction
```

### Context Object (ctx)

The `Context` object is passed to all `@public` and `@fallback` methods. It provides **immutable** access to transaction and block data.

#### Properties

```python
@public
def example(self, ctx: Context) -> None:
    # ========== CALLER INFORMATION ==========

    # Caller ID (can be Address or ContractId)
    caller_id = ctx.caller_id  # Address | ContractId

    # Type-safe caller access
    caller_addr = ctx.get_caller_address()  # Address or None
    caller_contract = ctx.get_caller_contract_id()  # ContractId or None

    # Example usage:
    if caller_addr := ctx.get_caller_address():
        # Called by a wallet address
        print(f"User: {caller_addr.hex()}")
    elif caller_contract := ctx.get_caller_contract_id():
        # Called by another contract
        print(f"Contract: {caller_contract.hex()}")

    # ========== ACTIONS (Deposits, Withdrawals, Authorities) ==========

    # All actions as a list
    all_actions = ctx.actions_list  # Sequence[NCAction]

    # Actions grouped by token UID
    actions_by_token = ctx.actions  # MappingProxyType[TokenUid, tuple[NCAction, ...]]

    # Get exactly one action for a token (raises NCFail if != 1)
    single_action = ctx.get_single_action(token_uid)

    # Iterate over all actions
    for action in ctx.actions_list:
        if isinstance(action, NCDepositAction):
            print(f"Deposit: {action.amount} of {action.token_uid.hex()}")
        elif isinstance(action, NCWithdrawalAction):
            print(f"Withdrawal: {action.amount} of {action.token_uid.hex()}")
        elif isinstance(action, NCGrantAuthorityAction):
            print(f"Grant authority: mint={action.mint}, melt={action.melt}")
        elif isinstance(action, NCAcquireAuthorityAction):
            print(f"Acquire authority: mint={action.mint}, melt={action.melt}")

    # ========== VERTEX (Transaction) DATA ==========

    # Transaction hash (unique identifier)
    tx_hash = ctx.vertex.hash  # bytes (32 bytes)

    # Transaction timestamp (NOT block timestamp!)
    tx_timestamp = ctx.vertex.timestamp  # int (seconds since epoch)

    # Transaction weight
    tx_weight = ctx.vertex.weight  # float

    # Transaction nonce
    tx_nonce = ctx.vertex.nonce  # int

    # Transaction version
    tx_version = ctx.vertex.version  # TxVersion

    # Transaction inputs (where funds came from)
    for tx_input in ctx.vertex.inputs:
        prev_tx_id = tx_input.tx_id  # VertexId
        output_index = tx_input.index  # int
        signature_data = tx_input.data  # bytes
        # Output info (if available)
        if tx_input.info:
            value = tx_input.info.value  # int
            script = tx_input.info.raw_script  # bytes
            token_data = tx_input.info.token_data  # int

    # Transaction outputs (where funds go)
    for tx_output in ctx.vertex.outputs:
        value = tx_output.value  # int (amount)
        script = tx_output.raw_script  # bytes
        token_data = tx_output.token_data  # int
        # Parsed script info (if available)
        if tx_output.parsed_script:
            script_type = tx_output.parsed_script.type  # "P2PKH" or "MultiSig"
            address = tx_output.parsed_script.address  # str (base58)
            timelock = tx_output.parsed_script.timelock  # int | None

    # Parent transactions
    parent_txs = ctx.vertex.parents  # tuple[VertexId, ...]

    # Custom tokens involved
    tokens = ctx.vertex.tokens  # tuple[TokenUid, ...]

    # ========== BLOCK DATA ==========

    # Block timestamp (when transaction was confirmed)
    block_timestamp = ctx.block.timestamp  # int (seconds since epoch)

    # Block height (blockchain position)
    block_height = ctx.block.height  # int

    # Block hash
    block_hash = ctx.block.hash  # VertexId (bytes)

    # ========== DEPRECATED ==========

    # Old property (use ctx.block.timestamp instead)
    timestamp = ctx.timestamp  # Same as ctx.block.timestamp
```

#### Common Patterns

```python
# Pattern 1: Require specific caller
@public
def only_owner(self, ctx: Context) -> None:
    caller = ctx.get_caller_address()
    if caller != self.owner:
        raise Unauthorized("Only owner can call this")

# Pattern 2: Check time constraints
@public
def before_deadline(self, ctx: Context) -> None:
    if ctx.block.timestamp > self.deadline:
        raise TooLate(f"Deadline was {self.deadline}")

# Pattern 3: Handle deposits
@public(allow_deposit=True)
def deposit(self, ctx: Context) -> None:
    action = ctx.get_single_action(self.token_uid)
    assert isinstance(action, NCDepositAction)

    caller = ctx.get_caller_address()
    self.balances[caller] = self.balances.get(caller, 0) + action.amount

# Pattern 4: Multi-token handling
@public(allow_deposit=True)
def multi_deposit(self, ctx: Context) -> None:
    caller = ctx.get_caller_address()

    for action in ctx.actions_list:
        if isinstance(action, NCDepositAction):
            token_uid = action.token_uid
            amount = action.amount
            # Update balance per token
            key = (caller, token_uid)
            self.balances[key] = self.balances.get(key, 0) + amount
```

#### Important Notes

- **Context is immutable**: You cannot modify `ctx` or any of its properties
- **Block data requires confirmation**: `ctx.block` is only available after the transaction is confirmed in a block
- **Actions are validated**: The system ensures actions match method permissions before calling your method
- **Caller identification**: Use `get_caller_address()` / `get_caller_contract_id()` for type-safe access instead of checking `caller_id` directly

### Actions (Deposits & Withdrawals)

```python
from hathor import (
    NCDepositAction,
    NCWithdrawalAction,
    NCGrantAuthorityAction,
    NCAcquireAuthorityAction
)

@public(allow_deposit=True)
def deposit_tokens(self, ctx: Context) -> None:
    # Access deposited tokens
    for action in ctx.actions_list:
        if isinstance(action, NCDepositAction):
            token_uid = action.token_uid
            amount = action.amount
            # Token is now in contract balance

@public(allow_withdrawal=True)
def withdraw_tokens(self, ctx: Context, amount: int) -> None:
    # Withdrawal is declared in the transaction
    # You just need to validate it's allowed
    action = ctx.get_single_action(self.token_uid)
    if action.amount > self.balances[ctx.get_caller_address()]:
        raise NCFail("Insufficient balance")
    # Withdrawal happens automatically
```

### Storage and State Management

State is automatically persisted in a Merkle Patricia Trie:

```python
class TokenContract(Blueprint):
    balances: dict[Address, int]  # Stored in trie
    total_supply: int              # Stored in trie

    @public
    def transfer(self, ctx: Context, to: Address, amount: int) -> None:
        from_addr = ctx.get_caller_address()

        # Read from storage
        from_balance = self.balances.get(from_addr, 0)

        # Validate
        if from_balance < amount:
            raise InsufficientBalance()

        # Update storage (automatically persisted)
        self.balances[from_addr] = from_balance - amount
        self.balances[to] = self.balances.get(to, 0) + amount
```

### Error Handling

```python
from hathor import NCFail

# Define custom errors
class InsufficientBalance(NCFail):
    pass

class Unauthorized(NCFail):
    pass

class InvalidAmount(NCFail):
    pass

# Use in methods
@public
def withdraw(self, ctx: Context, amount: int) -> None:
    if amount <= 0:
        raise InvalidAmount("Amount must be positive")

    balance = self.balances.get(ctx.get_caller_address(), 0)
    if balance < amount:
        raise InsufficientBalance(f"Balance: {balance}, requested: {amount}")
```

## Advanced Features (self.syscall)

The `self.syscall` object provides access to powerful system operations. Use these for advanced contract patterns:

### Contract Interactions

```python
# Call another contract's public method
result = self.syscall.call_public_method(
    nc_id=other_contract_id,
    method_name="transfer",
    actions=[],  # Optional actions to pass
    from_addr=ctx.get_caller_address(),
    to_addr=recipient,
    amount=100
)

# Call another contract's view method (read-only)
balance = self.syscall.call_view_method(
    nc_id=other_contract_id,
    method_name="get_balance",
    address=some_address
)

# Proxy call (delegatecall pattern - runs blueprint code with current contract's state)
result = self.syscall.proxy_call_public_method(
    blueprint_id=other_blueprint_id,
    method_name="method_name",
    actions=[],
    arg1, arg2
)
```

### Token Creation & Management

```python
# Create a new token
token_uid = self.syscall.create_token(
    token_name="MyToken",
    token_symbol="MTK",
    amount=1000,              # Initial supply (goes to contract)
    mint_authority=True,      # Contract can mint more
    melt_authority=True       # Contract can melt (burn)
)

# Mint tokens (requires mint authority)
self.syscall.mint_tokens(
    token_uid=token_uid,
    amount=500
)

# Melt tokens (requires melt authority)
self.syscall.melt_tokens(
    token_uid=token_uid,
    amount=200
)

# Revoke authorities (cannot be undone!)
self.syscall.revoke_authorities(
    token_uid=token_uid,
    revoke_mint=True,
    revoke_melt=False
)
```

### Balance & Authority Queries

```python
# Get current contract ID
contract_id = self.syscall.get_contract_id()

# Get blueprint ID
blueprint_id = self.syscall.get_blueprint_id()
other_blueprint = self.syscall.get_blueprint_id(contract_id=other_contract_id)

# Get balance (includes current call's actions)
balance = self.syscall.get_current_balance(
    token_uid=token_uid,
    contract_id=None  # None = current contract
)

# Get balance before current call (excludes current actions)
balance_before = self.syscall.get_balance_before_current_call(
    token_uid=token_uid
)

# Check mint/melt authority
can_mint = self.syscall.can_mint(token_uid)
can_melt = self.syscall.can_melt(token_uid)

# Check authority before current call
can_mint_before = self.syscall.can_mint_before_current_call(token_uid)
can_melt_before = self.syscall.can_melt_before_current_call(token_uid)
```

### Contract Creation

```python
# Create a child contract
new_contract_id, init_result = self.syscall.create_contract(
    blueprint_id=child_blueprint_id,
    salt=b"unique_salt",  # For deterministic addresses
    actions=[],           # Optional initial actions
    initial_value=100     # Constructor arguments
)
```

### Events & Upgrades

```python
# Emit custom event
self.syscall.emit_event(b"Transfer successful")

# Upgrade contract blueprint (advanced!)
self.syscall.change_blueprint(new_blueprint_id)
```

### Random Number Generation (self.syscall.rng)

**Unique Hathor Feature**: Hathor provides a **built-in deterministic RNG syscall** using ChaCha20 encryption. This is accessed via `self.syscall.rng` - a blockchain-native randomness source!

#### How It Works

1. **Transaction Seed**: Each transaction has a unique seed
2. **Per-Contract Derivation**: Each contract gets its own RNG derived from the transaction seed
3. **Deterministic**: Same transaction → same random sequence (consensus guaranteed)
4. **ChaCha20**: Uses cryptographically strong ChaCha20 stream cipher

```python
# Access the RNG (syscall - not a library import!)
rng = self.syscall.rng  # Returns: NanoRNG instance unique to this contract

# ========== BASIC METHODS ==========

# Random bytes
random_bytes = rng.randbytes(32)  # Returns: bytes of specified size
random_hash = rng.randbytes(32)   # Perfect for generating IDs

# Random integer in range [a, b] (inclusive on both ends!)
dice_roll = rng.randint(1, 6)     # Returns: 1, 2, 3, 4, 5, or 6
percent = rng.randint(0, 100)     # Returns: 0 to 100

# Random float in range [0, 1)
probability = rng.random()         # Returns: 0.0 <= value < 1.0

# ========== ADVANCED METHODS ==========

# Random integer with specific bit length
big_number = rng.randbits(256)    # Returns: 0 <= n < 2**256

# Random integer in range [0, n) (exclusive upper bound!)
index = rng.randbelow(10)         # Returns: 0 to 9

# Random integer in range [start, stop) with step
even_number = rng.randrange(0, 100, 2)  # Returns: 0, 2, 4, ..., 98

# Choose random element from sequence
winner = rng.choice(["Alice", "Bob", "Charlie"])
card = rng.choice([1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13])

# ========== SEED ACCESS ==========

# Get the seed used (read-only)
seed = rng.seed  # bytes (32 bytes)
```

#### Real-World Example: Dice Game (from Hathor Labs)

```python
from hathor import (
    Blueprint, public, view, Context, export,
    NCFail, Amount, CallerId
)

@export
class HathorDice(Blueprint):
    """Production dice game using self.syscall.rng for on-chain randomness."""

    token_uid: TokenUid
    max_bet_amount: Amount
    house_edge_basis_points: int  # 50 = 0.50%
    random_bit_length: int  # e.g., 32 for 2^32 range

    balances: dict[CallerId, int]
    available_tokens: Amount

    @public
    def initialize(self, ctx: Context, token_uid: TokenUid,
                   house_edge_basis_points: int, max_bet_amount: Amount,
                   random_bit_length: int) -> None:
        if random_bit_length < 16 or random_bit_length > 32:
            raise NCFail('random bit length must be 16-32')

        # Initialize all declared attributes
        self.balances = {}
        self.token_uid = token_uid
        self.house_edge_basis_points = house_edge_basis_points
        self.max_bet_amount = max_bet_amount
        self.random_bit_length = random_bit_length
        self.available_tokens = 0

    @public(allow_deposit=True)
    def place_bet(self, ctx: Context, bet_amount: Amount, threshold: int) -> int:
        """
        Player bets that lucky_number < threshold.

        Args:
            bet_amount: Amount to wager
            threshold: Win if random number is below this (0 to 2^random_bit_length)

        Returns:
            Payout amount (0 if lost)
        """
        if bet_amount > self.max_bet_amount:
            raise NCFail('bet amount too high')

        # Generate random number using syscall RNG
        lucky_number = self.syscall.rng.randbits(self.random_bit_length)

        if lucky_number >= threshold:
            # Lose - house keeps the bet
            self.available_tokens += bet_amount
            self.syscall.emit_event(f'{{"result": "lose", "number": {lucky_number}}}'.encode())
            return 0

        # Win - calculate payout with house edge
        payout = self.calculate_payout(bet_amount, threshold)

        if payout > self.available_tokens:
            raise NCFail('not enough liquidity')

        self.available_tokens -= (payout - bet_amount)
        self.balances[ctx.caller_id] = self.balances.get(ctx.caller_id, 0) + payout

        self.syscall.emit_event(f'{{"result": "win", "payout": {payout}}}'.encode())
        return payout

    @view
    def calculate_payout(self, bet_amount: Amount, threshold: int) -> int:
        """Calculate payout with house edge applied."""
        # fair_odds = 2^bits / threshold
        # adjusted_odds = fair_odds * (1 - house_edge)
        numerator = bet_amount * (2**self.random_bit_length) * (10_000 - self.house_edge_basis_points)
        denominator = 10_000 * threshold
        return numerator // denominator

```

**Key Points from This Example**:
- Uses `self.syscall.rng.randbits()` for on-chain randomness
- No external oracle needed
- House edge applied to payouts
- Emits events for game results
- Production-ready pattern from Hathor Labs

#### Important Notes

**Why This is Special**:
- **Built into the blockchain**: Not a library - it's a syscall!
- **Per-contract isolation**: Each contract gets its own RNG instance
- **Consensus-safe**: All nodes produce identical random numbers
- **ChaCha20-based**: Cryptographically strong stream cipher
- **No external oracles needed**: Randomness is on-chain!

**Determinism**:
- The RNG is **deterministic**: Same transaction seed → same random sequence
- Each contract's RNG is **derived** from the transaction seed using the contract ID
- All nodes will generate **identical** random numbers
- This ensures **consensus** across the network
- Formula: `contract_rng = NanoRNG(seed=transaction_rng.randbytes(32))`

**Security Considerations**:
- ✅ **Good for**: Game logic, lotteries, random selection, shuffling, dice rolls
- ❌ **NOT for**: Cryptographic keys, signatures, secret generation
- ⚠️ **Predictability**: Transaction creator can influence the seed by choosing transaction inputs
- **On-chain randomness**: No need for external oracles (like Chainlink VRF)

**Best Practices**:
```python
# ✅ GOOD: Use for game mechanics
winner = rng.choice(players)
damage = rng.randint(10, 20)

# ✅ GOOD: Use for fair random selection (if seed cannot be manipulated)
lottery_winner = rng.randbelow(len(participants))

# ❌ BAD: Don't use for security-critical operations
private_key = rng.randbytes(32)  # INSECURE!

# ⚠️ CAREFUL: User can influence seed by choosing inputs
# Consider using commit-reveal schemes for high-stakes randomness
```

**Commit-Reveal Pattern** (for unbiasable randomness):
```python
from hathor import Blueprint, public, Context, NCFail, Address, sha3

class UnbiasableLottery(Blueprint):
    commitments: dict[Address, bytes]  # Hash of secret
    reveals: dict[Address, bytes]      # Revealed secret

    @public
    def commit(self, ctx: Context, commitment: bytes) -> None:
        # Step 1: Users commit hash of their secret
        caller = ctx.get_caller_address()
        self.commitments[caller] = commitment

    @public
    def reveal(self, ctx: Context, secret: bytes) -> None:
        # Step 2: Users reveal their secret
        caller = ctx.get_caller_address()
        # Note: In real Hathor contracts, use self.syscall.sha3() for hashing
        if sha3(secret) != self.commitments[caller]:
            raise NCFail("Invalid reveal")
        self.reveals[caller] = secret

    @public
    def draw(self, ctx: Context) -> None:
        # Step 3: Combine all secrets to create unbiased seed
        combined = b"".join(self.reveals.values())
        # Note: In real Hathor contracts, use sha3() for hashing
        seed = sha3(combined)

        # Use combined seed for RNG (more complex implementation needed)
        # This prevents any single party from biasing the result
```

---

## Testing Blueprints

### Test File Structure

```python
from hathor import (
    Blueprint, public, view, Context, export,
    Address, TokenUid, NCDepositAction, NCWithdrawalAction
)
from hathor import make_nc_type_for_arg_type as make_nc_type
from tests.nanocontracts.blueprints.unittest import BlueprintTestCase

# Type helpers for storage assertions
INT_NC_TYPE = make_nc_type(int)
STR_NC_TYPE = make_nc_type(str)

class MyBlueprintTest(BlueprintTestCase):
    def setUp(self) -> None:
        super().setUp()
        # Register blueprint
        self.blueprint_id = self._register_blueprint_class(MyBlueprint)
        # Generate test data
        self.token_uid = self.gen_random_token_uid()
        self.address = self.gen_random_address()

    def test_initialize(self) -> None:
        nc_id = self.gen_random_contract_id()
        ctx = self.create_context()

        # Create contract
        self.runner.create_contract(nc_id, self.blueprint_id, ctx, initial_value)

        # Verify state
        storage = self.runner.get_storage(nc_id)
        self.assertEqual(storage.get_obj(b'field_name', INT_NC_TYPE), expected_value)

    def test_public_method(self) -> None:
        # ... setup contract ...

        # Call public method
        ctx = self.create_context()
        self.runner.call_public_method(nc_id, 'method_name', ctx, arg1, arg2)

        # Verify state changed
        storage = self.runner.get_storage(nc_id)
        self.assertEqual(storage.get_obj(b'counter', INT_NC_TYPE), 1)

    def test_view_method(self) -> None:
        # View methods don't need context
        result = self.runner.call_view_method(nc_id, 'get_value')
        self.assertEqual(result, expected_value)

    def test_deposit(self) -> None:
        actions = [NCDepositAction(token_uid=self.token_uid, amount=100)]
        ctx = self.create_context(actions)
        self.runner.call_public_method(nc_id, 'deposit', ctx)

        # Verify balance
        storage = self.runner.get_storage(nc_id)
        balance = storage.get_balance(self.token_uid)
        self.assertEqual(balance.value, 100)

    def test_error_handling(self) -> None:
        with self.assertRaises(CustomError):
            self.runner.call_public_method(nc_id, 'failing_method', ctx)

        # Verify state was NOT changed (rollback)
        storage = self.runner.get_storage(nc_id)
        self.assertEqual(storage.get_obj(b'counter', INT_NC_TYPE), original_value)
```

### Testing Best Practices

1. **Test initialization**: Verify initial state is correct
2. **Test state changes**: Confirm @public methods update storage
3. **Test view methods**: Ensure read-only methods work
4. **Test deposits/withdrawals**: Verify token operations
5. **Test error cases**: Use assertRaises for NCFail errors
6. **Test edge cases**: Zero amounts, empty dicts, None values
7. **Verify rollback**: Failed txs don't change state

---

## Complete Examples

### Example 1: Simple Counter Blueprint

```python
from hathor import Blueprint, public, view, Context, export

@export
class Counter(Blueprint):
    count: int

    @public
    def initialize(self, ctx: Context, initial: int) -> None:
        self.count = initial

    @public
    def increment(self, ctx: Context) -> None:
        self.count += 1

    @public
    def decrement(self, ctx: Context) -> None:
        self.count -= 1

    @view
    def get_count(self) -> int:
        return self.count
```

### Example 1b: Counter with Container Usage

```python
from hathor import Blueprint, public, view, Context, export, Address
from typing import Optional

@export
class CounterWithHistory(Blueprint):
    count: int
    # DequeContainer - behaves like deque, not list
    history: list[str]
    # DictContainer - specialized blockchain dict
    user_counts: dict[Address, int]
    # SetContainer - specialized blockchain set
    authorized_users: set[Address]

    @public
    def initialize(self, ctx: Context, initial: int) -> None:
        self.count = initial
        # Initialize all containers (required!)
        self.history = []                    # Empty DequeContainer
        self.user_counts = {}               # Empty DictContainer
        self.authorized_users = set()       # Empty SetContainer

    @public
    def increment(self, ctx: Context) -> None:
        caller = ctx.get_caller_address()
        if caller not in self.authorized_users:
            self.authorized_users.add(caller)  # SetContainer.add()

        self.count += 1

        # DequeContainer operations (deque-like, not list-like)
        self.history.append(f"Count incremented to {self.count}")

        # DictContainer operations
        self.user_counts[caller] = self.user_counts.get(caller, 0) + 1

    @view
    def get_user_increment_count(self, user: Address) -> int:
        # DictContainer.get() works like dict.get()
        return self.user_counts.get(user, 0)

    @view
    def is_authorized(self, user: Address) -> bool:
        # SetContainer membership testing
        return user in self.authorized_users
```

**Test File** (`/tests/test_counter.py`):
```python
from hathor import make_nc_type_for_arg_type as make_nc_type
from tests.nanocontracts.blueprints.unittest import BlueprintTestCase

INT_NC_TYPE = make_nc_type(int)

class CounterTest(BlueprintTestCase):
    def setUp(self) -> None:
        super().setUp()
        from Counter import Counter
        self.blueprint_id = self._register_blueprint_class(Counter)

    def test_counter(self) -> None:
        nc_id = self.gen_random_contract_id()
        ctx = self.create_context()

        # Initialize with 0
        self.runner.create_contract(nc_id, self.blueprint_id, ctx, 0)
        storage = self.runner.get_storage(nc_id)
        self.assertEqual(storage.get_obj(b'count', INT_NC_TYPE), 0)

        # Increment
        self.runner.call_public_method(nc_id, 'increment', ctx)
        self.assertEqual(storage.get_obj(b'count', INT_NC_TYPE), 1)

        # View method
        result = self.runner.call_view_method(nc_id, 'get_count')
        self.assertEqual(result, 1)
```

### Example 2: Token Betting Contract

```python
from typing import Optional
from hathor import (
    Blueprint, public, view, Context, export,
    Address, TokenUid, NCDepositAction, NCWithdrawalAction,
    SignedData, TxOutputScript, NCFail
)

class InvalidToken(NCFail):
    pass

class TooLate(NCFail):
    pass

class ResultNotAvailable(NCFail):
    pass

@export
class Bet(Blueprint):
    # Total bets per result (DictContainer, not Python dict)
    bets_total: dict[str, int]

    # Bets per (result, address) (DictContainer with tuple keys)
    bets_address: dict[tuple[str, Address], int]

    # Withdrawals per address (DictContainer)
    withdrawals: dict[Address, int]

    # Total pool
    total: int

    # Final result (None until set)
    final_result: Optional[str]

    # Oracle script to verify result
    oracle_script: TxOutputScript

    # Deadline for bets
    date_last_bet: int

    # Token for betting
    token_uid: TokenUid

    @public
    def initialize(
        self,
        ctx: Context,
        oracle_script: TxOutputScript,
        token_uid: TokenUid,
        date_last_bet: int
    ) -> None:
        # Initialize all declared attributes
        self.bets_total = {}
        self.bets_address = {}
        self.withdrawals = {}
        self.oracle_script = oracle_script
        self.token_uid = token_uid
        self.date_last_bet = date_last_bet
        self.final_result = None
        self.total = 0

    @public(allow_deposit=True)
    def bet(self, ctx: Context, address: Address, score: str) -> None:
        # Get the deposit action
        if self.token_uid not in ctx.actions:
            raise InvalidToken(f"Expected token {self.token_uid.hex()}")

        action = ctx.get_single_action(self.token_uid)
        if not isinstance(action, NCDepositAction):
            raise NCFail("Expected deposit action")

        # Check deadline
        if ctx.block.timestamp > self.date_last_bet:
            raise TooLate(f"Deadline was {self.date_last_bet}")

        # Check result not set
        if self.final_result is not None:
            raise NCFail("Betting closed, result already set")

        amount = action.amount
        self.total += amount

        # Update totals
        if score not in self.bets_total:
            self.bets_total[score] = amount
        else:
            self.bets_total[score] += amount

        # Update address bets
        key = (score, address)
        if key not in self.bets_address:
            self.bets_address[key] = amount
        else:
            self.bets_address[key] += amount

    @public
    def set_result(self, ctx: Context, result: SignedData[str]) -> None:
        # Verify oracle signature
        if not result.checksig(self.syscall.get_contract_id(), self.oracle_script):
            raise NCFail("Invalid oracle signature")

        self.final_result = result.data

    @public(allow_withdrawal=True)
    def withdraw(self, ctx: Context) -> None:
        if self.final_result is None:
            raise ResultNotAvailable()

        action = ctx.get_single_action(self.token_uid)
        if not isinstance(action, NCWithdrawalAction):
            raise NCFail("Expected withdrawal action")

        caller = ctx.get_caller_address()
        assert caller is not None

        max_allowed = self.get_max_withdrawal(caller)
        if action.amount > max_allowed:
            raise NCFail(f"Max withdrawal: {max_allowed}")

        # Track withdrawal
        if caller not in self.withdrawals:
            self.withdrawals[caller] = action.amount
        else:
            self.withdrawals[caller] += action.amount

    @view
    def get_max_withdrawal(self, address: Address) -> int:
        total_won = self.get_winner_amount(address)
        already_withdrawn = self.withdrawals.get(address, 0)
        return total_won - already_withdrawn

    @view
    def get_winner_amount(self, address: Address) -> int:
        if self.final_result is None:
            return 0

        if self.final_result not in self.bets_total:
            return 0

        result_total = self.bets_total[self.final_result]
        if result_total == 0:
            return 0

        address_bet = self.bets_address.get((self.final_result, address), 0)
        return address_bet * self.total // result_total

```

---

## Anti-Patterns

### Anti-Pattern 1: Not Initializing All Attributes
```python
❌ WRONG:
class TokenContract(Blueprint):
    balances: dict[Address, int]
    total_supply: int
    owner: Address

@public
def initialize(self, ctx: Context, owner: Address):
    self.owner = owner
    # Missing initialization of balances and total_supply!

✅ CORRECT:
class TokenContract(Blueprint):
    balances: dict[Address, int]
    total_supply: int
    owner: Address

@public
def initialize(self, ctx: Context, owner: Address):
    self.owner = owner
    self.balances = {}  # Initialize all attributes
    self.total_supply = 0
```

### Anti-Pattern 2: Using __init__
```python
❌ WRONG:
def __init__(self, initial_value: int):
    self.value = initial_value

✅ CORRECT:
@public
def initialize(self, ctx: Context, initial_value: int) -> None:
    self.value = initial_value
```

### Anti-Pattern 3: Missing Decorators
```python
❌ WRONG:
def transfer(self, ctx: Context, to: Address, amount: int):
    # No decorator!

✅ CORRECT:
@public
def transfer(self, ctx: Context, to: Address, amount: int) -> None:
    # Has @public decorator
```

### Anti-Pattern 4: View Method Modifying State
```python
❌ WRONG:
@view
def increment(self) -> int:
    self.count += 1  # NCViewMethodError!
    return self.count

✅ CORRECT:
@view
def get_count(self) -> int:
    return self.count  # Read-only

@public
def increment(self, ctx: Context) -> None:
    self.count += 1  # State change in @public
```

### Anti-Pattern 5: Default Values in Public/View Methods
```python
❌ WRONG:
@public
def transfer(self, ctx: Context, to: Address, amount: int = 100):
    # Default values not allowed!

@view
def get_info(self, detailed: bool = False) -> str:
    # Default values not allowed!

✅ CORRECT:
@public
def transfer(self, ctx: Context, to: Address, amount: int) -> None:
    # Caller must provide all arguments

@view
def get_info(self, detailed: bool) -> str:
    # All parameters required

# Internal methods can have defaults:
def _calculate(self, base: int, multiplier: int = 2) -> int:
    return base * multiplier
```

### Anti-Pattern 6: Missing Blueprint Export
```python
❌ WRONG:
class MyBlueprint(Blueprint):
    # ... methods ...
# Missing @export decorator!

✅ CORRECT:
from hathor import export

@export
class MyBlueprint(Blueprint):
    # ... methods ...
```

### Anti-Pattern 7: Forbidden Imports
```python
❌ WRONG:
import hashlib                          # NO - forbidden import
import datetime                         # NO - forbidden import
from datetime import datetime           # NO - forbidden import
from typing import Dict, List           # NO - only specific typing imports allowed
from json import loads                  # NO - forbidden import

def some_method(self):
    hash_val = hashlib.sha256(b"data")  # Will fail!

✅ CORRECT:
from hathor import sha3                 # Use Hathor's sha3 function
from typing import Optional, Union      # Only these typing imports allowed

def some_method(self):
    hash_val = sha3(b"data")           # Use Hathor's built-in hashing
```

### Anti-Pattern 8: Wrong Import Syntax
```python
❌ WRONG:
import hathor                          # NO - use 'from x import y' syntax only
from hathor import *                   # NO - wildcard imports forbidden

✅ CORRECT:
from hathor import Blueprint, public, view, Context, export  # Explicit imports only
```

### Anti-Pattern 9: Misusing Container Types
```python
❌ WRONG: Assuming standard Python container behavior
class BadContract(Blueprint):
    balances: dict[Address, int]

    @public
    def bad_method(self, ctx: Context) -> None:
        # These operations may NOT work as expected!
        keys = self.balances.keys()        # May not be implemented
        values = list(self.balances.values())  # May not work
        items = dict(self.balances.items())    # May not work

        # Wrong list operations
        self.my_list.sort()               # Not implemented
        self.my_list.insert(0, "item")    # Not implemented

        # Wrong set operations
        new_set = self.my_set.copy()      # Not implemented
        union_set = self.my_set.union(other)  # Not implemented

✅ CORRECT: Use supported container operations
class GoodContract(Blueprint):
    balances: dict[Address, int]
    history: list[str]
    members: set[Address]

    @public
    def good_method(self, ctx: Context) -> None:
        # ✅ Use supported DictContainer operations
        self.balances[address] = 100
        balance = self.balances.get(address, 0)
        if address in self.balances:
            del self.balances[address]

        # ✅ Use supported DequeContainer operations
        self.history.append("new_event")
        self.history.appendleft("urgent_event")
        recent = self.history.pop()

        # ✅ Use supported SetContainer operations
        self.members.add(address)
        self.members.discard(address)
        if address in self.members:
            pass
```

### Anti-Pattern 10: Container Initialization Errors
```python
❌ WRONG: Not initializing containers
class BadContract(Blueprint):
    balances: dict[Address, int]

    @public
    def initialize(self, ctx: Context) -> None:
        # Missing container initialization!
        pass

    @public
    def use_balances(self, ctx: Context) -> None:
        self.balances[ctx.get_caller_address()] = 100  # WILL FAIL!

✅ CORRECT: Always initialize containers
class GoodContract(Blueprint):
    balances: dict[Address, int]
    history: list[str]
    members: set[Address]

    @public
    def initialize(self, ctx: Context) -> None:
        # ✅ Initialize all declared containers
        self.balances = {}      # Empty DictContainer
        self.history = []       # Empty DequeContainer
        self.members = set()    # Empty SetContainer
```
