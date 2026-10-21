# Money, Currency & Indian Cost Modeling

> **Zero Floats Policy** — In Honba, floating-point numbers (`f32`/`f64`) are strictly forbidden across all prices, quantities, portfolio holdings, margins, and cost calculations. All financial math is conducted with exact 128-bit decimal arithmetic via `rust_decimal::Decimal`.

---

## 1. Dimensional Types & Monetary Purity

In trading systems, treating prices, quantities, and cash as generic floats inevitably introduces rounding errors, IEEE 754 precision drifts (e.g., `0.1 + 0.2 != 0.3`), and dimensional bugs (such as accidentally adding an equity share quantity to an INR cash balance).

Honba enforces type safety through distinct value objects defined in `honba-core`:

```rust
use rust_decimal::Decimal;

/// An exact execution or quote price (e.g. ₹2,450.75)
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
pub struct Price(pub Decimal);

/// An exact share or contract count (e.g. 50 shares, 0.001 crypto)
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
pub struct Quantity(pub Decimal);

/// An exact monetary amount in a specific currency minor-unit
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
pub struct Money {
    pub amount: Decimal,
    pub currency: Currency,
}
```

### Type Invariants & Operations
* `Price * Quantity = Money` (with currency derived from the instrument quote currency).
* `Money + Money = Money` (only valid when currencies match; cross-currency operations require an explicit FX rate).
* Floating-point numbers cannot be used to instantiate `Price`, `Quantity`, or `Money`. Conversions must use string parsing (`Decimal::from_str("2450.75")`) or exact integer ratios.

---

## 2. Currency Minor Units

`Currency` represents the fiat or digital currency and defines its decimal minor exponent:

| Currency | Symbol | Minor Unit Name | `minor_exponent` | Minor Units per Major |
| :---: | :---: | :---: | :---: | :---: |
| **INR** | ₹ | Paisa / Paise | 2 | 100 |
| **USD** | $ | Cent / Cents | 2 | 100 |
| **EUR** | € | Cent / Cents | 2 | 100 |
| **GBP** | £ | Penny / Pence | 2 | 100 |
| **JPY** | ¥ | Yen | 0 | 1 |

For the Indian market, all calculations settle in **INR**. `Money::format_minor()` formats values into standard financial notation (e.g., `₹1,25,000.50` or `12,500,050 paise`).

---

## 3. Comprehensive Indian Statutory Cost Model

Backtesting without realistic Indian transaction costs produces completely deceptive returns, particularly for intraday and options strategies where statutory taxes can consume 30% to 70% of gross profits.

`honba-ya-backtest` and `honba-kokoro-portfolio` implement the full Indian statutory cost stack using exact `rust_decimal::Decimal` arithmetic:

```text
                                GROSS TURNOVER
       ┌──────────────────────────────┴──────────────────────────────┐
       │                                                             │
       ▼                                                             ▼
  STATUTORY TAXES                                            EXCHANGE & SEBI FEES
  - STT (Securities Transaction Tax)                         - Exchange Turnover Charges (NSE/BSE)
  - Stamp Duty (State Revenue)                               - SEBI Turnover Fee (₹10 / crore)
  - GST (18% on Brokerage & Fees)                            - Clearing / DP Charges
```

### 3.1 Statutory Rates by Asset Class & Product

| Cost Component | Equity Delivery (CNC) | Equity Intraday (MIS) | Equity Futures | Equity Options (on Premium) |
| :--- | :--- | :--- | :--- | :--- |
| **STT (Securities Transaction Tax)** | 0.1% on Buy & Sell | 0.025% on Sell only | 0.0125% on Sell only | 0.0625% on Sell (exercised: 0.125%) |
| **Stamp Duty** | 0.015% on Buy only | 0.003% on Buy only | 0.002% on Buy only | 0.003% on Buy only |
| **Exchange Turnover Fee (NSE)** | 0.00297% on Buy & Sell | 0.00297% on Buy & Sell | 0.00173% on Buy & Sell | 0.03503% on Buy & Sell (Premium) |
| **SEBI Turnover Charges** | ₹10 per crore (0.0001%) | ₹10 per crore (0.0001%) | ₹10 per crore (0.0001%) | ₹10 per crore (0.0001%) |
| **GST (Goods & Services Tax)** | 18% on (Brokerage + Exchange + SEBI) | 18% on (Brokerage + Exchange + SEBI) | 18% on (Brokerage + Exchange + SEBI) | 18% on (Brokerage + Exchange + SEBI) |
| **DP Charges (Depository Fee)** | ₹13.50 + 18% GST per scrip debit | Nil | Nil | Nil |

### 3.2 Exact Mathematical Formulation

For an equity delivery transaction of $Q$ shares at price $P$:

$$\text{Turnover} = P \times Q$$

$$\text{STT} = \text{round}(\text{Turnover} \times 0.001)$$

$$\text{StampDuty} = \text{floor}(\text{Turnover} \times 0.00015)$$

$$\text{ExchangeCharges} = \text{Turnover} \times 0.0000297$$

$$\text{SEBICharges} = \text{Turnover} \times 0.000001$$

$$\text{TaxableBase} = \text{Brokerage} + \text{ExchangeCharges} + \text{SEBICharges}$$

$$\text{GST} = \text{TaxableBase} \times 0.18$$

$$\text{TotalCosts} = \text{Brokerage} + \text{STT} + \text{StampDuty} + \text{ExchangeCharges} + \text{SEBICharges} + \text{GST} + \text{DPCharges}$$

All divisions and multiplications use `Decimal` with rounding modes calibrated to Indian Clearing Corporation Limited (ICCL) clearing rules.

---

## 4. T+1 Rolling Settlement Lifecycle

India operates on a strict **T+1 rolling settlement** cycle for all equity cash transactions:

* **Trade Day ($T$)**:
  - The trade executes.
  - Required margin is blocked immediately in `honba-kokoro-ledger`.
  - Intraday positions (MIS) must be squared off before the exchange cut-off time (15:20 IST).
* **Settlement Day ($T+1$)**:
  - Payout of funds and demat securities occurs.
  - Delivery shares are credited to the demat account (NSDL / CDSL).
  - Withheld sale proceeds from $T$ become fully available for unencumbered cash withdrawals.

`honba-kokoro-portfolio` tracks the settlement status of every executed lot:

```rust
pub enum SettlementStatus {
    /// Executed today, awaiting T+1 clearing
    PendingSettlement {
        settlement_date: NaiveDate,
    },
    /// Cleared and available in Demat / Ledger
    Settled,
}
```
