# Complete Blueprint Examples

## Simple Counter

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

## Counter with Containers

```python
from hathor import Blueprint, public, view, Context, export, Address

@export
class CounterWithHistory(Blueprint):
    count: int
    history: list[str]              # DequeContainer
    user_counts: dict[Address, int] # DictContainer
    authorized: set[Address]        # SetContainer

    @public
    def initialize(self, ctx: Context, initial: int) -> None:
        self.count = initial
        self.history = []
        self.user_counts = {}
        self.authorized = set()

    @public
    def increment(self, ctx: Context) -> None:
        caller = ctx.get_caller_address()
        if caller not in self.authorized:
            self.authorized.add(caller)
        self.count += 1
        self.history.append(f"Incremented to {self.count}")
        self.user_counts[caller] = self.user_counts.get(caller, 0) + 1

    @view
    def get_user_count(self, user: Address) -> int:
        return self.user_counts.get(user, 0)
```

## Token Vault (Deposits & Withdrawals)

```python
from hathor import (
    Blueprint, public, view, Context, export,
    Address, TokenUid, NCDepositAction, NCWithdrawalAction, NCFail
)

@export
class TokenVault(Blueprint):
    token_uid: TokenUid
    balances: dict[Address, int]

    @public
    def initialize(self, ctx: Context, token_uid: TokenUid) -> None:
        self.token_uid = token_uid
        self.balances = {}

    @public(allow_deposit=True)
    def deposit(self, ctx: Context) -> None:
        action = ctx.get_single_action(self.token_uid)
        assert isinstance(action, NCDepositAction)
        caller = ctx.get_caller_address()
        self.balances[caller] = self.balances.get(caller, 0) + action.amount

    @public(allow_withdrawal=True)
    def withdraw(self, ctx: Context) -> None:
        action = ctx.get_single_action(self.token_uid)
        assert isinstance(action, NCWithdrawalAction)
        caller = ctx.get_caller_address()
        balance = self.balances.get(caller, 0)
        if action.amount > balance:
            raise NCFail(f"Insufficient: have {balance}, want {action.amount}")
        self.balances[caller] = balance - action.amount

    @view
    def get_balance(self, address: Address) -> int:
        return self.balances.get(address, 0)
```

## Dice Game (Using RNG)

```python
from hathor import (
    Blueprint, public, view, Context, export,
    NCFail, Amount, CallerId, TokenUid
)

@export
class HathorDice(Blueprint):
    token_uid: TokenUid
    max_bet: Amount
    house_edge_bps: int      # Basis points (50 = 0.50%)
    bit_length: int
    balances: dict[CallerId, int]
    liquidity: Amount

    @public
    def initialize(self, ctx: Context, token_uid: TokenUid,
                   house_edge_bps: int, max_bet: Amount, bit_length: int) -> None:
        if bit_length < 16 or bit_length > 32:
            raise NCFail('bit_length must be 16-32')
        self.token_uid = token_uid
        self.house_edge_bps = house_edge_bps
        self.max_bet = max_bet
        self.bit_length = bit_length
        self.balances = {}
        self.liquidity = 0

    @public(allow_deposit=True)
    def bet(self, ctx: Context, amount: Amount, threshold: int) -> int:
        """Win if random < threshold. Returns payout (0 if lost)."""
        if amount > self.max_bet:
            raise NCFail('Bet too high')

        lucky = self.syscall.rng.randbits(self.bit_length)

        if lucky >= threshold:
            self.liquidity += amount
            self.syscall.emit_event(f'{{"result":"lose","num":{lucky}}}'.encode())
            return 0

        payout = self._calc_payout(amount, threshold)
        if payout > self.liquidity:
            raise NCFail('Insufficient liquidity')

        self.liquidity -= (payout - amount)
        self.balances[ctx.caller_id] = self.balances.get(ctx.caller_id, 0) + payout
        self.syscall.emit_event(f'{{"result":"win","payout":{payout}}}'.encode())
        return payout

    def _calc_payout(self, amount: Amount, threshold: int) -> int:
        num = amount * (2**self.bit_length) * (10_000 - self.house_edge_bps)
        return num // (10_000 * threshold)
```

## Betting Contract (With Oracle)

```python
from typing import Optional
from hathor import (
    Blueprint, public, view, Context, export,
    Address, TokenUid, NCDepositAction, NCWithdrawalAction,
    SignedData, TxOutputScript, NCFail
)

@export
class Bet(Blueprint):
    bets_total: dict[str, int]
    bets_address: dict[tuple[str, Address], int]
    withdrawals: dict[Address, int]
    total: int
    result: Optional[str]
    oracle_script: TxOutputScript
    deadline: int
    token_uid: TokenUid

    @public
    def initialize(self, ctx: Context, oracle_script: TxOutputScript,
                   token_uid: TokenUid, deadline: int) -> None:
        self.bets_total = {}
        self.bets_address = {}
        self.withdrawals = {}
        self.oracle_script = oracle_script
        self.token_uid = token_uid
        self.deadline = deadline
        self.result = None
        self.total = 0

    @public(allow_deposit=True)
    def bet(self, ctx: Context, address: Address, score: str) -> None:
        if self.result is not None:
            raise NCFail("Betting closed")
        if ctx.block.timestamp > self.deadline:
            raise NCFail("Past deadline")

        action = ctx.get_single_action(self.token_uid)
        assert isinstance(action, NCDepositAction)

        self.total += action.amount
        self.bets_total[score] = self.bets_total.get(score, 0) + action.amount
        key = (score, address)
        self.bets_address[key] = self.bets_address.get(key, 0) + action.amount

    @public
    def set_result(self, ctx: Context, signed_result: SignedData[str]) -> None:
        if not signed_result.checksig(self.syscall.get_contract_id(), self.oracle_script):
            raise NCFail("Invalid signature")
        self.result = signed_result.data

    @public(allow_withdrawal=True)
    def withdraw(self, ctx: Context) -> None:
        if self.result is None:
            raise NCFail("Result not set")
        caller = ctx.get_caller_address()
        max_amount = self.get_winnings(caller)
        action = ctx.get_single_action(self.token_uid)
        assert isinstance(action, NCWithdrawalAction)
        if action.amount > max_amount:
            raise NCFail(f"Max withdrawal: {max_amount}")
        self.withdrawals[caller] = self.withdrawals.get(caller, 0) + action.amount

    @view
    def get_winnings(self, address: Address) -> int:
        if self.result is None or self.result not in self.bets_total:
            return 0
        result_total = self.bets_total[self.result]
        if result_total == 0:
            return 0
        bet = self.bets_address.get((self.result, address), 0)
        total_won = bet * self.total // result_total
        return total_won - self.withdrawals.get(address, 0)
```

## Testing Example

```python
from hathor import make_nc_type_for_arg_type as make_nc_type
from tests.nanocontracts.blueprints.unittest import BlueprintTestCase

INT_NC_TYPE = make_nc_type(int)

class CounterTest(BlueprintTestCase):
    def setUp(self) -> None:
        super().setUp()
        from counter import Counter
        self.blueprint_id = self._register_blueprint_class(Counter)

    def test_counter(self) -> None:
        nc_id = self.gen_random_contract_id()
        ctx = self.create_context()

        self.runner.create_contract(nc_id, self.blueprint_id, ctx, 0)
        storage = self.runner.get_storage(nc_id)
        self.assertEqual(storage.get_obj(b'count', INT_NC_TYPE), 0)

        self.runner.call_public_method(nc_id, 'increment', ctx)
        self.assertEqual(storage.get_obj(b'count', INT_NC_TYPE), 1)

        result = self.runner.call_view_method(nc_id, 'get_count')
        self.assertEqual(result, 1)
```
