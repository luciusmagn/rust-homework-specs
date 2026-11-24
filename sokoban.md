# Sokoban

Implement a Sokoban board engine. Submit `solution.rs`, using Rust 1.88, edition 2021, and the standard library. Judge programs import `use solution::*;`.

## Interface

Define these public types. Derive `Debug`, `Clone`, `Copy`, `PartialEq` and `Eq` for `Direction`, `Cell` and `MoveResult`; derive `Debug`, `PartialEq` and `Eq` for `ParseError`.

```rust
pub enum Direction { Up, Down, Left, Right }

pub enum Cell {
    Wall, Floor, Goal, Crate, CrateOnGoal, Player, PlayerOnGoal,
}

pub enum MoveResult { Blocked, Moved, Pushed }

pub enum ParseError {
    Empty,
    Ragged,
    InvalidChar { x: usize, y: usize, ch: char },
    PlayerCount { found: usize },
    CrateGoalMismatch { crates: usize, goals: usize },
}
```

Define a public `Board` with private storage and these inherent methods:

| Signature | Meaning |
| --- | --- |
| `pub fn parse(text: &str) -> Result<Self, ParseError>` | Parse a board. |
| `pub fn width(&self) -> usize` | Number of columns. |
| `pub fn height(&self) -> usize` | Number of rows. |
| `pub fn player(&self) -> (usize, usize)` | Current player coordinates. |
| `pub fn cell(&self, x: usize, y: usize) -> Option<Cell>` | Current cell, or `None` outside the board. |
| `pub fn step(&mut self, direction: Direction) -> MoveResult` | Attempt one move. |
| `pub fn is_solved(&self) -> bool` | Whether every crate is on a goal. |

Coordinates start at `(0, 0)` in the upper-left corner; x increases rightward and y downward.

## Board format

| Character | Cell |
| --- | --- |
| `#` | Wall |
| space | Floor |
| `.` | Goal |
| `$` | Crate on floor |
| `*` | Crate on goal |
| `@` | Player on floor |
| `+` | Player on goal |

Split rows with `str::lines()`. Accept LF and CRLF, with or without a final line ending. Preserve spaces and blank rows; do not trim or pad. Widths and character coordinates count Unicode scalar values, but only the seven characters above are valid.

Check errors in this order:

1. `Empty` if there are no rows or the first row is empty.
2. `Ragged` if any row has a different width.
3. `InvalidChar` for the first invalid character in row-major order.
4. `PlayerCount` unless there is exactly one player (`@` or `+`).
5. `CrateGoalMismatch` unless the crate count (`$`, `*`) equals the goal count (`.`, `*`, `+`).

Zero crates and zero goals are valid and already solved. Boards need not be enclosed by walls, connected, or solvable. Tests use at most 256 rows and 256 columns.

## Movement

Move one cell in the given direction. Entering floor or an unoccupied goal returns `Moved`. If the adjacent cell contains a crate, push it one cell in the same direction and return `Pushed`, provided its destination is floor or an unoccupied goal. Move the player into the crate's previous cell.

A wall, the board boundary, or a crate that cannot be pushed returns `Blocked` without changing the board. Never push two crates at once. Goals persist when the player or a crate leaves them. Moves are allowed after the board is solved, and a later push may make it unsolved again.

## Example

```rust
use solution::*;

fn main() {
    let mut board = Board::parse("#######\n#@ $. #\n#######\n").unwrap();
    assert_eq!((board.width(), board.height()), (7, 3));
    assert!(!board.is_solved());
    assert_eq!(board.step(Direction::Right), MoveResult::Moved);
    assert_eq!(board.step(Direction::Right), MoveResult::Pushed);
    assert_eq!(board.player(), (3, 1));
    assert_eq!(board.cell(4, 1), Some(Cell::CrateOnGoal));
    assert!(board.is_solved());
    assert_eq!(board.step(Direction::Right), MoveResult::Pushed);
    assert_eq!(board.cell(4, 1), Some(Cell::PlayerOnGoal));
    assert!(!board.is_solved());
}
```
