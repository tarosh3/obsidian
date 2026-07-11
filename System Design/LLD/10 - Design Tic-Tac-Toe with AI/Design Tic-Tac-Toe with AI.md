---
title: "Design Tic-Tac-Toe (with AI)"
aliases: [Design Tic-Tac-Toe, Tic-Tac-Toe LLD, Minimax]
tags: [system-design, lld, case-study, golang, design-patterns, strategy-pattern, algorithms]
status: reference-quality
---

# Design Tic-Tac-Toe (with AI)

> [!abstract] What you'll be able to do after this chapter
> Implement minimax correctly (not just describe it) with the perspective-tracking bookkeeping actually right, and swap AI difficulty via Strategy without the game loop knowing whether it's playing a human or a machine.

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Design Tic-Tac-Toe supporting two human players, or a human vs. an AI opponent with swappable difficulty levels."

## Step 2 — Requirement clarification

3×3 board, alternating X/O, win/draw detection. An AI opponent that can play automatically, with **swappable difficulty** — easy (weak) vs. unbeatable optimal play — without the core game loop needing to know which it's facing.

## Step 3 — The bad first draft (kept brief)

AI move selection hardcoded inline in the game loop (e.g. "always pick the first empty cell" — trivially weak). Adding a "hard" difficulty means editing the game loop directly — the familiar Open/Closed shape.

## Step 4 — Refactor: Strategy for move selection

`MoveStrategy` — `SelectMove(board, player) (row, col int)` — implemented independently by `RandomMoveStrategy` (easy) and `MinimaxStrategy` (unbeatable). A human player is simply one with **no strategy at all** — `Game` checks for a `nil` strategy and takes externally-supplied moves instead.

> [!info] Minimax deserves real implementation here, not a description
> Unlike most patterns in this book (where the *pattern* is the point), this chapter's actual substance is the **algorithm** minimax provides — worth implementing precisely and correctly, since "I know minimax exists" and "I can implement minimax correctly" are very different interview signals.

---

## Step 5 — Complete, compilable Go implementation

```go
// ============================================================
// FILE: board.go
// ============================================================
package tictactoe

type Mark int

const (
	Empty Mark = iota
	X
	O
)

type Board struct {
	cells [3][3]Mark
}

func NewBoard() *Board {
	return &Board{}
}

func (b *Board) Get(row, col int) Mark        { return b.cells[row][col] }
func (b *Board) Set(row, col int, mark Mark)  { b.cells[row][col] = mark }

func (b *Board) EmptyCells() [][2]int {
	var cells [][2]int
	for r := 0; r < 3; r++ {
		for c := 0; c < 3; c++ {
			if b.cells[r][c] == Empty {
				cells = append(cells, [2]int{r, c})
			}
		}
	}
	return cells
}

// Clone deep-copies the board. `cells` is a Go ARRAY (not a slice),
// so `clone.cells = b.cells` copies all 9 values by value — no
// aliasing between the original and the clone, exactly what
// minimax's exploratory "what if I play here" trials need.
func (b *Board) Clone() *Board {
	clone := &Board{}
	clone.cells = b.cells
	return clone
}

func (b *Board) Winner() Mark {
	lines := [][3][2]int{
		{{0, 0}, {0, 1}, {0, 2}}, {{1, 0}, {1, 1}, {1, 2}}, {{2, 0}, {2, 1}, {2, 2}},
		{{0, 0}, {1, 0}, {2, 0}}, {{0, 1}, {1, 1}, {2, 1}}, {{0, 2}, {1, 2}, {2, 2}},
		{{0, 0}, {1, 1}, {2, 2}}, {{0, 2}, {1, 1}, {2, 0}},
	}
	for _, line := range lines {
		a := b.cells[line[0][0]][line[0][1]]
		m := b.cells[line[1][0]][line[1][1]]
		c := b.cells[line[2][0]][line[2][1]]
		if a != Empty && a == m && m == c {
			return a
		}
	}
	return Empty
}

func (b *Board) IsFull() bool {
	return len(b.EmptyCells()) == 0
}
```

```go
// ============================================================
// FILE: move_strategy.go
// ============================================================
package tictactoe

import "math/rand"

// MoveStrategy replaced the hardcoded "always pick the first empty
// cell" AI from the bad first draft.
type MoveStrategy interface {
	SelectMove(board *Board, player Mark) (row, col int)
}

// RandomMoveStrategy — "easy" difficulty.
type RandomMoveStrategy struct{}

func (s RandomMoveStrategy) SelectMove(board *Board, player Mark) (int, int) {
	empty := board.EmptyCells()
	choice := empty[rand.Intn(len(empty))]
	return choice[0], choice[1]
}

// MinimaxStrategy — unbeatable optimal play: recursively explore
// every possible continuation, assuming the opponent also plays
// optimally, and pick the move that maximizes THIS player's
// guaranteed outcome.
type MinimaxStrategy struct{}

func (s MinimaxStrategy) SelectMove(board *Board, player Mark) (int, int) {
	opponent := X
	if player == X {
		opponent = O
	}

	bestScore := -2
	bestRow, bestCol := -1, -1

	for _, cell := range board.EmptyCells() {
		trial := board.Clone()
		trial.Set(cell[0], cell[1], player)
		score := minimax(trial, opponent, player, false)
		if score > bestScore {
			bestScore = score
			bestRow, bestCol = cell[0], cell[1]
		}
	}
	return bestRow, bestCol
}

// minimax scores a board position from maximizingPlayer's
// perspective: +1 if maximizingPlayer eventually wins, -1 if the
// opponent wins, 0 for a draw. `currentTurn` and `isMaximizing`
// stay in lockstep throughout the recursion — isMaximizing is true
// exactly when currentTurn == maximizingPlayer, at every level.
func minimax(board *Board, currentTurn Mark, maximizingPlayer Mark, isMaximizing bool) int {
	winner := board.Winner()
	if winner == maximizingPlayer {
		return 1
	}
	opponent := X
	if maximizingPlayer == X {
		opponent = O
	}
	if winner == opponent {
		return -1
	}
	if board.IsFull() {
		return 0
	}

	nextTurn := X
	if currentTurn == X {
		nextTurn = O
	}

	if isMaximizing {
		best := -2
		for _, cell := range board.EmptyCells() {
			trial := board.Clone()
			trial.Set(cell[0], cell[1], currentTurn)
			score := minimax(trial, nextTurn, maximizingPlayer, false)
			if score > best {
				best = score
			}
		}
		return best
	}

	best := 2
	for _, cell := range board.EmptyCells() {
		trial := board.Clone()
		trial.Set(cell[0], cell[1], currentTurn)
		score := minimax(trial, nextTurn, maximizingPlayer, true)
		if score < best {
			best = score
		}
	}
	return best
}
```

```go
// ============================================================
// FILE: game.go
// ============================================================
package tictactoe

import "errors"

var (
	ErrCellOccupied = errors.New("tictactoe: cell already occupied")
	ErrGameOver     = errors.New("tictactoe: game already over")
)

// Player.Strategy is nil for a human — moves come from external
// input instead of being computed automatically.
type Player struct {
	Mark     Mark
	Strategy MoveStrategy
}

type Game struct {
	board         *Board
	players       [2]*Player
	currentPlayer int
}

func NewGame(playerX, playerO *Player) *Game {
	return &Game{
		board:   NewBoard(),
		players: [2]*Player{playerX, playerO},
	}
}

func (g *Game) CurrentPlayer() *Player {
	return g.players[g.currentPlayer]
}

// PlayTurn plays one turn. If the current player has a MoveStrategy,
// the move is computed automatically and row/col are ignored;
// otherwise row/col must be supplied externally (a human move).
func (g *Game) PlayTurn(row, col int) error {
	if g.board.Winner() != Empty || g.board.IsFull() {
		return ErrGameOver
	}

	player := g.CurrentPlayer()
	if player.Strategy != nil {
		row, col = player.Strategy.SelectMove(g.board, player.Mark)
	}

	if g.board.Get(row, col) != Empty {
		return ErrCellOccupied
	}

	g.board.Set(row, col, player.Mark)
	g.currentPlayer = 1 - g.currentPlayer
	return nil
}

func (g *Game) Board() *Board {
	return g.board
}
```

```go
// ============================================================
// FILE: main.go  (adjust import path to your module name)
// ============================================================
package main

import (
	"fmt"

	tictactoe "example.com/tictactoe"
)

func main() {
	human := &tictactoe.Player{Mark: tictactoe.X} // Strategy nil — human
	ai := &tictactoe.Player{Mark: tictactoe.O, Strategy: tictactoe.MinimaxStrategy{}}

	game := tictactoe.NewGame(human, ai)

	if err := game.PlayTurn(1, 1); err != nil { // human plays center
		fmt.Println("error:", err)
	}
	if err := game.PlayTurn(0, 0); err != nil { // AI plays automatically
		fmt.Println("error:", err)
	}

	fmt.Println("Winner so far:", game.Board().Winner())
}
```

---

## 🎯 Interview follow-up Q&A

> [!quote]- "What's the time complexity of this minimax implementation, and does it matter for Tic-Tac-Toe specifically?"
> Roughly `O(9!)` in the worst case (exploring the full game tree from an empty board) — for Tic-Tac-Toe's tiny state space this is trivially fast in practice. For a larger game (Connect Four, Chess), this exact approach would need **alpha-beta pruning** to remain tractable — worth naming as the standard optimization, even though it's unnecessary at this board size.

> [!quote]- "How would you add a 'medium' difficulty — better than random, but beatable?"
> A new `MoveStrategy` implementation — e.g. one that blocks an immediate opponent win or takes an immediate win for itself if available, otherwise falls back to a random move. No changes anywhere else, same as every prior Strategy-pattern chapter's extensibility story.

> [!quote]- "Why does `Board.Clone()` matter for correctness here, not just cleanliness?"
> `minimax` explores many hypothetical futures from the same starting position — without a true independent copy per trial, exploring one branch would corrupt the board state needed for exploring sibling branches. Using a Go array (value-copy semantics) for `cells` makes `Clone()` trivially correct; using a slice-of-slices instead would require manually deep-copying each row, an easy detail to get wrong.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[LLD/04 - Design a Rate Limiter/Design a Rate Limiter|Design a Rate Limiter (LLD)]]*
