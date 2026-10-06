# Money and Currency

`Money` is an exact signed integer count of the **minor unit** of its currency,
plus the currency. Fractional input is rejected at construction, and the wire
format carries the integer, never a float. Prices remain floats (market
observations); anything settled, summed or compared is `Money`.
See ADR 0011 in the `honba` repository for the rounding rules.

## Minor units per currency

Each currency defines a `minor_exponent`: one major unit equals
`10 ** minor_exponent` minor units. Generic code and docs say "minor unit";
the local names below are used only where a currency is the subject.

| Currency | Minor unit (singular / plural) | `minor_exponent` | Minor per major |
|---|---|---|---|
| INR | paisa / paise | 2 | 100 |
| USD | cent / cents | 2 | 100 |
| EUR | cent / cents | 2 | 100 |
| GBP | penny / pence | 2 | 100 |

## Accessors

| Python | Rust | Purpose |
|---|---|---|
| `Currency.minor_exponent` | `Currency::minor_exponent()` | Decimal exponent of the minor unit |
| `Currency.minor_per_major` | n/a | `10 ** minor_exponent` |
| `Currency.minor_unit` | `Currency::minor_unit()` | Singular and plural minor-unit names |
| `Money.format_minor()` | `Money::format_minor()` | Human text such as `1,250 paise`, `1 cent`, `300 pence` |
| `honba._honba.currency_minor_units()` | n/a | Table of all currencies' minor units |

!!! warning "Deprecated"
    `MINOR_PER_MAJOR` is deprecated. It still equals 100 but emits a
    `DeprecationWarning`; use `Currency.minor_per_major` instead of a
    hard-coded 100.

The serialized `Money` schema is unchanged (`SCHEMA_VERSION` 3); its
description now reads "see Currency minor_exponent" rather than "paise/cents".
