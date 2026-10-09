# Indian Market Calendar

The Indian market calendar is not a simple "weekdays minus holidays" rule.
It has pre-open sessions, special sessions (Muhurat), segment-specific
holidays, and settlement holidays that differ from trading holidays. Honba
models all of these in `honba-market::calendar` (or Python `honba.markets.india.calendar`).

*Last verified: 2026-09-30 against NSE circulars.*

## Trading Hours

The normal equity and F&O session on NSE and BSE:

| Session | IST window | Notes |
|---|---|---|
| Pre-open call auction | 09:00 – 09:15 | Orders collected, no matching |
| Pre-open matching | 09:08 – 09:15 | Indicative price, no trades |
| Normal trading | 09:15 – 15:30 | Continuous matching |
| Closing session | 15:30 – 15:40 | Closing price discovery |
| Post-close | 15:40 – 16:00 | Market orders at closing price |

The pre-open session has three sub-phases: order collection (09:00–09:08),
order matching and price discovery (09:08–09:12), and buffer period
(09:12–09:15). Strategies that operate on the open should typically start
processing at 09:15:00, not earlier.

Honba represents session boundaries as `Session` enum values:

    pub enum Session {
        PreOpen,
        PreOpenMatching,
        Normal,
        Closing,
        PostClose,
    }

`MarketCalendar::session_at(ts: DateTime<IST>) -> Option<Session>` returns
`None` outside trading hours.

## 2026 Trading Holidays

NSE and BSE observe **15 full trading holidays** in 2026. Both exchanges
close on all of them for equities and derivatives.[reference:0]

| # | Date | Day | Occasion |
|---|---|---|---|
| 1 | 26 Jan 2026 | Mon | Republic Day |
| 2 | 3 Mar 2026 | Tue | Holi |
| 3 | 26 Mar 2026 | Thu | Shri Ram Navami |
| 4 | 31 Mar 2026 | Tue | Shri Mahavir Jayanti |
| 5 | 3 Apr 2026 | Fri | Good Friday |
| 6 | 14 Apr 2026 | Tue | Dr. Baba Saheb Ambedkar Jayanti |
| 7 | 1 May 2026 | Fri | Maharashtra Day |
| 8 | 28 May 2026 | Thu | Bakri Id |
| 9 | 26 Jun 2026 | Fri | Muharram |
| 10 | 14 Sep 2026 | Mon | Ganesh Chaturthi |
| 11 | 2 Oct 2026 | Fri | Mahatma Gandhi Jayanti |
| 12 | 20 Oct 2026 | Tue | Dussehra |
| 13 | 10 Nov 2026 | Tue | Diwali-Balipratipada |
| 14 | 24 Nov 2026 | Tue | Prakash Gurpurb Sri Guru Nanak Dev |
| 15 | 25 Dec 2026 | Fri | Christmas |

[reference:1]

Note that **15 Jan 2026** (Municipal Corporation Election – Maharashtra)
appears in several published lists but is a settlement holiday, not a
trading holiday.[reference:2] The distinction matters: on a settlement
holiday, trading may be open but settlement is deferred.

## Muhurat Trading

Muhurat trading is a special one-hour session held on Diwali Laxmi Pujan.
In 2026, it falls on **8 November 2026 (Sunday)**. Timings are notified
separately by NSE each year, typically in October. Honba's calendar
includes Muhurat as a distinct session type because:

- The session is short (usually 18:15–19:15 IST).
- Pre-open rules differ.
- Some brokers do not route API orders during Muhurat.

    pub enum DayType {
        TradingDay,
        TradingHoliday,
        Weekend,
        MuhuratTrading,   // special
    }

## Settlement Holidays vs Trading Holidays

Settlement holidays are dates when clearing and settlement do not occur,
even if trading is open. Honba tracks both because a strategy that holds
delivery positions overnight needs to know when funds will actually be
available.

Selected 2026 settlement holidays (not all coincide with trading
holidays):[reference:3]

| Date | Occasion |
|---|---|
| 19 Feb 2026 | Chhatrapati Shivaji Maharaj Jayanti |
| 19 Mar 2026 | Gudhi Padwa |
| 1 Apr 2026 | Annual Bank Closing |
| 26 Aug 2026 | Id-E-Milad |

## Weekend Holidays

Some holidays fall on weekends and therefore do not reduce the trading
count. The 2026 list includes Mahashivratri (15 Feb, Sunday), Id-Ul-Fitr
(21 Mar, Saturday), Independence Day (15 Aug, Saturday), and Diwali Laxmi
Pujan (8 Nov, Sunday — but Muhurat trading occurs).[reference:4]

## The `MarketCalendar` API

    use honba_market::calendar::{MarketCalendar, NseCalendar, Session, DayType};

    let cal = NseCalendar::new(2026);

    cal.is_trading_day(NaiveDate::from_ymd_opt(2026, 3, 3).unwrap());
    // → false (Holi)

    cal.is_trading_day(NaiveDate::from_ymd_opt(2026, 1, 15).unwrap());
    // → true (settlement holiday only)

    cal.session_at(datetime!(2026-01-02 09:10:00 IST));
    // → Some(Session::PreOpenMatching)

    cal.session_at(datetime!(2026-01-02 15:35:00 IST));
    // → Some(Session::Closing)

    cal.next_trading_day(NaiveDate::from_ymd_opt(2026, 3, 2).unwrap());
    // → 2026-03-04 (skips Holi on 3 Mar)

## Segment-Specific Calendars

Not every segment follows the equity calendar:

- **Currency derivatives** trade on some days when equities are closed
  (for example, certain US holidays do not affect INR pairs, but the
  equity market may be closed).
- **Commodity derivatives (MCX)** follow a partially different holiday
  list and have an evening session.
- **Debt market** has a separate calendar published by RBI/FIMMDA.

Honba currently models the **equity and equity derivatives** calendar
only. Currency and commodity calendars are planned.

## Configuring the Calendar

    # configs/backtest/nifty50_momentum.toml
    [calendar]
    exchange = "NSE"
    year = 2026
    include_pre_open = false
    include_post_close = false
    muhurat_trading = true

If `include_pre_open = false`, bars whose timestamp falls in the
09:00–09:15 window are discarded before reaching the strategy. This is
the safe default for strategies that were not designed for the auction
session.

## Testing

    #[test]
    fn holi_2026_is_not_a_trading_day() {
        let cal = NseCalendar::new(2026);
        let d = NaiveDate::from_ymd_opt(2026, 3, 3).unwrap();
        assert!(!cal.is_trading_day(d));
    }

    #[test]
    fn jan_15_2026_is_trading_but_not_settlement() {
        let cal = NseCalendar::new(2026);
        let d = NaiveDate::from_ymd_opt(2026, 1, 15).unwrap();
        assert!(cal.is_trading_day(d));
        assert!(!cal.is_settlement_day(d));
    }

## Source of Truth

The authoritative calendar is published by NSE in an annual circular
(usually in December for the following year) and updated by amendment.
BSE publishes a mirror. Honba's calendar data lives in
`crates/honba-market/src/india/calendar/` and `honba.markets.india.calendar` and is regenerated from the
circular text. When NSE issues an amendment, update the data file and
bump the patch version of `honba-market`.
