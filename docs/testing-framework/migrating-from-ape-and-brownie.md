# Migrating from Ape & Brownie

## Importing contracts

In Woke, contract types must be imported from `pytypes`, a directory generated using:

```shell
woke init pytypes
```

An optional `-w` flag can be used to generate `pytypes` in a watch mode.

If there is a `Counter` contract in `contracts/Counter.sol`, then the following import statement can be used:

```python
from pytypes.contracts.Counter import Counter
```

A contract named `ERC1967Proxy` in `node_modules/@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol` can be imported using:

```python
from pytypes.node_modules.openzeppelin.contracts.proxy.ERC1967.ERC1967Proxy import ERC1967Proxy
```

## Accessing accounts

In Woke, accounts are a property of a chain. With the default chain instance named `default_chain`:

```python
from woke.testing import *

@default_chain.connect()
def test_accounts():
    print(default_chain.accounts)
```

## Configuring networks

Woke does not support configuring networks in configuration files. Instead, a chain instance can be created:

- without a URI (`@default_chain.connect()`), which will launch a new development chain,
- with a URI (`@default_chain.connect("http://localhost:8545")`), which will connect to an existing chain.

A development chain executable and its arguments can be configured in `woke.toml` in the project root:

```toml title="woke.toml"
[testing]
cmd = "anvil"  # other options: "hardhat", "ganache"

[testing.anvil]
cmd_args = "--prune-history 100 --transaction-block-keeper 10 --steps-tracing --silent"
```

Commonly used parameters can be set as keyword arguments in `@default_chain.connect()`:

```python
@default_chain.connect(
    accounts=20,  # number of accounts to generate
    chain_id=1337,  # chain ID
    fork="https://eth-mainnet.alchemyapi.io/v2/...@12345678",  # fork from a block
    hardfork="london",  # hardfork to use
)
```

## Events and user-defined errors

Events and user-defined errors are generated in `pytypes` in a form of dataclasses.

If there is an event named `Incremented` and error named `NotOwner` in `contracts/Counter.sol`, then the following can be used to test the contract:

```python
from woke.testing import *
from pytypes.contracts.Counter import Counter

@default_chain.connect()
def test_counter():
    default_chain.set_default_accounts(default_chain.accounts[0])

    counter = Counter.deploy()
    tx = counter.increment()
    assert Counter.Incremented() in tx.events

    acc = default_chain.accounts[1]
    with must_revert(Counter.NotOwner()):
        counter.addToWhitelist(acc, from_=acc)
```

## Transaction parameters

Like in Ape, Woke uses keyword arguments to specify transaction parameters. A transaction sender can be specified using `from_`:

```python
# Ape
counter.increment(sender=acc)

# Brownie
counter.increment({'from': acc})

# Woke
counter.increment(from_=acc)
```

## Expecting reverts

Woke uses `may_revert` and `must_revert` context managers to expect reverts:

```python

# Ape
with ape.reverts(r"b'NH{q\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x11'"):
    counter.decrement()

# Brownie
with brownie.reverts("Integer overflow"):
    counter.decrement()

# Woke
with must_revert(Panic(PanicCodeEnum.UNDERFLOW_OVERFLOW)):
    counter.decrement()
```

## Multi-chain testing

Woke does not use context managers to change the current chain interface. Instead, the `chain` keyword argument can be passed when deploying a contract:

```python
from woke.testing import *
from pytypes.contracts.Counter import Counter

chain1 = Chain()
chain2 = Chain()

@chain1.connect()
@chain2.connect()
def test_counter():
    counter1 = Counter.deploy(from_=chain1.accounts[0], chain=chain1)
    counter2 = Counter.deploy(from_=chain2.accounts[0], chain=chain2)
```
