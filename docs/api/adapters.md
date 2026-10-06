# Adapters: typed registry helpers

Adapters implement the contract in `honba/python/src/honba/adapters/base.py` and are found
through the registry in `honba/python/src/honba/adapters/registry.py`. Adapter packages advertise
themselves under the `honba.adapters` entry-point group; discovery is lazy and explicit
registration always wins. See also [Adapter Interface](../architecture/adapter_interface.md).

## Roles and Client protocols

`Adapter` is the facade (`capabilities()`, `connect()`, `disconnect()`, `session()`,
`is_connected()`). Two role protocols say what an adapter can do:

| Protocol | Methods |
|---|---|
| `MarketDataAdapter` | `instruments`, `search_instruments`, `quote`, `depth`, `historical_bars`, `subscribe`, `unsubscribe` |
| `ExecutionAdapter` | `place_order`, `modify_order`, `cancel_order`, `cancel_all`, `order_status`, `orders`, `trades`, `positions`, `holdings`, `funds`, `margin` |

`MarketDataClient` and `ExecutionClient` combine the lifecycle members (`name`, `capabilities`,
`connect`, `disconnect`, `is_connected`, `session`) with one role. All four protocols are
`@runtime_checkable`.

## Creating a typed adapter

`create_market_data` and `create_execution` build an adapter by name, check its role and return
it typed, so a type checker sees the role's methods:

```python
from honba.adapters.registry import default_registry

registry = default_registry()
adapter = registry.create_market_data(name, **config)
await adapter.connect()
found = await adapter.search_instruments("INFY")
```

`create_execution(name, **config)` works the same way for order placement. Module-level
shortcuts over the default registry are `resolve_market_data_adapter(name, **config)` and
`resolve_execution_adapter(name, **config)`; `resolve_adapter` returns the untyped `Adapter`.

Errors (all in `honba.adapters.errors`):

- `AdapterNotFound` if no adapter is registered under `name` (the message lists the installed
  ones).
- `AdapterError` if the factory returns something that is not an `Adapter`, or if the adapter
  does not implement the requested role (`adapter 'x' does not implement market data` /
  `execution`).

## The role check is structural

The role check is `isinstance(adapter, MarketDataClient)` (or `ExecutionClient`) against a
runtime-checkable `Protocol`. That only tests that the **members exist** (the role's methods, the lifecycle methods and `name`). It does not consult
`capabilities()`: an adapter that has the methods but declares no such capability still passes
the check. Capabilities are enforced separately, when the adapter refuses an unsupported call with
`CapabilityError` before sending a request (`require_capabilities`).
