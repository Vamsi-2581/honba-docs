# Market Calendar

`honba-market` (and Python `honba.markets.india.calendar`) models the Indian trading calendar as traits plus concrete implementations.

## Traits

### `TradingCalendar`

The interface every exchange calendar implements.

| Method | Purpose |
|---|---|
| `is_trading_day(date) -> bool` | Weekday and not a holiday |
| `session_open(date) -> Option<Time>` | Opening time, `None` on closed days |
| `session_close(date) -> Option<Time>` | Closing time |
| `next_trading_day(date) -> Option<NaiveDate>` | First trading day strictly after `date` |
| `prev_trading_day(date) -> Option<NaiveDate>` | First trading day strictly before `date` |
| `trading_days_between(start, end) -> Vec<NaiveDate>` | Inclusive range, holidays removed |

### `HolidaySource`

Anything that can produce a set of holiday dates. Implement this for a
static list, a CSV file, or a network-backed source.

| Method | Purpose |
|---|---|
| `holidays(year: i32) -> Result<BTreeSet<NaiveDate>, Self::Error>` | All holidays for a calendar year |

## Concrete implementation: `NseCalendar`

Two constructors:

```rust
use honba_market::{NseCalendar, HolidaySource};

// From an in-memory list
let cal = NseCalendar::from_holidays(vec![
    "2025-01-26".parse().unwrap(), // Republic Day
    "2025-03-14".parse().unwrap(), // Holi
]);

// From any HolidaySource
let cal = NseCalendar::from_source(my_csv_source)?;
```

## Session windows

NSE equity sessions run 09:15–15:30 IST. The calendar exposes them
per-date so a closed day returns `None`:

```rust
match cal.session_open(date) {
    Some(t) => println!("opens at {t}"),
    None    => println!("market closed"),
}
```

## Navigating around holidays

```rust
let next = cal.next_trading_day(NaiveDate::from_ymd_opt(2025, 1, 24).unwrap());
// skips the weekend and Republic Day → 2025-01-27 (Monday)

let days = cal.trading_days_between(
    NaiveDate::from_ymd_opt(2025, 1, 20).unwrap(),
    NaiveDate::from_ymd_opt(2025, 1, 31).unwrap(),
);
// 7 weekdays, minus any holidays in that window
```

## Why no bundled data

Exchange holidays are revised yearly and differ by segment (equity vs
F&O vs currency). Shipping a hardcoded list guarantees the calendar goes
stale. Instead:

1. Ship the traits and the NSE session times.
2. Let the user provide holidays via `HolidaySource`.
3. Provide a CSV-backed source as a convenience.

## Testing

`NseCalendar::from_holidays(vec![])` gives a pure weekday calendar. Use
it in unit tests where holidays would only obscure the logic under test.
