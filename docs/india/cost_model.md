# Cost Model

`honba-market` (and Python `honba.markets.india.costs`) separates *rates* (what the tax and fee schedule looks
like) from *application* (how they are applied to a specific fill).

## Rates: `SttRates`

Securities Transaction Tax is charged per segment, and the rate differs
between delivery and intraday. `SttRates::new` takes the full schedule:

```rust
use honba_market::{SttRates, Segment};

let stt = SttRates::new(
    /* delivery_buy_bps  */ 10.0,
    /* delivery_sell_bps */ 10.0,
    /* intraday_sell_bps */ 2.5,
    /* options_sell_bps  */ 6.25,
);
```

All rates are **basis points** (1 bps = 0.01%). Buy-side rates are zero
for intraday and options on NSE — that is the actual rule, not an
oversight.

## Segments

`Segment` identifies which rate table applies:

| Variant | Applies to |
|---|---|
| `Segment::EquityDelivery` | CNC equity |
| `Segment::EquityIntraday` | MIS equity |
| `Segment::Futures` | F&O futures |
| `Segment::Options` | F&O options |

## The model: `CostModel`

`CostModel::new` takes the full set of charges and produces a
`CostBreakdown` per fill:

```rust
use honba_market::{CostModel, SttRates, Segment};

let model = CostModel::new(
    stt,
    /* brokerage_bps  */ 3.0,
    /* exchange_bps   */ 0.325,
    /* gst_pct        */ 18.0,
    /* stamp_bps      */ 0.3,
    /* sebi_bps       */ 0.01,
);

let breakdown = model.compute(
    /* price   */ 22_450.0,
    /* qty     */ 50.0,
    /* side    */ Side::Buy,
    /* segment */ Segment::EquityDelivery,
);
```

## Python Statutory Costs

In Python backtests, statutory costs are calculated using the built-in NSE cost model:

```python
from honba.markets.india.costs import nse_equity_delivery_fill_cost
from honba.domain.order import OrderSide

cost = nse_equity_delivery_fill_cost(OrderSide.BUY, quantity=100.0, price=2500.0)
```
