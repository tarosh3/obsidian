---
title: "Design a Stock Exchange / Order Matching Engine"
aliases: [Order Matching Engine LLD, Limit Order Book, Price-Time Priority]
tags: [system-design, lld, case-study, go, concurrency, strategy-pattern]
status: reference-quality
---

# Design a Stock Exchange / Order Matching Engine

> [!abstract] What you'll be able to do after this chapter
> Implement price-time priority matching with correct partial fills across multiple resting orders, and explain the real data-structure tradeoff between a simple sorted-slice book and a production heap/skip-list-based one.

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Design a simplified stock exchange order matching engine — accept buy/sell limit orders for a symbol, match compatible orders using price-time priority, and execute trades."

## Step 2 — Requirements

Submit a limit buy or sell order (price + quantity). Match against the opposite side's book using **price-time priority**: best price first, and among orders tied on price, earliest submitted first — the standard real-exchange matching rule. Support **partial fills** — one incoming order may match against multiple resting orders to fill its full quantity. Cancel an unfilled or partially-filled order.

> [!example]+ 🪜 How to build this live, step by step (interview execution order, with code)
> One of the more algorithmically dense LLD questions — build correctness on the simplest possible case first, then widen.
>
> **Checkpoint 1 (~8 min) — one resting order, one incoming order, exact same price, full fill.**
> ```go
> type Order struct {
>     ID       string
>     Side     Side
>     Price    int64
>     Quantity int64
>     Filled   int64
> }
>
> // Simplest possible match: if the incoming order's price equals the
> // one resting order's price, fill it completely. No book structure yet.
> func matchSimple(incoming, resting *Order) *Trade {
>     if incoming.Price == resting.Price {
>         qty := min64(incoming.Quantity, resting.Quantity)
>         return &Trade{Price: resting.Price, Quantity: qty}
>     }
>     return nil
> }
> ```
> **Pattern used: none.** No sorting, no price levels — just prove one trade can fire correctly.
>
> **Checkpoint 2 (~8 min) — FIFO within one price (time priority).**
> ```go
> // PriceLevel — a FIFO queue at ONE price. Enforces time priority
> // among orders already tied on price.
> type PriceLevel struct {
>     Price  int64
>     Orders []*Order // index 0 = oldest, matched first
> }
> ```
> Multiple resting orders at the same price now fill in arrival order — this is a real, easy-to-get-wrong detail worth calling out explicitly.
>
> **Checkpoint 3 (~10-12 min) — sorted price levels (price priority) + the real matching loop.**
> ```go
> // bids sorted DESCENDING (best/highest first), asks ASCENDING
> // (best/lowest first). This ordering + FIFO per level IS price-time priority.
> func (b *OrderBook) matchBuy(buyOrder *Order) []Trade {
>     for buyOrder.Remaining() > 0 && len(b.asks) > 0 {
>         best := b.asks[0]
>         if best.Price > buyOrder.Price {
>             break // sorted ascending — nothing further out qualifies either
>         }
>         // drain best.Orders FIFO, computing qty = min(remaining, remaining)
>     }
> }
> ```
> **Pattern used: none (this is the core matching algorithm, not a GoF pattern)** — say this if asked, and name the sorted-slice-vs-heap simplification (Step 4) unprompted, it's a real depth signal.
>
> **Checkpoint 4 (remaining time) — partial fills across TWO separate incoming orders.** Trace the exact numeric example already in Step 4's `main.go`: a 100-share resting sell, a 60-share buy (partial fill, resting order stays with 40 remaining), then a second 40-share buy (fully drains it). This is the scenario worth demoing live, not just claiming works.
>
> **If asked, not required — concurrency, cancel, market orders.** One mutex per `OrderBook` (Step 5), `orders map[string]*Order` for O(1) cancel lookup, market orders as a different `MatchingStrategy` (Step-5 Q&A) sweeping the book instead of stopping at a limit price.
>
> **If you're short on time:** stop after Checkpoint 3 with a single price level's matching fully correct. Multi-price-level sorting is "more of the same structure" — describe it verbally if you don't reach full implementation.

## Step 3 — The bad first draft

A flat, unsorted list of all pending orders, matched by **linearly scanning** the entire list for a compatible counterparty every time a new order arrives:

```go
// The bad draft — O(n) scan per order, no natural price-time ordering
func matchOrder(incoming *Order, allOrders []*Order) {
    for _, existing := range allOrders {
        if compatible(incoming, existing) {
            // match... but nothing here guarantees BEST price gets
            // matched first, or that ties are broken by arrival time
        }
    }
}
```

Breaks at real exchange volume — an O(n) scan per incoming order is far too slow at thousands of orders/sec — and doesn't naturally enforce price-time priority without extra, error-prone sorting logic bolted onto the matching code itself.

## Step 4 — Refactor: a price-ordered book with FIFO queues per price level

> [!tip] The real, industry-standard shape — with an honest scope note
> Production matching engines typically use a heap or skip-list-based book for faster best-price lookups at very high order-arrival rates. This chapter uses two **sorted slices** (bids descending by price, asks ascending by price), each holding a FIFO queue of orders at every distinct price level. It's simpler to implement correctly and still demonstrates price-time priority precisely — the core matching *rules* are what's being taught here, not micro-optimized data-structure selection. Worth naming this simplification explicitly in an interview rather than silently glossing over it.

```go
// FILE: order.go
package exchange

type Side int

const (
    Buy Side = iota
    Sell
)

type Order struct {
    ID       string
    Side     Side
    Price    int64 // cents — avoids floating point, same discipline as Splitwise
    Quantity int64
    Filled   int64
}

func (o *Order) Remaining() int64 {
    return o.Quantity - o.Filled
}
```

```go
// FILE: trade.go
package exchange

type Trade struct {
    BuyOrderID  string
    SellOrderID string
    Price       int64
    Quantity    int64
}
```

```go
// FILE: price_level.go
package exchange

// PriceLevel holds every order resting at ONE specific price, in
// FIFO order — this is what enforces TIME priority among orders
// already tied on PRICE priority.
type PriceLevel struct {
    Price  int64
    Orders []*Order // index 0 is the oldest — matched first
}
```

```go
// FILE: order_book.go
package exchange

import "sync"

// OrderBook keeps bids sorted DESCENDING by price (best/highest bid
// first) and asks sorted ASCENDING by price (best/lowest ask first).
// That ordering, combined with FIFO within each PriceLevel, is what
// implements price-time priority — the standard real-exchange rule.
type OrderBook struct {
    mu     sync.Mutex
    bids   []*PriceLevel
    asks   []*PriceLevel
    orders map[string]*Order
}

func NewOrderBook() *OrderBook {
    return &OrderBook{orders: make(map[string]*Order)}
}

// Submit adds a new order, matches it against the opposite side as
// far as possible, then rests any unfilled remainder in the book.
func (b *OrderBook) Submit(order *Order) []Trade {
    b.mu.Lock()
    defer b.mu.Unlock()

    b.orders[order.ID] = order

    var trades []Trade
    if order.Side == Buy {
        trades = b.matchBuy(order)
        if order.Remaining() > 0 {
            b.restBid(order)
        }
    } else {
        trades = b.matchSell(order)
        if order.Remaining() > 0 {
            b.restAsk(order)
        }
    }
    return trades
}

// matchBuy fills an incoming buy against resting asks, best (lowest)
// price first, oldest order first within a price level.
func (b *OrderBook) matchBuy(buyOrder *Order) []Trade {
    var trades []Trade
    for buyOrder.Remaining() > 0 && len(b.asks) > 0 {
        best := b.asks[0]
        if best.Price > buyOrder.Price {
            break // even the cheapest resting ask is too expensive for this buyer — asks are sorted ascending, so nothing further out will qualify either
        }

        for len(best.Orders) > 0 && buyOrder.Remaining() > 0 {
            sellOrder := best.Orders[0]
            qty := min64(buyOrder.Remaining(), sellOrder.Remaining())

            buyOrder.Filled += qty
            sellOrder.Filled += qty
            trades = append(trades, Trade{
                BuyOrderID: buyOrder.ID, SellOrderID: sellOrder.ID,
                Price: best.Price, Quantity: qty, // trade executes at the RESTING order's price — standard convention
            })

            if sellOrder.Remaining() == 0 {
                best.Orders = best.Orders[1:] // fully filled — drop from the FIFO queue
            }
        }

        if len(best.Orders) == 0 {
            b.asks = b.asks[1:] // price level exhausted — remove it
        }
    }
    return trades
}

func (b *OrderBook) matchSell(sellOrder *Order) []Trade {
    var trades []Trade
    for sellOrder.Remaining() > 0 && len(b.bids) > 0 {
        best := b.bids[0]
        if best.Price < sellOrder.Price {
            break // best resting bid still doesn't meet this seller's asking price
        }

        for len(best.Orders) > 0 && sellOrder.Remaining() > 0 {
            buyOrder := best.Orders[0]
            qty := min64(sellOrder.Remaining(), buyOrder.Remaining())

            sellOrder.Filled += qty
            buyOrder.Filled += qty
            trades = append(trades, Trade{
                BuyOrderID: buyOrder.ID, SellOrderID: sellOrder.ID,
                Price: best.Price, Quantity: qty,
            })

            if buyOrder.Remaining() == 0 {
                best.Orders = best.Orders[1:]
            }
        }

        if len(best.Orders) == 0 {
            b.bids = b.bids[1:]
        }
    }
    return trades
}

// restBid inserts an unfilled buy order into the bids, keeping
// descending price order, creating a new PriceLevel if none exists
// yet at this exact price.
func (b *OrderBook) restBid(order *Order) {
    for _, level := range b.bids {
        if level.Price == order.Price {
            level.Orders = append(level.Orders, order) // joins the FIFO queue at this price, correctly at the back
            return
        }
    }
    newLevel := &PriceLevel{Price: order.Price, Orders: []*Order{order}}
    for i, level := range b.bids {
        if order.Price > level.Price {
            b.bids = append(b.bids[:i], append([]*PriceLevel{newLevel}, b.bids[i:]...)...)
            return
        }
    }
    b.bids = append(b.bids, newLevel) // lowest price seen so far — goes at the end
}

func (b *OrderBook) restAsk(order *Order) {
    for _, level := range b.asks {
        if level.Price == order.Price {
            level.Orders = append(level.Orders, order)
            return
        }
    }
    newLevel := &PriceLevel{Price: order.Price, Orders: []*Order{order}}
    for i, level := range b.asks {
        if order.Price < level.Price {
            b.asks = append(b.asks[:i], append([]*PriceLevel{newLevel}, b.asks[i:]...)...)
            return
        }
    }
    b.asks = append(b.asks, newLevel)
}

func min64(a, b int64) int64 {
    if a < b {
        return a
    }
    return b
}
```

```go
// FILE: main.go
package main

import (
    "fmt"

    exchange "example.com/exchange"
)

func main() {
    book := exchange.NewOrderBook()

    // Resting sell order: 100 shares @ $50.00 (5000 cents).
    book.Submit(&exchange.Order{ID: "sell-1", Side: exchange.Sell, Price: 5000, Quantity: 100})

    // Incoming buy: 60 shares @ $50.00 — matches immediately, partial fill of sell-1.
    trades := book.Submit(&exchange.Order{ID: "buy-1", Side: exchange.Buy, Price: 5000, Quantity: 60})
    fmt.Println("Trades:", trades) // [{buy-1 sell-1 5000 60}]

    // Second buy for the remaining 40 shares — fully exhausts sell-1.
    trades2 := book.Submit(&exchange.Order{ID: "buy-2", Side: exchange.Buy, Price: 5000, Quantity: 40})
    fmt.Println("Trades:", trades2) // [{buy-2 sell-1 5000 40}]
}
```

> [!success] Trace this before trusting any matching engine code
> `sell-1` rests with 100 remaining. `buy-1` (60) matches 60 → `sell-1.Remaining()` drops to 40, stays in the book (not fully filled). `buy-2` (40) matches the remaining 40 → `sell-1.Remaining()` hits 0, removed from its price level, the now-empty price level removed from `asks`. Two separate incoming orders correctly draining one resting order across two trades — this is the actual behavior that matters, not just "it compiles."

---

## Step 5 — Concurrency, and why the whole book is one lock

> [!warning] A single mutex around the entire book is the correct default here, not a shortcut
> Matching must be **serialized per symbol** — two incoming orders for the same symbol racing to match against the same resting order would corrupt fills (both could believe they got the same shares). A single `sync.Mutex` per `OrderBook` (one book per symbol) gives that serialization directly and correctly. Real exchanges achieve far higher throughput via single-writer-per-symbol architectures (one goroutine/thread owns a given symbol's book, all order submissions for that symbol funnel through a channel to it) rather than lock contention — worth naming as the natural next-level answer to "how would you scale this," without needing to implement it here.

## Interview Q&A

> [!question]- Why does the trade execute at the resting order's price, not the incoming order's price?
> Standard exchange convention: the order already resting in the book set its price first and has been waiting — the incoming ("aggressive") order is the one choosing to transact *at* that already-published price. If the incoming order's price were used instead, resting orders would inherit unpredictable execution prices from whoever happens to trade against them.

> [!question]- How would you support market orders (execute immediately at any available price) in addition to limit orders?
> A market order skips the price-check entirely — its Strategy is "sweep the book from the best price outward until filled or the book is empty," rather than limit orders' Strategy of "stop once the best remaining price is worse than my limit." Modeling order types as a `MatchingStrategy` interface (limit vs. market as two implementations) is a natural extension of the pattern reuse this book has repeated in every chapter: same shape, new concrete strategy.

> [!question]- What happens on cancel — does it need to touch the sorted price-level structure?
> Yes — cancel must locate the order (the `orders map[string]*Order` gives O(1) lookup of the order itself) and remove it from its `PriceLevel`'s FIFO slice, then remove the `PriceLevel` entirely if it's now empty — the same cleanup already done inline in `matchBuy`/`matchSell` when an order fully fills, just triggered by a cancel instead of a fill.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[LLD/05 - Design Splitwise/Design Splitwise|Design Splitwise]] (integer-cents discipline) · [[Glossary/Strategy Pattern|Strategy Pattern]]*
