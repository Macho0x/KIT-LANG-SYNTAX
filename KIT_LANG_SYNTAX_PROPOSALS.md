# Kit Lang Advanced Syntax Design
## Computation Expressions, Native Decimals, and Units of Measure

This document proposes four language extensions for Kit Lang that would make it the definitive language for trading bots and financial systems. Each section includes:

- A synthesized analysis of the design space
- Original-language examples (F#, OCaml, Haskell)
- Native Kit Lang implementation proposals
- Module designs (`kit-computation`, `kit-decimal`, `kit-units`, `kit-crypto`)
- Trading bot examples showing before/after

---

## 1. Computation Expressions in Kit Lang

### 1.1 Summary

**Core question:** Would F#-style computation expressions (CEs) negate Kit's existing design (effect system, actors, pipelines, `guard`), or can they complement it?

**Verdict: CEs complement Kit perfectly — if designed as *effect-aware* syntactic sugar rather than copied verbatim from F#.**

#### Key Tensions

| Tension | Resolution |
|---|---|
| CE sugar vs. explicit effect tracking | No conflict if CEs are a syntactic layer *on top* of effects, not a replacement. |
| CE blocks vs. actor-model concurrency | CEs handle sequential composition *inside* actors. Actors remain the concurrency boundary. |
| `guard` vs. `let!`/`return` | `guard` inside a CE block becomes the builder's short-circuit path. They unify. |

#### Killer Synergy: Kit Can Do CEs Better Than F#

F# requires you to manually choose the builder: `async { }`, `option { }`, `result { }`. Kit's effect system can **infer** it.

Because Kit already knows what effects a function performs, a CE block could auto-select its binding strategy:

```kit
// Hypothetical: Kit infers the handler from effects
with {
  let data = fetch(url)        // Network effect detected
  let parsed = parse(data)     // Pure
  return parsed                // Compiler infers 'with async' handling
}
```

This is the **Koka/Lean 4 model**: the effect system tracks *what* happens, and the CE/handler determines *how* it's sequenced.

#### Recommended Design

Kit should add **effect-scoped computation expressions** using `with` blocks:

```kit
with async {
  let x = fetch(url)           // plain 'let' — monadic bind inferred from 'with'
  let pure y = 2 + 2           // opt-out: strict/non-monadic when needed
  guard x.status == 200 else Error("fetch failed")
  return x.body
}
```

**Why not copy F#?** F#'s `let!`/`do!`/`yield!` menagerie exists because F# has no effect system. Kit can be simpler: plain `let` inside `with`, with the block scope determining binding strategy.

---

### 1.2 Original Language: F# Computation Expressions

F# lets you define **custom control-flow syntax** via *computation expressions* (CEs). A strategy is fundamentally a chain of fallible, async, stateful operations.

#### The `async` CE

```fsharp
let placeLimitOrderAsync symbol price qty = async {
    let! tick = fetchPriceAsync symbol        // let! = async bind
    let! account = getAccountAsync ()
    if account.BuyingPower < price * qty then
        return Error InsufficientFunds
    else
        let order = { Symbol = symbol; Price = price; Qty = qty }
        let! submission = submitOrderAsync order
        return Ok submission
}
```

#### The `maybe` CE

```fsharp
let calculatePositionSize signal = maybe {
    let! entry = signal.EntryPrice
    let! stop = signal.StopLoss
    let! risk = portfolio.RiskPerTrade
    let riskAmount = portfolio.Value * risk
    let distance = entry - stop
    if distance <= 0.0M then return None
    let qty = riskAmount / distance
    return Some qty
}
```

#### A Custom `trading` CE

```fsharp
type TradingBuilder() =
    member _.Bind(result, f) = Result.bind f result
    member _.Return(x) = Ok x
    member _.ReturnFrom(x) = x

let trading = TradingBuilder()

let executeMomentumStrategy symbol = trading {
    let! price = fetchPrice symbol |> Result.requireSome PriceUnavailable
    let! sma20 = getSMA symbol 20 |> Result.requireSome IndicatorError
    let! position = getCurrentPosition symbol
    
    if price > sma20 && position = Flat then
        let! order = placeOrder (MarketBuy 100) |> Result.mapError OrderError
        let! fill = awaitFill order.Id (TimeSpan.FromSeconds 30)
        logFill fill
        return fill
    else
        return NoOp
}
```

**What makes this powerful:**
- `let!` automatically propagates errors (`Result.bind`)
- The logic reads top-to-bottom like a strategy spec
- No `match` noise, no `if err != nil` clutter

---

### 1.3 Kit Lang Native Design

Kit has effects, `guard`, `match`, and pipelines — but no `let!`-style syntactic sugar for monadic chains.

#### Current Kit (Without CEs)

```kit
// Pyramid of doom with Result.and-then
placeLimitOrder = fn(symbol, price, qty) =>
  fetchPrice(symbol)
  |> Result.and-then fn(tick) =>
    getAccount()
    |> Result.and-then fn(account) =>
      if account.buyingPower < price * qty then
        Error InsufficientFunds
      else
        submitOrder({symbol: symbol, price: price, qty: qty})
```

```kit
// Nested match explosion
executeMomentumStrategy = fn(symbol) =>
  match fetchPrice(symbol)
  | Error e -> Error PriceUnavailable
  | Ok price ->
    match getSMA(symbol, 20)
    | Error e -> Error IndicatorError
    | Ok sma20 ->
      match getCurrentPosition(symbol)
      | Error e -> Error PositionError
      | Ok position ->
        if price > sma20 && position == Flat then
          match placeOrder(MarketBuy(100))
          | Error e -> Error OrderError
          | Ok order ->
            match awaitFill(order.id, Seconds(30))
            | Error e -> Error FillTimeout
            | Ok fill ->
              logFill(fill)
              Ok fill
        else
          Ok NoOp
```

#### Proposed Kit with `with` Blocks

```kit
// Core syntax: 'with' declares an effect-handler scope
with result {
  let tick = fetchPrice(symbol)
  let account = getAccount()
  guard account.buyingPower >= price * qty else Error InsufficientFunds
  let submission = submitOrder({symbol: symbol, price: price, qty: qty})
  return submission
}
```

```kit
// Trading strategy using effect-aware CE
executeMomentumStrategy = fn(symbol) => with result {
  let price = fetchPrice(symbol)
  let sma20 = getSMA(symbol, 20)
  let position = getCurrentPosition(symbol)
  
  guard price > sma20 && position == Flat else Ok NoOp
  
  let order = placeOrder(MarketBuy(100))
  let fill = awaitFill(order.id, Seconds(30))
  
  logFill(fill)
  return fill
}
```

**Design choices:**
- **`with handler { }`** instead of bare `async { }` — signals effect handling.
- **Plain `let` inside `with`** — monadic bind inferred. No `let!` confusion.
- **`let pure x = ...`** — opt-out for strict binds when needed.
- **`guard` integrates naturally** — short-circuits through the builder.
- **Effect inference** — `with { ... }` without handler lets the compiler infer it.

#### Actor Integration

```kit
actor TradingStrategy {
  on Tick(price) {
    with result {
      let pos = getPosition()
      let signal = generateSignal(price)
      guard pos == Flat else Error("Already positioned")
      let order = placeOrder(signal)
      return order
    }
  }
}
```

#### Arbitrage with `and!` (Applicative Parallelism)

```kit
arbitrage = fn(opp) => with async {
  let bookA = fetchOrderBook(opp.exchangeA)
  and! bookB = fetchOrderBook(opp.exchangeB)
  and! balanceA = getBalance(opp.exchangeA, "USDT")
  and! balanceB = getBalance(opp.exchangeB, "USDT")
  
  let spread = calculateSpread(bookA, bookB)
  guard spread > opp.minSpread else Error SpreadTooSmall
  
  let buyOrder = submit(bookA.bestAsk)
  and! sellOrder = submit(bookB.bestBid)
  return {buy: buyOrder, sell: sellOrder}
}
```

---

### 1.4 Error Context Propagation in CEs

A `with result { }` block hides the failure site. When `fetchPrice(symbol)` returns `Err`, the CE short-circuits and returns that `Err` from the entire block. Without additional machinery, the caller knows *that* the block failed but not *which line* produced it.

#### The Problem

Explicit chains make the failure site visible:

```kit
fetchPrice(symbol)
|> Result.and-then fn(tick) => ...   # Err here is visible
|> Result.and-then fn(account) => ... # Err here is visible
```

Inside a CE, every `let` is an implicit bind. The `Err` propagates silently:

```kit
with result {
  let tick = fetchPrice(symbol)      # Err here short-circuits block
  let account = getAccount()         # never reached
  return submission
}
```

#### How Other Languages Handle It

| Language | Mechanism | Trade-off |
|----------|-----------|-----------|
| **F#** | `Result.mapError` at each step | Verbose; defeats CE sugar |
| **Rust** | `?` + `anyhow::Context` | `.with_context(|| "fetching price")` — nice but opt-in |
| **Haskell** | `ExceptT` + `MonadError` | No automatic stack trace; use `HasCallStack` |

#### What Kit Could Do Better

Kit's effect system gives it three opportunities to fix this:

**1. Source-Location Injection (Debug Builds)**

The compiler knows the file and line of every `let`. In debug builds, the `result` builder could implicitly wrap each bind with location metadata:

```kit
with result {
  let tick = fetchPrice(symbol)   # implicit: Result.mapError (fn(e) => Err {source: "line 42", cause: e})
}
```

Zero cost in release — the optimizer strips it when the effect set does not include `Trace`.

**2. Effect-Scoped Tracing**

A `with trace result { ... }` block could instruct the builder to accumulate a context stack as an effect:

```kit
with trace result {
  let tick = fetchPrice(symbol)       # tick : Result Price (Trace "fetchPrice")
  let account = getAccount()          # account : Result Account (Trace "getAccount" :: prev)
}
# On Err: Trace ["getAccount", "fetchPrice"] — a reversed call stack
```

**3. Explicit Context via `let?`**

Introduce `let?` as an opt-in labeled bind that adds a context label without leaving the CE:

```kit
with result {
  let? tick = "fetching price" = fetchPrice(symbol)
  let? account = "loading account" = getAccount()
}
# Err payload includes: ["loading account", "fetching price"]
```

This is lighter than `Result.mapError` but still explicit about intent.

#### Recommended Design

The `result` builder should support **two modes**:

- **Release mode**: plain `Result` semantics, no overhead.
- **Debug/trace mode**: automatic source-location wrapping or effect-scoped trace accumulation.

Programmers who need custom context labels can fall back to `Result.mapError` inside the block, or use a future `let?` syntax once stabilized.

---

### 1.5 Module Design: `kit-computation`

While core `with` syntax should be built into the compiler, user-defined CE builders live in a module.

```kit
// kit-computation/Builder.kit

module Computation.Builder where

// Type class for monadic bind/return
typeclass Monad m where
  bind : (a -> m b) -> m a -> m b
  return : a -> m a
  fail : String -> m a

// The CE builder protocol
type Builder m = Builder
  { bind : (a -> m b) -> m a -> m b
  , return : a -> m a
  , zero : m a          // for guard/failure
  , combine : m a -> m a -> m a
  , delay : (() -> m a) -> m a
  }

// Built-in builders
resultBuilder : Builder Result
resultBuilder = Builder
  { bind = Result.and-then
  , return = Ok
  , zero = Error("Zero")
  , combine = fn(x, y) => x  // left-biased
  , delay = fn(f) => f()
  }

optionBuilder : Builder Option
optionBuilder = Builder
  { bind = Option.and-then
  , return = Some
  , zero = None
  , combine = fn(x, y) => x
  , delay = fn(f) => f()
  }

// Usage: register a builder for 'with' syntax
// This would be compiler-magic in practice, but conceptually:
// with result { ... }  ===  runWithBuilder resultBuilder { ... }
```

---

## 2. Native Decimal Literals in Kit Lang

### 2.1 Summary

**Core question:** How should Kit Lang implement native decimal/fixed-point numeric literals for financial computing, drawing inspiration from OCaml's Zarith?

**Verdict: Kit needs a `d` suffix literal (`100.00d`) backed by arbitrary-precision `Decimal` and compile-time-scaled `Fixed(n)`.**

#### Why This Matters

IEEE 754 binary floats cannot represent `0.1` exactly. After hundreds of trades, accumulated drift makes `cash` wrong:

```kit
// Float: DANGEROUS
0.1 + 0.2 = 0.30000000000000004   // not 0.3
10000.00 - (4500.50 * 0.1)        // silently wrong after many ops
```

Every language that defaults `0.1` to binary float creates this footgun. F# succeeds because `0.1m` is a distinct, first-class literal.

#### OCaml Zarith: The Architectural Model

Zarith provides two layers:
- **`Z`** — Arbitrary-precision integers. Small integers are unboxed native ints; big integers transparently delegate to GMP.
- **`Q`** — Rationals `{num: D.t; den: D.t}`. Always normalized (coprime, positive denominator).

**What Kit should copy:** The fast-path optimization (small values use inline arithmetic) and the layered architecture (big-integers to decimals).

**What Kit should avoid:** Zarith has no literal syntax, no fixed-scale type, and no rounding modes — all essential for finance.

#### Mapping Zarith to Kit

Zarith's two-layer architecture does not map one-to-one to Kit's three types. Here is the honest correspondence:

| Zarith | Kit Type | Literal | What It Actually Is | Maps Cleanly? |
|--------|----------|---------|---------------------|---------------|
| **`Z`** | **`BigInt`** | `100n` | Arbitrary-precision integer. Fast path for small values (i64), GMP for big. | ✓ **Direct** |
| **`Q`** | **`Decimal`** | `100.00d` | Base-10 fixed-point `{coeff: BigInt, scale: Int}`. NOT a rational `num/den`. | ⚠️ **Replaces** — same "exact math" intent, different representation |
| *(not in Zarith)* | **`Fixed(n)`** | `100.00f2` | Compile-time-scaled `i64`. Zero allocation. | ✓ **New addition** — fills a gap Zarith does not address |

**Key distinction:** Zarith `Q` stores `0.1` as the rational `1/10` — mathematically exact but unbounded. Kit `Decimal` stores `0.1` as `coeff=1, scale=1` — exact *and* fixed-scale, which is what finance needs. `Fixed(n)` is the same semantic idea as `Decimal` but backed by `i64` for speed.

#### Recommended Design

| Literal | Type | Meaning |
|---------|------|---------|
| `100` | `Int` | Native integer |
| `100.0` | `Float` | IEEE 754 binary float |
| `100.00d` | `Decimal` | Arbitrary-precision base-10 decimal |
| `100.00f4` | `Fixed(4)` | Fixed-point with 4 decimal places (i64 under the hood) |
| `100n` | `BigInt` | Arbitrary-precision integer |

**Default behavior:** `100.00` stays `Float`. Financial programmers explicitly opt into safety with `d`. This follows F#'s proven precedent (`100.0` = float, `100.0m` = decimal).

**A module-level pragma could flip the default:**

```kit
{-# default-decimal #-}
// In this module, 100.00 is inferred as Decimal
```

---

### 2.2 Original Language: OCaml Zarith

#### Z — Arbitrary-Precision Integers

```ocaml
(* Small-integer fast path + GMP big-integer slow path *)
let add x y =
  if is_small_int x && is_small_int y then begin
    let z = unsafe_to_int x + unsafe_to_int y in
    if (z lxor unsafe_to_int x) land (z lxor unsafe_to_int y) >= 0
    then of_int z
    else c_add x y  (* overflow -> GMP *)
  end else
    c_add x y

(* Operator overloading within module scope *)
let (+) = add
let (-) = sub
let ( * ) = mul

(* Usage with local open *)
D.(~$2 + ~$5 * ~$10)
```

#### Q — Rationals

```ocaml
type t = { num: D.t; den: D.t }

(* Construction normalizes via GCD *)
let make_real n d =
  if n == D.zero || d == D.one then mk n D.one
  else
    let g = D.gcd n d in
    if g == D.one then mk n d
    else mk (D.divexact n g) (D.divexact d g)

(* Arithmetic *)
let add = aors D.add
let sub = aors D.sub
```

**Key insight:** Zarith stores `0.1` as the rational `1/10`. This is mathematically exact but overkill for finance, which needs fixed-scale decimal (e.g., 2 decimal places for USD).

---

### 2.3 Kit Lang Native Design

#### Literal Syntax

```kit
// Unambiguous literals
100          // Int
100.0        // Float
100.00d      // Decimal
100.00f2     // Fixed(2) — 2 decimal places, stored as i64
100n         // BigInt

// Type inference from context
let total = 100.00d + 50.00d    // Decimal
let total = 100.00d + 50.0      // TYPE ERROR: Decimal != Float
```

#### Why `d` suffix?

- F# uses `m` for `decimal` — but `m` suggests "money," which is too narrow.
- `d` means "decimal" (base-10), not "double."
- `100.00d` reads as "one hundred decimal" — self-documenting.

#### Arithmetic & Rounding

```kit
// Standard operators dispatch via Numeric type class
(+) : Numeric a => a -> a -> a
(-) : Numeric a => a -> a -> a
(*) : Numeric a => a -> a -> a
(/) : Numeric a => a -> a -> a

// Division semantics differ by type
10 / 3       : Int       -> 3 (truncation)
10.0 / 3.0   : Float     -> 3.333... (IEEE)
10.0d / 3.0d : Decimal   -> 3.333... (exact, configurable precision)
10.00f2 / 3.00f2 : Fixed(2) -> 3.33 (rounded to scale)

// Decimal-specific operations
Decimal.div : RoundingMode -> Int -> Decimal -> Decimal -> Decimal
Decimal.round : RoundingMode -> Int -> Decimal -> Decimal
```

**Rounding modes (essential for finance):**

```kit
type RoundingMode
  = RoundHalfEven     // Banker's rounding (default for USD)
  | RoundHalfUp       // Traditional
  | RoundHalfDown
  | RoundTowardZero   // Truncation
  | RoundTowardPosInf // Ceiling
  | RoundTowardNegInf // Floor
```

---

### 2.4 Module Design: `kit-decimal`

#### Internal Representation

A `Decimal` is stored as a signed coefficient (`BigInt`) and a scale (exponent):

```kit
// Decimal = coefficient * 10^(-scale)
// 100.00d is stored as coeff=10000, scale=2

type Decimal = Decimal
  { coeff : BigInt    // from kit-bigint
  , scale : Int
  }
```

#### API Sketch

```kit
module Decimal

import BigInt as B

// -- Types ------------------------------------------------

type Decimal = { coeff: B.BigInt, scale: Int }

type RoundingMode
  = RoundHalfEven | RoundHalfUp | RoundHalfDown
  | RoundTowardZero | RoundTowardPosInf | RoundTowardNegInf

// -- Construction ------------------------------------------

zero : Decimal
one  : Decimal
ofInt : Int -> Decimal
ofBigInt : B.BigInt -> Decimal
ofString : String -> Result Decimal ParseError
ofCents : Int -> Decimal      // 100 -> 1.00d (scale=2)

// -- Arithmetic --------------------------------------------

add : Decimal -> Decimal -> Decimal
sub : Decimal -> Decimal -> Decimal
mul : Decimal -> Decimal -> Decimal
div : RoundingMode -> Int -> Decimal -> Decimal -> Decimal
neg : Decimal -> Decimal
abs : Decimal -> Decimal

// -- Scale operations --------------------------------------

scale : Decimal -> Int
rescale : RoundingMode -> Int -> Decimal -> Decimal
quantize : RoundingMode -> Decimal -> Decimal -> Decimal

// -- Comparison --------------------------------------------

compare : Decimal -> Decimal -> Ordering
equal : Decimal -> Decimal -> Bool
lt : Decimal -> Decimal -> Bool
gt : Decimal -> Decimal -> Bool

// -- Conversion --------------------------------------------

toInt : Decimal -> Result Int Overflow
toFloat : Decimal -> Float        // may lose precision
toString : Decimal -> String

// -- Operators (module-scoped) -----------------------------

(+) = add
(-) = sub
(*) = mul
```

#### Fixed-Point Type: `Fixed(n)`

For HFT hot paths where zero allocation matters:

```kit
module Fixed

// Fixed(n) stores values as integers scaled by 10^n
// 100.00f2 is stored as the i64 10000

type Fixed : Int -> Type  // type-level scale parameter

ofInt : Int -> Fixed n
ofDecimal : Decimal -> Fixed n  // rounds to scale n

// Arithmetic: same scale required; type-checked at compile time
add : Fixed n -> Fixed n -> Fixed n
sub : Fixed n -> Fixed n -> Fixed n
mul : Fixed n -> Fixed m -> Fixed (n + m)
div : Fixed n -> Fixed m -> Fixed (n - m)

toDecimal : Fixed n -> Decimal
```

**`Decimal` vs `Fixed(n)` — same semantics, different performance profiles:**

| | `Decimal` | `Fixed(n)` |
|--|-----------|------------|
| **Backing** | `{coeff: BigInt, scale: Int}` | `i64` |
| **Overflow** | Never (arbitrary precision) | **Yes** — fixed 64-bit range |
| **Allocation** | Heap (BigInt) | Zero allocation |
| **Use case** | Ledgers, P&L, settlement | Matching engines, market data |
| **Scale** | Dynamic (runtime) | Static (compile-time) |

Choose `Decimal` when correctness matters more than speed. Choose `Fixed(n)` when you need nanosecond-level arithmetic and can prove values stay within `i64` bounds.

#### Relationship to `BigInt`

`Decimal` is **not** a layer on top of Zarith's `Q` (rationals). It is a separate type that borrows only the fast-path idea from Zarith's `Z`:

- **`BigInt`** (stdlib or `kit-bigint` package) provides arbitrary-precision integers. `Decimal` depends on it for the `coeff` field.
- **`Decimal`** (`kit-decimal`) adds a `scale` field to `BigInt`, making it a base-10 fixed-point number rather than a rational.
- **`Fixed(n)`** is a completely separate type backed by `i64`, not `BigInt`.

**Fast path**: If a `Decimal` coefficient fits in `i64`, the implementation may use inline integer arithmetic and skip heap allocation. This optimization is inspired by Zarith but is an internal implementation detail.

**Normalization**: After arithmetic, remove trailing zeros to minimize coefficient size.

```kit
// Internal: normalize by removing trailing decimal zeros
normalize : Decimal -> Decimal = fn({ coeff, scale }) =>
  if scale == 0 then { coeff, scale }
  else if B.rem coeff (B.ofInt 10) == B.zero then
    normalize { coeff = B.div coeff (B.ofInt 10), scale = scale - 1 }
  else { coeff, scale }
```

---

### 2.5 Trading Bot Examples

#### Before: Dangerous Floats

```kit
// DANGEROUS: Float accumulates error
module TradingBot where

type Price = Float
type Quantity = Float
type Cash = Float

calculatePnl : Price -> Price -> Quantity -> Cash
calculatePnl entryPrice currentPrice quantity =
  (currentPrice - entryPrice) * quantity

// BUG: After 1000 trades, cash drifts
// 0.1 + 0.2 = 0.30000000000000004
```

#### After: Safe Decimals

```kit
// SAFE: Exact decimal arithmetic
module TradingBot

import Decimal as D

type Price = D.Decimal
type Quantity = D.Decimal
type Cash = D.Decimal

calculatePnl : Price -> Price -> Quantity -> Cash = fn(entryPrice, currentPrice, quantity) =>
  D.mul (D.sub currentPrice entryPrice) quantity
// 0.1d + 0.2d = 0.3d exactly

// -- Full strategy with decimal safety ---------------------

sma : List Price -> Option Price = fn(prices) =>
  case prices of
  | [] | [_] -> None
  | _ ->
    let sum = List.foldl D.add D.zero prices
    let count = D.fromInt (length prices)
    Some (D.div RoundHalfEven 2 sum count)

// -- HFT hot path with Fixed -------------------------------

import Fixed as F

type OrderPrice = Fixed 2   // USD with 2 decimal places
type OrderQty = Fixed 8     // Crypto with 8 decimal places

matchOrder : OrderPrice -> OrderQty -> OrderPrice -> OrderQty -> (OrderPrice, OrderQty)
matchOrder bidPrice bidQty askPrice askQty =
  let fillQty = F.min bidQty askQty
  let fillPrice = askPrice
  (fillPrice, fillQty)
  // Integer arithmetic under the hood. Zero allocation.
```

---

## 3. Units of Measure in Kit Lang

### 3.1 Summary

**Core question:** How can Kit Lang implement F#-style units of measure for compile-time dimensional analysis?

**Verdict: Kit should implement units of measure as a compiler-supported type system feature backed by phantom types, not just a library.**

#### Why Units of Measure Matter

Consider this real trading bug:

```kit
// Without units: price and qty are both Decimal
let notional = price + quantity   // Oops! Added price to quantity
// Compiles fine. Loses money at runtime.
```

With units:

```kit
// With units: compile-time error
let notional : Decimal<USD> = price + quantity
// TYPE ERROR: Cannot add Decimal<USD> to Decimal<shares>
```

Other bugs prevented:
- Currency mismatches: USD vs USDC vs USDT
- Basis point vs percentage confusion (`0.01` = 1% or 1bp?)
- Price vs notional vs margin confusion

#### Design Philosophy

Kit has two features that make units of measure especially powerful:

1. **Refinement types** already exist: `{x: Int | x > 0}`. A unit system is a natural extension.
2. **Effect system** tracks *what* code does. Units track *what data represents*. They are philosophically aligned.

#### Recommended Design

Units should be **first-class in the compiler**, not a library, for three reasons:

1. **Type inference**: When `price : Decimal<USD>` and `qty : Decimal<shares>`, the compiler must infer `price * qty : Decimal<USD*shares>`. This requires type-level arithmetic that HM inference cannot express via libraries without significant syntactic pain.
2. **Zero runtime cost**: Units are erased at compile time. A library solution would require newtype wrappers, which affect interoperability and add syntactic noise.
3. **Literal syntax**: `100.00d<USD>` must be parsed natively.

**Library fallback**: If compiler support is deferred, a `kit-units` library using phantom types + macros can provide 80% of the value.

---

### 3.2 Original Language: F# Units of Measure

F# has the most powerful units-of-measure system of any production language. It is erased at compile time — zero runtime overhead.

#### Basic Units

```fsharp
[<Measure>] type USD
[<Measure>] type BTC
[<Measure>] type shares

let price = 100.0m<USD>           // decimal<USD>
let qty = 50.0m<shares>           // decimal<shares>
let notional = price * qty        // decimal<USD * shares>
// notional has type decimal<USD * shares> — a new derived unit
```

#### Derived Units

```fsharp
[<Measure>] type USD
[<Measure>] type BTC
[<Measure>] type hour

// Derived units are automatic
let rate = 50000.0m<USD/BTC>      // exchange rate
let value = rate * 2.0m<BTC>      // result: decimal<USD>

// Generic functions
let convert<'a, 'b> (rate: decimal<'a/'b>) (amount: decimal<'b>) : decimal<'a> =
    rate * amount
```

#### Dimensional Analysis

```fsharp
[<Measure>] type m
[<Measure>] type s

let speed = 10.0<m/s>
let time = 5.0<s>
let distance = speed * time       // m/s * s = m
```

#### Limitations

- Units are erased — no runtime conversion factors.
- Cannot express relationships like `1 USD = 100 cents` at the type level.
- No type-level exponentiation (e.g., m^2 for area is supported, m^3 for volume too, but not general exponents).

---

### 3.3 Kit Lang Native Design

#### Compiler-Supported Syntax (Recommended)

```kit
// -- Unit declarations -------------------------------------

unit USD
unit BTC
unit shares
unit cents = USD / 100      // 1 USD = 100 cents
unit basisPoint = 1 / 10000 // 1% = 100 basis points

// -- Usage in types ----------------------------------------

let price : Decimal<USD> = 100.00d<USD>
let qty : Decimal<shares> = 50.0d<shares>
let notional = price * qty
// inferred: notional : Decimal<USD * shares>

// -- Generic unit functions --------------------------------

convert : Decimal<a / b> -> Decimal<b> -> Decimal<a>
convert rate amount = rate * amount

// -- Derived units are automatic ---------------------------

let rate : Decimal<USD / BTC> = 45000.00d<USD / BTC>
let btcQty : Decimal<BTC> = 0.5d<BTC>
let usdValue = rate * btcQty
// inferred: usdValue : Decimal<USD>

// -- Division inverts units --------------------------------

let notional : Decimal<USD> = 5000.00d<USD>
let fillQty : Decimal<shares> = 100.0d<shares>
let avgPrice = notional / fillQty
// inferred: avgPrice : Decimal<USD / shares>

// -- Compile-time errors -----------------------------------

let bad = price + qty
// TYPE ERROR: Cannot add Decimal<USD> and Decimal<shares>

let alsoBad : Decimal<USD> = price * rate
// TYPE ERROR: USD * (USD/BTC) = USD^2/BTC, not USD
```

#### Unit Conversion

Since units are erased, conversions must be explicit:

```kit
// Runtime conversion function
USD.toCents : Decimal<USD> -> Decimal<cents>
USD.toCents usd = usd * 100.0d<cents / USD>

// Or via a conversion trait
typeclass Convertible from to where
  convert : Decimal<from> -> Decimal<to>

instance Convertible USD cents where
  convert usd = usd * 100.0d<cents / USD>
```

#### Refinement Types + Units

Kit's existing refinement types compose beautifully:

```kit
// Price must be positive AND in USD
type Price = {x: Decimal<USD> | x > 0.00d<USD>}

// Quantity must be non-negative
type Quantity = {x: Decimal<shares> | x >= 0.0d<shares>}

placeOrder : Price -> Quantity -> Result Order OrderError
placeOrder price qty =
  guard price > 0.00d<USD> else Error InvalidPrice
  guard qty >= 0.0d<shares> else Error InvalidQuantity
  Ok (Order { price, qty })
```

---

### 3.4 Module Design: `kit-units`

If full compiler support is deferred, a library can provide phantom-type units.

```kit
// kit-units/Unit.kit

module Units where

// Phantom type for units
type Unit : Type -> Type = Unit  // phantom: no runtime representation

// Numeric with phantom unit
type Numeric u a = Numeric a   // a is the underlying numeric type

// Constructors
ofInt : Int -> Numeric u Int
ofInt x = Numeric x

ofDecimal : Decimal -> Numeric u Decimal
ofDecimal x = Numeric x

// Extraction
value : Numeric u a -> a
value (Numeric x) = x

// -- Unit arithmetic (type-level only) ---------------------

// These are type-level operations, not value-level.
// In a library implementation, they require macros or
// extensive boilerplate.

// With compiler support, these are automatic:
// Numeric<USD> * Numeric<shares> = Numeric<USD * shares>
// Numeric<USD> / Numeric<shares> = Numeric<USD / shares>

// Without compiler support, users must manually track units
// using phantom types and wrapper functions:

mul : Numeric u1 a -> Numeric u2 a -> Numeric (u1 * u2) a
mul (Numeric x) (Numeric y) = Numeric (x * y)

div : Numeric u1 a -> Numeric u2 a -> Numeric (u1 / u2) a
div (Numeric x) (Numeric y) = Numeric (x * y)

add : Numeric u a -> Numeric u a -> Numeric u a
add (Numeric x) (Numeric y) = Numeric (x + y)
```

#### Library Usage (Less Ergonomic)

```kit
import Units as U

type USD = {}
type shares = {}

let price = U.ofDecimal 100.00d :: Numeric USD Decimal
let qty = U.ofDecimal 50.0d :: Numeric shares Decimal

let notional = U.mul price qty
// type: Numeric (USD * shares) Decimal

// Verbose compared to compiler-native:
// let notional = price * qty   -- would be automatic with compiler support
```

---

### 3.5 Trading Bot Examples

#### Without Units (Bug-Prone)

```kit
// All Decimals look the same
type Order = Order
  { price : Decimal
  , quantity : Decimal
  , fee : Decimal
  , notional : Decimal
  }

// Easy to mix up:
let order = Order
  { price = 100.00d
  , quantity = 50.0d
  , fee = 2.5d
  , notional = 50.0d          // Oops! Should be price * qty
  }

// Even worse: adding incompatible values
let totalCost = order.price + order.fee
// What if fee was in basis points? Catastrophe.
```

#### With Units (Compile-Time Safety)

```kit
// Each value carries its dimension
unit USD
unit shares
unit basisPoint = 1 / 10000

type Order = Order
  { price : Decimal<USD / shares>
  , quantity : Decimal<shares>
  , feeRate : Decimal<basisPoint>
  , notional : Decimal<USD>
  }

// Correct by construction
let order = Order
  { price = 100.00d<USD / shares>
  , quantity = 50.0d<shares>
  , feeRate = 25.0d<basisPoint>     // 25 bp = 0.25%
  , notional = 100.00d<USD / shares> * 50.0d<shares>  // = 5000.00d<USD>
  }

// The type system prevents nonsense
let bad = order.price + order.quantity
// TYPE ERROR: Cannot add Decimal<USD/shares> and Decimal<shares>

let fee = order.notional * order.feeRate
// fee : Decimal<USD * basisPoint>
// Need to convert: feeInUSD = fee / 10000.0d<basisPoint / 1>
```

#### Multi-Currency Strategy

```kit
unit USD
unit EUR
unit BTC
unit ETH

// Generic strategy that works for any currency pair
rsiStrategy : Decimal<a> -> Decimal<b> -> Decimal<a / b> -> Signal
rsiStrategy basePrice quotePrice rate =
  let priceInBase = quotePrice * rate   // b * (a/b) = a
  if priceInBase < basePrice * 0.95d<1> then Buy
  else if priceInBase > basePrice * 1.05d<1> then Sell
  else Hold

// Usage
btcSignal = rsiStrategy 45000.00d<USD> 0.5d<BTC> 45000.00d<USD / BTC>
ethSignal = rsiStrategy 3000.00d<USD> 10.0d<ETH> 3000.00d<USD / ETH>
```

---

### 3.6 Precedent Analysis

| Language | Approach | Strengths | Weaknesses |
|---|---|---|---|
| **F#** | Built-in compiler feature, erased | Zero-cost, excellent inference, derived units | No runtime conversion, limited exponents |
| **Haskell** | `dimensional` library | Type-level units, many physical units | Heavy syntax, lots of boilerplate, verbose error messages |
| **Rust** | `uom` crate (type-level) | Zero-cost, many units | Complex trait bounds, long compile times, steep learning curve |
| **C++** | `Boost.Units` | Powerful, supports many dimensions | Template metaprogramming hell, unreadable errors |
| **Python** | `pint` library | Runtime units, conversions | Not compile-time, slow, errors at runtime |

**What Kit should learn from F#:**
- Units are **erased** — no runtime cost.
- **Inference** is automatic — `price * qty` derives the unit.
- **Derived units** happen automatically — `USD/BTC` from division.

**What Kit should improve over F#:**
- **Unit aliases with conversion factors**: `unit cents = USD / 100` gives both the unit and the conversion ratio.
- **Composition with refinement types**: `{x: Decimal<USD> | x > 0}` combines dimensional analysis with value constraints.
- **First-class in a systems language**: F# is .NET-managed. Kit compiles to native via Zig, so unit-erased code is truly zero-cost.

---

## 4. Crypto Primitives for Kit Lang

### 4.1 Summary

**Core question:** How should Kit Lang provide foundational cryptographic primitives—specifically Keccak-256, secp256k1, and RIPEMD-160—needed for blockchain trading bots and secure financial systems?

**Verdict: Kit should ship a `kit-crypto` module that exposes type-safe, zero-cost wrappers around battle-tested implementations (libsecp256k1, tiny-keccak, etc.), not reinvent low-level cryptography.**

#### Why This Matters

Trading bots interacting with Ethereum or Bitcoin need:
- **Keccak-256** for hashing transactions and deriving addresses.
- **secp256k1** for signing transactions and verifying signatures.
- **RIPEMD-160** as part of Bitcoin address generation (`RIPEMD-160(SHA-256(pubkey))`).

Rolling custom crypto is a security liability. Kit should wrap optimized C/Zig libraries and expose them through strongly typed Kit APIs.

#### Recommended Design

| Primitive | Source Library | Kit Wrapper |
|-----------|---------------|-------------|
| Keccak-256 | tiny-keccak / Zig impl | `Hash.keccak256` |
| RIPEMD-160 | OpenSSL / compact C | `Hash.ripemd160` |
| secp256k1 | libsecp256k1 | `Secp256k1.sign`, `verify`, `recover` |

All hash outputs should be typed as fixed-size byte arrays (`Bytes32`, `Bytes20`) to prevent mixing up digests.

---

### 4.2 Original Language: Existing Cryptographic Libraries

#### libsecp256k1 (C)

The Bitcoin Core library for secp256k1 elliptic curve operations.

```c
#include <secp256k1.h>

secp256k1_context *ctx = secp256k1_context_create(SECP256K1_CONTEXT_SIGN);

unsigned char msg[32];
unsigned char seckey[32];
unsigned char signature[64];

// Sign
secp256k1_ecdsa_sign(ctx, &sig, msg, seckey, NULL, NULL);
```

#### tiny-keccak (Rust)

Pure Rust implementation of Keccak and SHA-3.

```rust
use tiny_keccak::{Keccak, Hasher};

let mut keccak = Keccak::v256();
keccak.update(b"hello");
let mut output = [0u8; 32];
keccak.finalize(&mut output);
```

**Key insight:** These libraries are fast and audited. Kit should FFI-bind or port the Zig equivalents rather than write from scratch.

---

### 4.3 Kit Lang Native Design

#### Typed Byte Arrays

```kit
type Bytes20 = Bytes 20
type Bytes32 = Bytes 32
type Signature = Bytes 64
type PublicKey = Bytes 33   // compressed
type PrivateKey = Bytes 32
```

#### Hash Module

```kit
// kit-crypto/Hash.kit

module Crypto.Hash where

keccak256 : Bytes -> Bytes32
keccak256 data = // FFI or native impl

ripemd160 : Bytes -> Bytes20
ripemd160 data = // FFI or native impl
```

#### Secp256k1 Module

```kit
// kit-crypto/Secp256k1.kit

module Crypto.Secp256k1 where

type PrivateKey = Bytes32
type PublicKey = Bytes33
type Signature = Bytes64
type RecoveryId = Int

generateKeyPair : IO (PrivateKey, PublicKey)
sign : PrivateKey -> Bytes32 -> IO Signature
verify : PublicKey -> Bytes32 -> Signature -> Bool
recoverPublicKey : Bytes32 -> Signature -> RecoveryId -> Option PublicKey
```

---

### 4.4 Module Design: `kit-crypto`

```kit
// kit-crypto/Hash.kit
module Crypto.Hash where

type Bytes20 = Bytes 20
type Bytes32 = Bytes 32

keccak256 : Bytes -> Bytes32
keccak256 = extern "kit_crypto_keccak256"

ripemd160 : Bytes -> Bytes20
ripemd160 = extern "kit_crypto_ripemd160"
```

```kit
// kit-crypto/Secp256k1.kit
module Crypto.Secp256k1

type PrivateKey = Bytes32
type PublicKey = Bytes33
type Signature = Bytes64

# FFI bindings to optimized C/Zig implementations
extern-c sign(pk: PrivateKey, msg: Bytes32) -> Signature from "kit_crypto" link "libkit_crypto.a"
extern-c signRecoverable(pk: PrivateKey, msg: Bytes32) -> (Signature, Int) from "kit_crypto" link "libkit_crypto.a"
extern-c verify(pk: PublicKey, msg: Bytes32, sig: Signature) -> Bool from "kit_crypto" link "libkit_crypto.a"
extern-c recover(msg: Bytes32, sig: Signature, recId: Int) -> Option PublicKey from "kit_crypto" link "libkit_crypto.a"
```

---

### 4.5 Trading Bot Examples

#### Ethereum Address Derivation

```kit
import Crypto.Hash as H
import Crypto.Secp256k1 as Secp

deriveAddress : PublicKey -> Bytes20 = fn(pubkey) =>
  let hash = H.keccak256 (Secp.decompress pubkey)
  slice 12 32 hash
```

#### Signing a Raw Transaction

```kit
signTransaction : PrivateKey -> Bytes -> Signature = fn(pk, tx) =>
  let txHash = H.keccak256 tx
  Secp.sign pk txHash

# For Ethereum, you also need the recovery id (v) to verify the signer
signTransactionRecoverable : PrivateKey -> Bytes -> (Signature, Int) = fn(pk, tx) =>
  let txHash = H.keccak256 tx
  Secp.signRecoverable pk txHash
```

#### EIP-712 Typed Data Signing

EIP-712 is the standard for signing typed structured data in Ethereum wallets (MetaMask, Ledger, etc.). It prevents replay attacks and phishing by including a domain separator and type hash in the signature payload.

```kit
import Crypto.Hash as H
import Crypto.Secp256k1 as Secp

# -- EIP-712 constants --------------------------------------

EIP712_DOMAIN_TYPE : String = "EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"
ORDER_TYPE : String = "Order(address trader,uint256 amount,uint256 price,address token)"

# -- Domain separator ---------------------------------------

type EIP712Domain = {
  name: String,
  version: String,
  chainId: Int,
  verifyingContract: Bytes20
}

hashDomain : EIP712Domain -> Bytes32 = fn(domain) =>
  let typeHash = H.keccak256 (String.to-bytes EIP712_DOMAIN_TYPE)
  H.keccak256 (Bytes.concat
    typeHash
    (H.keccak256 (String.to-bytes domain.name))
    (H.keccak256 (String.to-bytes domain.version))
    (Bytes32.fromInt domain.chainId)
    (Bytes32.fromBytes20 domain.verifyingContract))

# -- Struct hash --------------------------------------------

type Order = {
  trader: Bytes20,
  amount: Int,
  price: Int,
  token: Bytes20
}

hashOrder : Order -> Bytes32 = fn(order) =>
  let typeHash = H.keccak256 (String.to-bytes ORDER_TYPE)
  H.keccak256 (Bytes.concat
    typeHash
    (Bytes32.fromBytes20 order.trader)
    (Bytes32.fromInt order.amount)
    (Bytes32.fromInt order.price)
    (Bytes32.fromBytes20 order.token))

# -- Final digest & sign ------------------------------------

# Returns the raw 64-byte signature (r || s) only
signTypedData : PrivateKey -> EIP712Domain -> Order -> Signature = fn(pk, domain, order) =>
  let domainSeparator = hashDomain domain
  let structHash = hashOrder order
  let prefix = String.to-bytes "\x19\x01"
  let digest = H.keccak256 (Bytes.concat prefix domainSeparator structHash)
  Secp.sign pk digest

# Returns the full Ethereum signature: (r || s, v) where v = 27 or 28
signTypedDataFull : PrivateKey -> EIP712Domain -> Order -> (Signature, Int) = fn(pk, domain, order) =>
  let domainSeparator = hashDomain domain
  let structHash = hashOrder order
  let prefix = String.to-bytes "\x19\x01"
  let digest = H.keccak256 (Bytes.concat prefix domainSeparator structHash)
  let (sig, recId) = Secp.signRecoverable pk digest
  let v = recId + 27   # Ethereum convention: v = 27 or 28
  (sig, v)

# Assemble the 65-byte signature expected by smart contracts and wallets
assembleEthSignature : Signature -> Int -> Bytes65 = fn(sig, v) =>
  Bytes.concat sig (Bytes1.fromInt v)
```

**What this demonstrates:**
- `keccak256` for type-hash and struct-hash computation.
- `Bytes32` typed arrays to prevent mixing up 32-byte digests with 20-byte addresses.
- Domain separation — the same `Order` struct signed on Ethereum mainnet vs. a testnet produces different signatures.
- Recovery id (`v`) — libsecp256k1 returns `recId` (0 or 1); Ethereum adds 27 to get `v` (27 or 28). Wallets and `ecrecover` in Solidity expect the 65-byte `(r, s, v)` format.
- `signRecoverable` returns both the 64-byte `(r, s)` and the recovery id, which you need to broadcast transactions or verify on-chain.

#### Bitcoin Address Generation

```kit
bitcoinAddress : PublicKey -> Bytes20 = fn(pubkey) =>
  // In practice: SHA-256 first, then RIPEMD-160
  H.ripemd160 pubkey
```

*(Note: Bitcoin uses SHA-256 then RIPEMD-160; the example focuses on the RIPEMD-160 step provided by `kit-crypto`.)*

---
