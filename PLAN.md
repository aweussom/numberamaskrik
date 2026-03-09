# Number Match — Game Design Specification

## Overview

Number Match (also known as Numberama, Seeds, or the Russian Numbers Game) is a single-player puzzle game played on a grid of digits. The player removes pairs of numbers that are equal or sum to 10. The goal is to clear the entire board.

---

## Core Rules

### Valid Pairs

Two cells can be matched if:
- Their values are **equal** (e.g., 5 & 5), OR
- Their values **sum to 10** (e.g., 3 & 7, 1 & 9, 2 & 8)

### Valid Adjacency

Two cells are adjacent (and therefore matchable) if there is a **clear path** between them with no intervening active (non-empty) cells, along one of these paths:

- **Horizontal**: same row, no active cells between them
- **Vertical**: same column, no active cells between them
- **Row-wrap**: end of one row to start of the next, with no active cells between (treating the grid as a single flat sequence)

Note: Diagonal adjacency is not valid.

### Matching

When a valid pair is selected, both cells are cleared (removed from the board). Cleared cells are visually empty but remain in the grid structure — they do not collapse or shift other cells.

### Appending

When no valid moves remain, the player may trigger an **Append**. All remaining active cells, read left-to-right, top-to-bottom, are copied and appended as new rows at the bottom of the board. This may create new adjacencies.

### Win Condition

The board is cleared entirely — no active cells remain.

### Loss / Deadlock

The game cannot technically be lost, but a board can reach a state where appending produces no new valid moves. This is a practical dead end. Whether to present this as a "game over" or allow infinite appending is an implementation choice.

---

## Board Initialization

### Classic Mode

The starting board is the fixed sequence:

```
1 2 3 4 5 6 7 8 9 1 2 3 4 5 6 7 8 9
```

Laid out in rows of 9:

```
1 2 3 4 5 6 7 8 9
1 2 3 4 5 6 7 8 9
```

This is the canonical starting position used in the original paper/graph-paper version of the game.

### Randomized Mode

Generate a board by:
1. Deciding on grid width (typically 8 or 9 columns)
2. Filling N rows with random digits 1–9
3. Optionally seeding with a fixed RNG seed so the board is reproducible (required for daily challenges and share/compare features)

Solvability is not guaranteed with pure random generation. See **Solvability** section below.

---

## Variants

### Daily Challenge

- One board per calendar day, identical for all players worldwide
- Generated from a fixed seed derived from the date (e.g., `seed = YYYYMMDD` fed into a seeded RNG)
- Track and display: moves made, appends used, time taken
- Optional: leaderboard or "share result" (similar to Wordle's share mechanic)

### Seeded / Shareable Boards

- Player can share a seed string; recipient loads the same starting board
- Useful for competitive play or comparing strategies

### Timed Mode

- Standard rules with a countdown timer
- Penalize appends (e.g., -30 seconds per append)

### Hardcore / No-Append Mode

- Appending is disabled; the player must clear the board from the initial state only
- Dramatically increases difficulty; only viable with solvable board generation

### Tutorial Mode

- Fixed hand-crafted boards that teach one mechanic at a time:
  1. Simple adjacent pairs
  2. Row-wrap adjacency
  3. Planning removal order to open new adjacencies
  4. Append usage

---

## Solvability

Pure random boards are not guaranteed solvable. Options:

1. **Ignore it** (most apps do): the puzzle may deadlock; player just starts a new game
2. **Generate-and-verify**: generate a board, run a solver, discard unsolvable boards — expensive for large grids
3. **Generative construction**: build the board backwards from a solved state by reversing valid match pairs — guarantees solvability but limits board variety
4. **Hint system**: if a board is unsolvable, surface a warning or auto-restart

For daily challenges, pre-verify solvability offline.

---

## Data Model

```
Board:
  cells: 2D array of Cell
  width: int
  height: int  // grows on append

Cell:
  value: int (1–9) | null  // null = cleared
  row: int
  col: int

GameState:
  board: Board
  selectedCell: Cell | null
  moveCount: int
  appendCount: int
  startTime: timestamp
  seed: string | null
```

---

## Interaction Model

### Selection

- Tap/click a cell to select it (highlight)
- Tap/click a second cell:
  - If the pair is valid → clear both, deselect
  - If invalid → flash error, keep or deselect first selection (choose one UX convention and stick to it)
- Tap/click the selected cell again → deselect

### Swipe (mobile)

- Swipe from one cell to another to attempt a match
- More fluid but requires careful hit-testing to distinguish swipe from scroll

### Append Button

- Always visible but disabled when valid moves exist (enforce this to prevent abuse)
- Or: always enabled, but appending when moves exist costs a penalty (timed mode)

---

## Adjacency Algorithm

```python
def is_adjacent(board, a, b):
    flat = flatten(board)  # left-to-right, top-to-bottom, preserving nulls
    idx_a = flat_index(a)
    idx_b = flat_index(b)
    if idx_a == idx_b:
        return False

    lo, hi = min(idx_a, idx_b), max(idx_a, idx_b)

    # Check flat (row-wrap) adjacency: no active cells between them
    between_flat = flat[lo+1:hi]
    if all(c is None for c in between_flat):
        return True

    # Check same-column vertical adjacency
    if a.col == b.col:
        col_cells_between = cells_between_in_column(board, a, b)
        if all(c is None for c in col_cells_between):
            return True

    return False
```

The flat adjacency check covers both same-row horizontal and row-wrap cases simultaneously, since the flat sequence is the canonical ordering.

---

## UI/UX Notes

- Grid cells should be square with clear borders; cleared cells visually distinct (blank, not hidden)
- Selected cell: strong highlight
- Valid second selection: brief positive animation before clearing
- Invalid pair: brief shake or color flash
- On append: animate new rows sliding in from bottom
- Show remaining cell count prominently — this is the player's primary progress indicator
- Undo (at least one level) is standard in modern implementations and strongly recommended

---

## Tech Stack Suggestions

- **Web**: HTML Canvas or CSS Grid + vanilla JS; React with useReducer for state
- **Mobile**: Flutter (single codebase), or native SwiftUI / Jetpack Compose
- **State**: the entire game state is serializable as a flat array of integers plus metadata — trivial to save/restore/share

---

## Out of Scope (explicit non-goals unless you want them)

- Multiplayer
- IAP / ads hooks (architecture decision, not game logic)
- Animations beyond basic feedback
