---
title: "Design Chess"
aliases: [Design Chess, Chess LLD]
tags: [system-design, lld, case-study, golang, design-patterns, polymorphism]
status: reference-quality
---

# Design Chess

> [!abstract] What you'll be able to do after this chapter
> Replace a piece-type switch statement with real polymorphism, share sliding-movement logic across Rook/Bishop/Queen without duplication, and implement check detection by reusing each piece's own move logic — not a separate, parallel "attack" system.

> [!info] Scope, stated deliberately
> This chapter implements core piece movement, captures, and check detection. Castling, en passant, and pawn promotion are named as real extensions but omitted from the code — a deliberate scope decision to keep the implementation focused, not an oversight.

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Design a chess game — board, pieces with their own movement rules, turn-based play, check detection."

## Step 2 — Requirement clarification

8×8 board, six piece types each with distinct movement, alternating turns, detect check, reject moves that leave the mover's own king in check.

> [!example]+ 🪜 How to build this live, step by step (interview execution order, with code)
> Chess is one of the heaviest LLD questions asked — push to scope it down explicitly ("just legal movement + turns, or also check/checkmate/castling?") before writing anything. Realistically you code 2-3 piece types fully, not all 6.
>
> **Checkpoint 1 (~8-10 min) — `Board` + ONE piece type, no `Game` yet.**
> ```go
> type Position struct{ Row, Col int }
>
> func (p Position) InBounds() bool {
>     return p.Row >= 0 && p.Row < 8 && p.Col >= 0 && p.Col < 8
> }
>
> type Piece interface {
>     Color() Color
>     ValidMoves(board *Board, from Position) []Position
> }
>
> type basePiece struct{ color Color }
>
> func (b basePiece) Color() Color { return b.color }
>
> type Rook struct{ basePiece }
>
> func NewRook(color Color) *Rook { return &Rook{basePiece{color}} }
>
> func (r *Rook) ValidMoves(board *Board, from Position) []Position {
>     return slideMoves(board, from, r.color, []Position{{1, 0}, {-1, 0}, {0, 1}, {0, -1}})
> }
>
> type Board struct{ cells [8][8]Piece }
>
> func (b *Board) Get(pos Position) Piece    { return b.cells[pos.Row][pos.Col] }
> func (b *Board) Set(pos Position, p Piece) { b.cells[pos.Row][pos.Col] = p }
> ```
> **Pattern used: none yet — but name where it's going.** Say out loud: *"I'm making `Piece` an interface now instead of a type-switch, because I know I'll need Bishop, Queen, Knight, King, Pawn to plug in the same way."* `slideMoves` itself comes next checkpoint — for now, place one `Rook` on the board and call `ValidMoves` directly to prove the wiring compiles.
>
> **Checkpoint 2 (~10 min) — the shared `slideMoves` helper + a second piece type, proving the abstraction generalizes.**
> ```go
> // slideMoves: walk each direction until the edge, a friendly piece,
> // or a capture. Rook, Bishop, Queen ALL call this — they differ only
> // in WHICH directions they pass in, not the sliding mechanics itself.
> func slideMoves(board *Board, from Position, color Color, directions []Position) []Position {
>     var moves []Position
>     for _, d := range directions {
>         pos := Position{from.Row + d.Row, from.Col + d.Col}
>         for pos.InBounds() {
>             occupant := board.Get(pos)
>             if occupant == nil {
>                 moves = append(moves, pos)
>             } else {
>                 if occupant.Color() != color {
>                     moves = append(moves, pos) // capture, then stop
>                 }
>                 break
>             }
>             pos = Position{pos.Row + d.Row, pos.Col + d.Col}
>         }
>     }
>     return moves
> }
>
> type Bishop struct{ basePiece }
>
> func (b *Bishop) ValidMoves(board *Board, from Position) []Position {
>     return slideMoves(board, from, b.color, []Position{{1, 1}, {1, -1}, {-1, 1}, {-1, -1}})
> }
> ```
> **Pattern used: polymorphism (structurally the same idea as Strategy elsewhere in this book) replacing a type-switch.** This is the checkpoint that actually proves the design — one shared helper, zero duplicated sliding logic. Add Knight (fixed-offset jumps, genuinely different shape, Step 5) if time allows, to show you handle more than one movement style.
>
> **Checkpoint 3 (~8-10 min) — `Game` with turn management and basic legality, fully wired.**
> ```go
> type Game struct {
>     board       *Board
>     currentTurn Color
> }
>
> func (g *Game) Move(from, to Position) error {
>     piece := g.board.Get(from)
>     if piece == nil {
>         return ErrNoPieceAtSource
>     }
>     if piece.Color() != g.currentTurn {
>         return ErrNotYourTurn
>     }
>     legal := false
>     for _, m := range piece.ValidMoves(g.board, from) {
>         if m == to {
>             legal = true
>             break
>         }
>     }
>     if !legal {
>         return ErrIllegalMove
>     }
>     g.board.Set(to, piece)
>     g.board.Set(from, nil)
>     if g.currentTurn == White {
>         g.currentTurn = Black
>     } else {
>         g.currentTurn = White
>     }
>     return nil
> }
> ```
> **Pattern used: none new** — this wires the pieces from Checkpoints 1-2 into an actual playable sequence. Self-check (does this move leave MY OWN king exposed) is deliberately deferred to Checkpoint 4.
>
> **Checkpoint 4 (remaining time, or if asked) — check detection, reusing existing move logic.**
> ```go
> // IsInCheck asks EVERY opposing piece for its own ValidMoves and
> // checks if any lands on the king's square — reuses Piece.ValidMoves
> // instead of building a separate, parallel "attack" system.
> func (b *Board) IsInCheck(color Color) bool {
>     kingPos := b.findKing(color)
>     for r := 0; r < 8; r++ {
>         for c := 0; c < 8; c++ {
>             p := b.cells[r][c]
>             if p != nil && p.Color() != color {
>                 for _, move := range p.ValidMoves(b, Position{r, c}) {
>                     if move == kingPos {
>                         return true
>                     }
>                 }
>             }
>         }
>     }
>     return false
> }
> ```
> This is the single best "aha" insight to state if you reach it: check detection isn't new logic, it's the *same* `ValidMoves` every piece already has, asked a different question. Castling/en passant/promotion should stay verbal-only (Step 4's Q&A) unless there's genuinely time left.
>
> **If you're short on time:** stop after Checkpoint 2. `Board` + 3 piece types with correctly shared sliding logic, demoed with a couple of `ValidMoves` calls, is a strong, complete-feeling answer — describe `Game`, turn management, and check detection verbally as the next layers.

## Step 3 — The bad first draft (kept brief — a familiar shape by now)

A single `Piece` struct with a type field, and movement validation via a giant switch on piece type living **inside the board/game logic itself**. Adding a new piece variant (or a chess variant with custom pieces) means editing that central switch — the same Open/Closed shape as every prior chapter.

## Step 4 — Refactor: polymorphism replaces the switch entirely

Each piece type becomes its own struct implementing a common `Piece` interface — `ValidMoves(board, position) []Position`. The board never needs to know Bishop-specific or Knight-specific rules; it just calls the interface method.

> [!tip] This is arguably Strategy again, under a different name
> "One struct per case instead of a switch" is often just called polymorphism, but it's structurally identical to every Strategy application elsewhere in this handbook — each piece type is a swappable move-computation strategy. Pattern names blur together once interfaces are used well; that's expected, not a sign of imprecision.

---

## Step 5 — Complete, compilable Go implementation

```go
// ============================================================
// FILE: types.go
// ============================================================
package chess

type Color int

const (
	White Color = iota
	Black
)

type Position struct {
	Row, Col int
}

func (p Position) InBounds() bool {
	return p.Row >= 0 && p.Row < 8 && p.Col >= 0 && p.Col < 8
}
```

```go
// ============================================================
// FILE: piece.go
// ============================================================
package chess

// Piece replaced the switch-on-piece-type chain from the bad first
// draft — each type computes its own valid moves.
type Piece interface {
	Color() Color
	ValidMoves(board *Board, from Position) []Position
	Symbol() rune
}

type basePiece struct {
	color Color
}

func (b basePiece) Color() Color { return b.color }
```

```go
// ============================================================
// FILE: pieces.go
// ============================================================
package chess

// ---- Rook, Bishop, Queen share slideMoves — they differ only in
// WHICH directions they slide, not the sliding mechanics itself.
// Sharing this helper avoids duplicating the same "walk until edge,
// friendly piece, or capture" logic three times. ----

type Rook struct{ basePiece }

func NewRook(color Color) *Rook { return &Rook{basePiece{color}} }
func (r *Rook) Symbol() rune {
	if r.color == White {
		return 'R'
	}
	return 'r'
}
func (r *Rook) ValidMoves(board *Board, from Position) []Position {
	return slideMoves(board, from, r.color, []Position{{1, 0}, {-1, 0}, {0, 1}, {0, -1}})
}

type Bishop struct{ basePiece }

func NewBishop(color Color) *Bishop { return &Bishop{basePiece{color}} }
func (b *Bishop) Symbol() rune {
	if b.color == White {
		return 'B'
	}
	return 'b'
}
func (b *Bishop) ValidMoves(board *Board, from Position) []Position {
	return slideMoves(board, from, b.color, []Position{{1, 1}, {1, -1}, {-1, 1}, {-1, -1}})
}

type Queen struct{ basePiece }

func NewQueen(color Color) *Queen { return &Queen{basePiece{color}} }
func (q *Queen) Symbol() rune {
	if q.color == White {
		return 'Q'
	}
	return 'q'
}
func (q *Queen) ValidMoves(board *Board, from Position) []Position {
	directions := []Position{{1, 0}, {-1, 0}, {0, 1}, {0, -1}, {1, 1}, {1, -1}, {-1, 1}, {-1, -1}}
	return slideMoves(board, from, q.color, directions)
}

// ---- Knight: fixed L-shaped jumps, ignores pieces in between ----

type Knight struct{ basePiece }

func NewKnight(color Color) *Knight { return &Knight{basePiece{color}} }
func (k *Knight) Symbol() rune {
	if k.color == White {
		return 'N'
	}
	return 'n'
}
func (k *Knight) ValidMoves(board *Board, from Position) []Position {
	offsets := []Position{{2, 1}, {2, -1}, {-2, 1}, {-2, -1}, {1, 2}, {1, -2}, {-1, 2}, {-1, -2}}
	var moves []Position
	for _, o := range offsets {
		to := Position{from.Row + o.Row, from.Col + o.Col}
		if to.InBounds() && canLandOn(board, to, k.color) {
			moves = append(moves, to)
		}
	}
	return moves
}

// ---- King: one step in any direction (castling omitted — Step 0) ----

type King struct{ basePiece }

func NewKing(color Color) *King { return &King{basePiece{color}} }
func (k *King) Symbol() rune {
	if k.color == White {
		return 'K'
	}
	return 'k'
}
func (k *King) ValidMoves(board *Board, from Position) []Position {
	directions := []Position{{1, 0}, {-1, 0}, {0, 1}, {0, -1}, {1, 1}, {1, -1}, {-1, 1}, {-1, -1}}
	var moves []Position
	for _, d := range directions {
		to := Position{from.Row + d.Row, from.Col + d.Col}
		if to.InBounds() && canLandOn(board, to, k.color) {
			moves = append(moves, to)
		}
	}
	return moves
}

// ---- Pawn: forward-only movement, diagonal-only captures
// (en passant, promotion omitted — Step 0) ----

type Pawn struct{ basePiece }

func NewPawn(color Color) *Pawn { return &Pawn{basePiece{color}} }
func (p *Pawn) Symbol() rune {
	if p.color == White {
		return 'P'
	}
	return 'p'
}
func (p *Pawn) ValidMoves(board *Board, from Position) []Position {
	var moves []Position
	direction := 1
	if p.color == Black {
		direction = -1
	}

	forward := Position{from.Row + direction, from.Col}
	if forward.InBounds() && board.Get(forward) == nil {
		moves = append(moves, forward)
	}

	for _, dc := range []int{-1, 1} {
		capture := Position{from.Row + direction, from.Col + dc}
		if capture.InBounds() {
			target := board.Get(capture)
			if target != nil && target.Color() != p.color {
				moves = append(moves, capture)
			}
		}
	}
	return moves
}

// ---- shared helpers ----

func slideMoves(board *Board, from Position, color Color, directions []Position) []Position {
	var moves []Position
	for _, d := range directions {
		pos := Position{from.Row + d.Row, from.Col + d.Col}
		for pos.InBounds() {
			occupant := board.Get(pos)
			if occupant == nil {
				moves = append(moves, pos)
			} else {
				if occupant.Color() != color {
					moves = append(moves, pos) // capture, then stop
				}
				break
			}
			pos = Position{pos.Row + d.Row, pos.Col + d.Col}
		}
	}
	return moves
}

func canLandOn(board *Board, pos Position, color Color) bool {
	occupant := board.Get(pos)
	return occupant == nil || occupant.Color() != color
}
```

```go
// ============================================================
// FILE: board.go
// ============================================================
package chess

type Board struct {
	cells [8][8]Piece
}

func NewBoard() *Board {
	b := &Board{}
	b.setupBackRank(0, White)
	b.setupPawns(1, White)
	b.setupBackRank(7, Black)
	b.setupPawns(6, Black)
	return b
}

func (b *Board) setupBackRank(row int, color Color) {
	order := []func(Color) Piece{
		func(c Color) Piece { return NewRook(c) },
		func(c Color) Piece { return NewKnight(c) },
		func(c Color) Piece { return NewBishop(c) },
		func(c Color) Piece { return NewQueen(c) },
		func(c Color) Piece { return NewKing(c) },
		func(c Color) Piece { return NewBishop(c) },
		func(c Color) Piece { return NewKnight(c) },
		func(c Color) Piece { return NewRook(c) },
	}
	for col, factory := range order {
		b.cells[row][col] = factory(color)
	}
}

func (b *Board) setupPawns(row int, color Color) {
	for col := 0; col < 8; col++ {
		b.cells[row][col] = NewPawn(color)
	}
}

func (b *Board) Get(pos Position) Piece       { return b.cells[pos.Row][pos.Col] }
func (b *Board) Set(pos Position, p Piece)    { b.cells[pos.Row][pos.Col] = p }

func (b *Board) findKing(color Color) Position {
	for r := 0; r < 8; r++ {
		for c := 0; c < 8; c++ {
			p := b.cells[r][c]
			if p != nil {
				if _, ok := p.(*King); ok && p.Color() == color {
					return Position{r, c}
				}
			}
		}
	}
	return Position{-1, -1}
}

// IsInCheck asks EVERY opposing piece for its own valid moves and
// checks if any lands on the king's square — reusing each Piece's
// existing ValidMoves rather than a separate, duplicated "attack
// logic" system.
func (b *Board) IsInCheck(color Color) bool {
	kingPos := b.findKing(color)
	for r := 0; r < 8; r++ {
		for c := 0; c < 8; c++ {
			p := b.cells[r][c]
			if p != nil && p.Color() != color {
				for _, move := range p.ValidMoves(b, Position{r, c}) {
					if move == kingPos {
						return true
					}
				}
			}
		}
	}
	return false
}
```

```go
// ============================================================
// FILE: game.go
// ============================================================
package chess

import "errors"

var (
	ErrNoPieceAtSource   = errors.New("chess: no piece at source position")
	ErrNotYourTurn       = errors.New("chess: not this color's turn")
	ErrIllegalMove       = errors.New("chess: illegal move for this piece")
	ErrMoveLeavesInCheck = errors.New("chess: move would leave your own king in check")
)

type Game struct {
	board       *Board
	currentTurn Color
}

func NewGame() *Game {
	return &Game{board: NewBoard(), currentTurn: White}
}

func (g *Game) Move(from, to Position) error {
	piece := g.board.Get(from)
	if piece == nil {
		return ErrNoPieceAtSource
	}
	if piece.Color() != g.currentTurn {
		return ErrNotYourTurn
	}

	legal := false
	for _, m := range piece.ValidMoves(g.board, from) {
		if m == to {
			legal = true
			break
		}
	}
	if !legal {
		return ErrIllegalMove
	}

	// Simulate the move to ensure it doesn't leave the mover's own
	// king in check — a real, distinct legality rule beyond the
	// piece's own movement pattern.
	captured := g.board.Get(to)
	g.board.Set(to, piece)
	g.board.Set(from, nil)

	if g.board.IsInCheck(g.currentTurn) {
		g.board.Set(from, piece)
		g.board.Set(to, captured)
		return ErrMoveLeavesInCheck
	}

	if g.currentTurn == White {
		g.currentTurn = Black
	} else {
		g.currentTurn = White
	}
	return nil
}

func (g *Game) Board() *Board { return g.board }
```

```go
// ============================================================
// FILE: main.go  (adjust import path to your module name)
// ============================================================
package main

import (
	"fmt"

	chess "example.com/chess"
)

func main() {
	game := chess.NewGame()

	if err := game.Move(chess.Position{Row: 1, Col: 4}, chess.Position{Row: 3, Col: 4}); err != nil {
		fmt.Println("move failed:", err)
	} else {
		fmt.Println("White pawn moved e2->e4")
	}

	if err := game.Move(chess.Position{Row: 6, Col: 4}, chess.Position{Row: 4, Col: 4}); err != nil {
		fmt.Println("move failed:", err)
	} else {
		fmt.Println("Black pawn moved e7->e5")
	}

	fmt.Println("White in check:", game.Board().IsInCheck(chess.White))
}
```

---

## 🎯 Interview follow-up Q&A

> [!quote]- "Why does `Move` simulate the move and check for self-check rather than just validating the piece's own movement pattern?"
> A move can be perfectly legal for the piece itself (a Bishop sliding diagonally) while still being illegal in chess overall, if it exposes the mover's own king to capture. That's a board-level legality rule, not a piece-level one — it has to be checked after tentatively applying the move, which is exactly why the move is applied, checked, and rolled back if it fails.

> [!quote]- "How would you add castling?"
> It doesn't fit cleanly into `Piece.ValidMoves` alone, since it involves two pieces (King and Rook) moving together and depends on history (neither piece has moved yet) — it would need to be modeled as a special move type checked at the `Game` level, with its own validity rules, rather than forced into the existing per-piece interface.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[LLD/01 - Design a Parking Lot/Design a Parking Lot|Design a Parking Lot]]*
