# Container Types Reference

Hathor containers persist to blockchain storage via Merkle Patricia Trie. They look like Python containers but have different capabilities.

## DictContainer (`dict[K, V]`)

### Works
```python
self.balances[address] = 100          # __setitem__
balance = self.balances[address]       # __getitem__
balance = self.balances.get(addr, 0)   # get() with default
exists = address in self.balances      # __contains__
del self.balances[address]             # __delitem__
length = len(self.balances)            # __len__
self.balances.update({a: 1, b: 2})     # update()
```

### Does NOT Work
```python
self.balances.keys()       # NOT IMPLEMENTED
self.balances.values()     # NOT IMPLEMENTED
self.balances.items()      # NOT IMPLEMENTED
self.balances.copy()       # NOT IMPLEMENTED
self.balances.pop(key)     # NOT IMPLEMENTED
self.balances.popitem()    # NOT IMPLEMENTED
self.balances.clear()      # NOT IMPLEMENTED
self.balances.setdefault() # NOT IMPLEMENTED
dict(self.balances)        # FAILS
for k in self.balances:    # NO ITERATION
```

### Workaround for Iteration
Track keys separately if needed:
```python
class Contract(Blueprint):
    balances: dict[Address, int]
    balance_keys: list[Address]  # Track keys manually
```

## DequeContainer (`list[T]`)

### Works
```python
self.items.append("x")         # append()
self.items.extend([a, b])      # extend()
self.items.appendleft("x")     # appendleft()
self.items.extendleft([a, b])  # extendleft()
item = self.items[0]           # __getitem__
self.items[0] = "new"          # __setitem__
last = self.items.pop()        # pop()
first = self.items.popleft()   # popleft()
self.items.reverse()           # reverse()
length = len(self.items)       # __len__
for item in self.items:        # __iter__ WORKS
    pass
```

### Does NOT Work
```python
self.items.insert(i, x)   # NOT IMPLEMENTED
self.items.remove(x)      # NOT IMPLEMENTED
self.items.sort()         # NOT IMPLEMENTED
self.items.index(x)       # NOT IMPLEMENTED
self.items.count(x)       # NOT IMPLEMENTED
self.items.copy()         # NOT IMPLEMENTED
self.items.clear()        # NOT IMPLEMENTED
```

### Workaround for index()
```python
for i, item in enumerate(self.items):
    if item == target:
        found_index = i
        break
```

## SetContainer (`set[T]`)

### Works
```python
self.members.add(address)              # add()
self.members.discard(address)          # discard() - no error if missing
self.members.remove(address)           # remove() - KeyError if missing
self.members.update([a, b, c])         # update()
exists = address in self.members       # __contains__
length = len(self.members)             # __len__
self.members.isdisjoint(other_list)    # isdisjoint()
self.members.issuperset(other_list)    # issuperset()
self.members.intersection(other_list)  # intersection()
self.members.difference_update(other)  # difference_update()
```

### Does NOT Work
```python
self.members.copy()                    # NOT IMPLEMENTED
self.members.union(other)              # NOT IMPLEMENTED
self.members.difference(other)         # NOT IMPLEMENTED
self.members.symmetric_difference()    # NOT IMPLEMENTED
self.members.issubset(other)           # NOT IMPLEMENTED
self.members.pop()                     # NOT IMPLEMENTED
self.members.clear()                   # NOT IMPLEMENTED
self.members | other                   # NOT IMPLEMENTED
self.members & other                   # NOT IMPLEMENTED
set(self.members)                      # MAY FAIL
for m in self.members:                 # NO ITERATION
```

### Workaround for union()
```python
for item in other_items:
    self.members.add(item)
```

## Initialization

All containers must be initialized in `initialize()`:
```python
@public
def initialize(self, ctx: Context) -> None:
    self.balances = {}      # Empty DictContainer
    self.history = []       # Empty DequeContainer
    self.members = set()    # Empty SetContainer

    # Or with initial values:
    self.balances = {owner: 1000}
    self.history = ["Created"]
    self.members = {owner}
```
