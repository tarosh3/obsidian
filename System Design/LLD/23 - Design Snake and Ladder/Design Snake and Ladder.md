---
title: "Design Snake and Ladder"
aliases: [Snake and Ladder LLD, Snakes and Ladders]
tags: [system-design, lld, case-study, go, strategy-pattern]
status: reference-quality
---

# Design Snake and Ladder

> [!abstract] What you'll be able to do after this chapter
> Recognize that a snake and a ladder are the exact same concept in disguise (a "jump" — only the direction differs), model that insight so the board never branches on which one it's looking at, and make the dice swappable via Strategy so testing/cheat variants cost zero changes to the game loop.

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Design the core game engine for Snake and Ladder — a board with snakes and ladders, N players taking turns rolling a die, and a winner when someone reaches the final cell exactly."

## Step 2 — Requirements

A board of a fixed size (standard: 100 cells). Some cells are **snake heads** — landing there sends the player *backward* to a tail cell. Some cells are **ladder bottoms** — landing there sends the player *forward* to a top cell. Players take turns rolling a die and moving forward by the roll. Overshooting the final cell forfeits the move (standard official rule — must land *exactly* on the last cell). First player to land exactly on the final cell wins.

## Step 3 — The bad first draft

Two separate maps, checked via two separate branches every time a player moves:

```go
// The bad draft — snakes and ladders treated as two different things
func resolvePosition(pos int, snakes, ladders map[int]int) int {
    if dest, ok := snakes[pos]; ok {
        return dest
    }
    if dest, ok := ladders[pos]; ok {
        return dest
    }
    return pos
}
```

This *works*, but it's duplicating identical logic twice for no real behavioral reason — a snake and a ladder do the **exact same thing**: "landing on cell X sends you to cell Y." The only difference between them is whether `Y < X` (snake) or `Y > X` (ladder), and the game logic never actually needs to know which — it just needs "where do I actually end up." Keeping them as two separate concepts means every future piece of code that touches board resolution (rendering, move validation, board-setup validation) has to remember to check *both* maps, every time — a maintenance tax for a distinction that carries zero behavioral difference.

## Step 4 — Refactor: unify into one "jump" concept

A single `jumps map[int]int` — source cell to destination cell — represents *both* snakes and ladders. The board doesn't care which one a given entry represents; nothing downstream needs to.

```go
// FILE: dice.go
package snakeandladder

// Dice is the one piece of this game genuinely worth making a
// Strategy — a fair single die, a "loaded" die for deterministic
// testing, or a two-dice-summed variant, all swappable without
// touching the game loop that calls Roll().
type Dice interface {
    Roll() int
}
```

```go
// FILE: fair_dice.go
package snakeandladder

import "math/rand"

type FairDice struct {
    sides int
}

func NewFairDice(sides int) *FairDice {
    return &FairDice{sides: sides}
}

func (d *FairDice) Roll() int {
    return rand.Intn(d.sides) + 1
}
```

```go
// FILE: board.go
package snakeandladder

import "errors"

// Board owns the size and every jump — snake or ladder — as ONE
// unified concept. A snake's source > destination; a ladder's source
// < destination. The board never branches on which one it is.
type Board struct {
    size  int
    jumps map[int]int
}

func NewBoard(size int) *Board {
    return &Board{size: size, jumps: make(map[int]int)}
}

// AddJump registers a snake OR a ladder — same method, same
// validation, because the board only cares about "source sends you
// to destination," never about which real-world thing it represents.
func (b *Board) AddJump(source, destination int) error {
    if source <= 1 || source >= b.size {
        return errors.New("jump source must be strictly within the board, excluding start and final cell")
    }
    if destination < 1 || destination > b.size {
        return errors.New("jump destination out of board bounds")
    }
    if _, exists := b.jumps[source]; exists {
        return errors.New("a jump already starts at this cell")
    }
    // A chained jump — landing on a cell that is ITSELF the start of
    // another jump — is a real, easy-to-miss board-construction bug.
    // Reject it at setup time instead of surprising a player mid-game
    // with an ambiguous double-jump.
    if _, chained := b.jumps[destination]; chained {
        return errors.New("destination cell is itself a jump source — chained jumps are not allowed")
    }
    b.jumps[source] = destination
    return nil
}

// Resolve returns the cell a player actually lands on after moving —
// applying a jump if one starts at this position, otherwise the
// position itself, unchanged.
func (b *Board) Resolve(position int) int {
    if dest, ok := b.jumps[position]; ok {
        return dest
    }
    return position
}

func (b *Board) Size() int {
    return b.size
}
```

```go
// FILE: player.go
package snakeandladder

type Player struct {
    Name     string
    Position int
}

func NewPlayer(name string) *Player {
    return &Player{Name: name, Position: 0}
}
```

```go
// FILE: game.go
package snakeandladder

import "fmt"

type Game struct {
    board       *Board
    dice        Dice
    players     []*Player
    currentTurn int
}

func NewGame(board *Board, dice Dice, players []*Player) *Game {
    return &Game{board: board, dice: dice, players: players}
}

// PlayTurn advances exactly one player's turn and returns the winner
// if this turn produced one, nil otherwise. Keeping a single turn as
// its own testable step — rather than one monolithic Play() loop —
// is what lets a test roll a specific sequence of dice values and
// assert on intermediate board state without running a full game.
func (g *Game) PlayTurn() *Player {
    player := g.players[g.currentTurn]
    roll := g.dice.Roll()

    target := player.Position + roll
    if target > g.board.Size() {
        // Official rule: an overshoot forfeits the move entirely —
        // the player stays put, and the turn simply passes.
        fmt.Printf("%s rolled %d, overshoots %d — stays at %d\n", player.Name, roll, g.board.Size(), player.Position)
        g.advanceTurn()
        return nil
    }

    resolved := g.board.Resolve(target)
    if resolved != target {
        fmt.Printf("%s rolled %d, moved %d -> %d, hit a jump to %d\n", player.Name, roll, player.Position, target, resolved)
    } else {
        fmt.Printf("%s rolled %d, moved %d -> %d\n", player.Name, roll, player.Position, target)
    }
    player.Position = resolved

    if player.Position == g.board.Size() {
        return player
    }

    g.advanceTurn()
    return nil
}

func (g *Game) advanceTurn() {
    g.currentTurn = (g.currentTurn + 1) % len(g.players)
}

// Play runs turns until a player wins, returning the winner.
func (g *Game) Play() *Player {
    for {
        if winner := g.PlayTurn(); winner != nil {
            return winner
        }
    }
}
```

```go
// FILE: main.go
package main

import (
    "fmt"

    sl "example.com/snakeandladder"
)

func main() {
    board := sl.NewBoard(100)

    // Snakes: head -> tail (destination < source).
    board.AddJump(99, 41)
    board.AddJump(70, 55)
    board.AddJump(52, 42)

    // Ladders: bottom -> top (destination > source).
    board.AddJump(3, 22)
    board.AddJump(8, 30)
    board.AddJump(60, 85)

    dice := sl.NewFairDice(6)
    players := []*sl.Player{sl.NewPlayer("Alice"), sl.NewPlayer("Bob")}

    game := sl.NewGame(board, dice, players)
    winner := game.Play()

    fmt.Printf("\n%s wins!\n", winner.Name)
}
```

---

## Step 5 — Why turn-based play is intentionally NOT concurrent

> [!info] A deliberate contrast with earlier chapters
> Unlike [[LLD/06 - Design BookMyShow - Seat Booking/Design BookMyShow - Seat Booking|BookMyShow]] or [[LLD/11 - Design an ATM/Design an ATM|the ATM]], this design has **no `sync.Mutex` anywhere**, and that's not an oversight — a turn-based board game has no genuine concurrent-access requirement by definition: only one player acts at a time, strictly sequentially. Reaching for a mutex here would be solving a problem that doesn't exist. Recognizing when concurrency control is *and isn't* needed is as much a signal in an interview as knowing how to add it.

## Interview Q&A

> [!question]- Why unify snakes and ladders into one `jumps` map instead of two separate types?
> They're behaviorally identical — "landing on cell X sends you to cell Y" — and the *only* difference (direction) is never actually checked by any consuming code. Two separate types would mean every future piece of code touching board resolution has to remember to check both, for a distinction with zero behavioral consequence — pure duplication with no payoff.

> [!question]- How would you support a "loaded dice" variant for testing, or a cheat-detection feature?
> Swap the `Dice` implementation — a `LoadedDice` that always returns 6, or a `RecordingDice` that logs every roll for later replay in a test — with zero changes to `Game`. This is the entire payoff of making Dice a Strategy: the game loop only ever depends on the `Dice` interface, never a concrete implementation.

> [!question]- How do you prevent a pathological board configuration (e.g., a snake and ladder chained together)?
> `Board.AddJump` explicitly rejects registering a jump whose destination is *itself* the source of another jump — caught at board-construction time, not discovered mid-game as a confusing double-jump.

> [!question]- How would you extend this to a "must roll exactly a 6 to leave the starting cell" house rule?
> Add a `hasStarted bool` field to `Player`, and in `PlayTurn`, only apply the moved position if `player.hasStarted` is true or the roll is 6 (setting `hasStarted = true` on that roll) — a small, additive change that slots into the existing turn logic without touching `Board`, `Dice`, or the win-detection logic at all, which is exactly what a well-separated design should allow.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[LLD/09 - Design Chess/Design Chess|Design Chess]] · [[LLD/10 - Design Tic-Tac-Toe with AI/Design Tic-Tac-Toe with AI|Design Tic-Tac-Toe with AI]] · [[CS Fundamentals/Design Principles/Design Patterns Cheat Sheet|Design Patterns Cheat Sheet]]*
