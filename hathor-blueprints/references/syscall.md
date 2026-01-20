# Syscall Reference

Access advanced system operations via `self.syscall`.

## Contract Interactions

```python
# Call another contract's public method
result = self.syscall.call_public_method(
    nc_id=other_contract_id,
    method_name="transfer",
    actions=[],
    from_addr=ctx.get_caller_address(),
    to_addr=recipient,
    amount=100
)

# Call another contract's view method
balance = self.syscall.call_view_method(
    nc_id=other_contract_id,
    method_name="get_balance",
    address=some_address
)

# Proxy call (delegatecall - runs other blueprint's code with current contract's state)
result = self.syscall.proxy_call_public_method(
    blueprint_id=other_blueprint_id,
    method_name="method_name",
    actions=[],
    arg1, arg2
)
```

## Token Management

```python
# Create new token
token_uid = self.syscall.create_token(
    token_name="MyToken",
    token_symbol="MTK",
    amount=1000,           # Initial supply (goes to contract)
    mint_authority=True,   # Contract can mint more
    melt_authority=True    # Contract can burn
)

# Mint tokens (requires mint authority)
self.syscall.mint_tokens(token_uid=token_uid, amount=500)

# Melt/burn tokens (requires melt authority)
self.syscall.melt_tokens(token_uid=token_uid, amount=200)

# Revoke authorities (IRREVERSIBLE!)
self.syscall.revoke_authorities(
    token_uid=token_uid,
    revoke_mint=True,
    revoke_melt=False
)
```

## Balance & Authority Queries

```python
# Get contract IDs
contract_id = self.syscall.get_contract_id()
blueprint_id = self.syscall.get_blueprint_id()
other_bp = self.syscall.get_blueprint_id(contract_id=other_contract_id)

# Get balance (includes current call's actions)
balance = self.syscall.get_current_balance(token_uid=token_uid, contract_id=None)

# Get balance before current call
balance_before = self.syscall.get_balance_before_current_call(token_uid=token_uid)

# Check authorities
can_mint = self.syscall.can_mint(token_uid)
can_melt = self.syscall.can_melt(token_uid)
can_mint_before = self.syscall.can_mint_before_current_call(token_uid)
can_melt_before = self.syscall.can_melt_before_current_call(token_uid)
```

## Contract Creation

```python
new_contract_id, init_result = self.syscall.create_contract(
    blueprint_id=child_blueprint_id,
    salt=b"unique_salt",  # For deterministic addresses
    actions=[],
    initial_value=100     # Constructor arguments
)
```

## Events & Upgrades

```python
self.syscall.emit_event(b"Transfer successful")
self.syscall.change_blueprint(new_blueprint_id)  # Advanced!
```

## Random Number Generation (RNG)

Hathor provides built-in deterministic RNG via `self.syscall.rng`. Uses ChaCha20 cipher.

**Properties:**
- Deterministic: same transaction seed → same random sequence
- Per-contract isolation: each contract gets its own derived RNG
- Consensus-safe: all nodes produce identical numbers

### Basic Methods

```python
rng = self.syscall.rng

rng.randbytes(32)           # Random bytes of specified size
rng.randint(1, 6)           # Random int in [a, b] (inclusive both ends)
rng.random()                # Random float in [0, 1)
rng.randbits(256)           # Random int with n bits (0 <= x < 2^n)
rng.randbelow(10)           # Random int in [0, n) (exclusive upper)
rng.randrange(0, 100, 2)    # Random in range with step (0, 2, 4, ..., 98)
rng.choice(["A", "B", "C"]) # Random element from sequence
rng.seed                    # bytes (32 bytes) - read-only
```

### Security Considerations

- ✅ **Good for**: Games, lotteries, random selection, dice rolls
- ❌ **NOT for**: Cryptographic keys, signatures, secrets
- ⚠️ Transaction creator can influence seed by choosing inputs

### Commit-Reveal Pattern (Unbiasable Randomness)

For high-stakes randomness where seed manipulation is a concern:

```python
class Lottery(Blueprint):
    commitments: dict[Address, bytes]
    reveals: dict[Address, bytes]

    @public
    def commit(self, ctx: Context, commitment: bytes) -> None:
        self.commitments[ctx.get_caller_address()] = commitment

    @public
    def reveal(self, ctx: Context, secret: bytes) -> None:
        caller = ctx.get_caller_address()
        if sha3(secret) != self.commitments[caller]:
            raise NCFail("Invalid reveal")
        self.reveals[caller] = secret

    @public
    def draw(self, ctx: Context) -> None:
        combined = b"".join(self.reveals.values())
        seed = sha3(combined)
        # Use seed for selection
```
