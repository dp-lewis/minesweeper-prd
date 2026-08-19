# Product Requirements Document: Grid Deduction Game ("Minesweeper")

**Version:** 1.0
**Status:** Ready for implementation
**Audience:** An engineer who has never seen or played this game

---

## 0. How to read this document

This PRD is written under one deliberate constraint: **the reader is assumed to have zero prior knowledge of the product.** No rule is left to "everyone knows that." Where a rule is commonly assumed but rarely written down, it is marked **[COMMONLY OMITTED]** — these are the requirements that separate a working build from a broken one.

Requirements are numbered (`FR-n`, `UX-n`, `NFR-n`) so they can be referenced in tickets, tests, and review comments. Every functional requirement is written to be independently verifiable.

---

## 1. Context and intent

### 1.1 What we are building

A single-player logic puzzle played on a rectangular grid. A fixed number of hidden "mines" are distributed randomly across the grid. The player's job is to identify every cell that does *not* contain a mine, using numeric clues that are revealed as they play. The game ends in a win when all safe cells have been identified, or a loss the moment a mine is disturbed.

The core loop is **deduction, not reflex**. The player is never asked to react quickly, aim precisely, or guess blindly under normal play. Every well-formed board position should be solvable by reasoning, and the game's job is to present the information needed for that reasoning as clearly as possible.

### 1.2 Why this product

Short-session puzzle games have durable appeal: a round lasts between ten seconds and ten minutes, requires no narrative context, no tutorial beyond a few sentences, and no persistent account. The design succeeds because the rules are trivially small but the state space is large, so difficulty comes from the player's reasoning rather than from mechanical complexity.

### 1.3 Target player

| | |
|---|---|
| **Primary** | An adult on a desktop or laptop with a keyboard and mouse, looking for a 1–10 minute break. Comfortable with basic computer use. No gaming experience assumed. |
| **Secondary** | A returning player optimising for completion time on the hardest preset. Cares about the timer, precise input, and the absence of unlucky losses. |

### 1.4 Success criteria

- **SC-1** A first-time player who reads the in-product instructions can complete a Beginner board without external help.
- **SC-2** A round never ends because of software behaviour the player could not have anticipated (see FR-14, first-click safety).
- **SC-3** All input responds within one animation frame; no interaction feels laggy on the largest supported board.

### 1.5 Explicitly out of scope for v1

Multiplayer, accounts, server-side persistence, monetisation, sound, animation beyond simple state changes, procedurally *guaranteed-solvable* boards (see §9, Open Questions), custom themes, mobile-first touch design (see §8, Non-goals).

---

## 2. Domain model

Define these terms once; the rest of the document uses them precisely.

**Board** — a rectangular grid of cells, `width` columns by `height` rows. Cell positions are addressed `(row, col)`, zero-indexed from the top-left.

**Cell** — one square of the grid. Every cell has:

| Attribute | Type | Meaning |
|---|---|---|
| `hasMine` | boolean | Whether a mine is hidden in this cell. Never directly visible to the player during play. |
| `adjacentMines` | integer 0–8 | How many of this cell's neighbours contain mines. Derived, not stored independently. |
| `state` | enum | One of `COVERED`, `REVEALED`, `FLAGGED`. See §2.1. |

**Neighbours** — the up-to-eight cells orthogonally *and diagonally* adjacent to a given cell. **[COMMONLY OMITTED]** Diagonals count. A cell in the middle of the board has 8 neighbours; an edge cell has 5; a corner cell has 3. Every rule in this document that mentions "adjacent" or "neighbouring" uses this eight-way definition.

```
Neighbours of X (all 8 marked ·):

· · ·
· X ·
· · ·
```

**Mine** — a hidden hazard. Revealing one ends the game in a loss.

**Flag** — a player-placed marker on a covered cell, meaning "I believe a mine is here." A flag is a note to the player and a safety interlock; it has no effect on the hidden board.

**Reveal** (also "open", "uncover") — the act of exposing a cell's contents.

### 2.1 Cell state machine

```
                  reveal (FR-9)
   ┌──────────┐ ─────────────────► ┌──────────┐
   │ COVERED  │                    │ REVEALED │  (terminal)
   └──────────┘ ◄────┐             └──────────┘
        │            │
  flag  │            │ unflag
(FR-13) │            │ (FR-13)
        ▼            │
   ┌──────────┐ ─────┘
   │ FLAGGED  │
   └──────────┘
```

- **FR-1** `REVEALED` is terminal. A revealed cell can never return to `COVERED` or `FLAGGED`.
- **FR-2** A `FLAGGED` cell cannot be revealed by any player action. **[COMMONLY OMITTED]** This is the entire practical purpose of flagging: it prevents the player from destroying their own game with a misclick. A build where flags are decorative is a broken build.

---

## 3. Board generation

- **FR-3** A new game is defined by three parameters: `width`, `height`, `mineCount`.
- **FR-4** Mines are placed uniformly at random, without replacement, across eligible cells. Every eligible cell must have equal probability of receiving a mine. Exactly `mineCount` mines are placed — no more, no fewer.
- **FR-5** `mineCount` must satisfy `1 ≤ mineCount ≤ (width × height) − 9`. The `− 9` reserve exists to guarantee first-click safety (FR-14) can always be honoured, since that rule may need to keep a full 3×3 block mine-free.

### 3.1 Difficulty presets

| Preset | Width | Height | Mines | Density |
|---|---|---|---|---|
| Beginner | 9 | 9 | 10 | 12.3% |
| Intermediate | 16 | 16 | 40 | 15.6% |
| Expert | 30 | 16 | 99 | 20.6% |

- **FR-6** All three presets must be available. Expert is wider than it is tall; this is intentional and must not be "corrected" to a square.
- **FR-7** A Custom option accepts player-supplied `width`, `height`, `mineCount`, validated against FR-5 and NFR-2 bounds. Invalid input is rejected with a message naming the violated bound; it must not silently clamp.

### 3.2 Deriving the clues

- **FR-8** For every cell, `adjacentMines` equals the count of its neighbours (eight-way, §2) that have `hasMine == true`. Compute this after mine placement. A cell that itself contains a mine still has an `adjacentMines` value, but it is never displayed — revealing that cell ends the game instead.

**Worked example.** A 4×4 board with mines at (0,1), (2,0), (2,2). Left: hidden truth (`*` = mine). Right: the derived clue numbers.

```
hidden truth          derived adjacentMines
. * . .                 1 * 1 0
. . . .                 2 3 2 1
* . * .                 1 * 1 1
. . . .                 1 2 2 1
```

Check (1,1): its eight neighbours are (0,0) (0,1) (0,2) (1,0) (1,2) (2,0) (2,1) (2,2). Three of those — (0,1), (2,0), (2,2) — hold mines, so `adjacentMines = 3`.

---

## 4. Core interaction

Three player actions exist. Nothing else changes board state.

| Action | Default input | Requirement |
|---|---|---|
| Reveal | Left click | FR-9 |
| Toggle flag | Right click | FR-13 |
| Chord | Both buttons, or middle click | FR-17 |

### 4.1 Revealing

- **FR-9** Left-clicking a `COVERED` cell reveals it. Left-clicking a `FLAGGED` or `REVEALED` cell does nothing (per FR-1, FR-2).
- **FR-10** Revealing resolves in exactly one of three ways:

**(a) The cell has a mine.** The game is lost immediately. Go to §6.2.

**(b) The cell has `adjacentMines ≥ 1`.** Reveal only that cell and display its number. Stop.

**(c) The cell has `adjacentMines == 0`.** Reveal it, then **recursively reveal all of its neighbours**. This is the single most important rule in the document.

- **FR-11 — Cascade / flood fill. [COMMONLY OMITTED]** When a cell with `adjacentMines == 0` is revealed, every neighbouring cell is also revealed, and this repeats outward from any newly revealed cell that is itself zero. The expansion **stops at numbered cells**: a cell with `adjacentMines ≥ 1` is revealed as part of the cascade but does not propagate further.

  Algorithm (breadth- or depth-first, both acceptable):

  ```
  reveal(cell):
      if cell.state != COVERED: return        # skips flags and already-revealed
      cell.state = REVEALED
      if cell.adjacentMines == 0:
          for n in neighbours(cell):
              reveal(n)
  ```

  Notes that make this correct rather than merely plausible:
  - The `state != COVERED` guard is what terminates the recursion. Without it the algorithm loops forever between adjacent zero-cells.
  - A cascade **must not** reveal flagged cells (FR-2). It flows around them.
  - A cascade can never reveal a mine, because a mine-containing cell always has a numbered neighbour standing between it and any zero-cell. This is a property of the rules, not something to enforce with an extra check.
  - On the largest supported board a single cascade may reveal several hundred cells. Use an explicit stack or queue if the language's recursion depth is a concern (NFR-3).

**Worked example of a cascade.** Player clicks (3,3), a zero-cell. Left: before. Right: after the single click.

```
before                        after
▓ ▓ ▓ ▓ ▓ ▓                   ▓ ▓ ▓ ▓ ▓ ▓
▓ ▓ ▓ ▓ ▓ ▓                   ▓ 1 1 1 1 1
▓ ▓ ▓ ▓ ▓ ▓                   ▓ 1 · · · ·
▓ ▓ ▓ ▓ ▓ ▓        ────►      ▓ 1 · · · ·
▓ ▓ ▓ ▓ ▓ ▓                   ▓ 1 · · · ·
▓ ▓ ▓ ▓ ▓ ▓                   ▓ 1 1 1 1 1

▓ = covered   · = revealed, zero adjacent   1 = revealed, one adjacent mine
```

One click opened twenty-nine cells. The ring of `1`s is where the expansion halted. This is what makes the game feel generous rather than tedious — without it, the player must click every single safe cell by hand and the game is unplayable. **A build that reveals only the clicked cell is not a slower version of this game; it is a different and worse one.**

### 4.2 First-click safety

- **FR-12 — [COMMONLY OMITTED]** The player's **first reveal of a game must never hit a mine**, and should open a cascade. Because mines are placed at random, this cannot be left to chance. Implement by **deferring mine placement until after the first click**:

  1. On new game, create the grid with no mines. Do not start the timer.
  2. On the first reveal at `(r, c)`: place all `mineCount` mines uniformly at random among cells **excluding `(r, c)` and its eight neighbours**.
  3. Compute `adjacentMines` for the whole board (FR-8).
  4. Now perform the reveal normally (FR-10). Because the clicked cell and its whole neighbourhood are mine-free, it is guaranteed to be a zero-cell, so the player always begins with a useful cascade rather than a bare `3`.

  The weaker variant — excluding only the clicked cell — is acceptable if the 3×3 exclusion would violate FR-5, but the 3×3 form is the requirement. Do **not** implement this by placing mines up front and re-rolling the board on a bad first click; re-rolling biases the distribution and is observably unfair over many games.

  Rationale: a loss on move one is unavoidable by any amount of skill. It teaches the player that the game is arbitrary, which is the opposite of the product's premise (§1.1). This rule is the single largest driver of SC-2.

### 4.3 Flagging

- **FR-13** Right-clicking cycles a cell `COVERED → FLAGGED → COVERED`. Right-clicking a `REVEALED` cell does nothing.
- **FR-14** Flags are **advisory**. Placing a flag on a cell with no mine is legal, produces no warning, and no feedback. The game must never confirm or deny a flag's correctness during play — doing so would hand the player the answer and destroy the puzzle. **[COMMONLY OMITTED]**
- **FR-15** The player may place more flags than there are mines. The mine counter (UX-3) simply goes negative.
- **FR-16** Right-click must not open the browser/OS context menu over the board.

### 4.4 Chording

- **FR-17 — [COMMONLY OMITTED]** Clicking a `REVEALED` numbered cell whose number **equals the count of flags among its neighbours** reveals all of its remaining `COVERED`, unflagged neighbours at once — each resolving normally per FR-10, cascades included.

  - If the flag count does not equal the number, nothing happens.
  - If any of those flags is wrong, this reveals a mine and **loses the game**. That is correct and intended: chording is a speed tool that trusts the player's own flags, and the risk is what makes it a real decision.
  - Chording is a convenience over repeated single clicks; it never reveals anything the player could not have revealed one click at a time.

  Example — the `2` has exactly two flagged neighbours, so clicking it opens the three remaining covered neighbours:

  ```
  before          after
  ⚑ ▓ ▓           ⚑ 1 ·
  ▓ 2 ▓    ───►   ▓ 2 ·
  ⚑ ▓ ▓           ⚑ 1 ·
  ```

---

## 5. Game state

- **FR-18** A game is in exactly one of four states:

| State | Entered when | Board interactive |
|---|---|---|
| `READY` | New game created; no cell revealed yet | Yes |
| `PLAYING` | First cell revealed (mines now placed, timer running) | Yes |
| `WON` | Win condition met (FR-19) | No |
| `LOST` | A mine was revealed | No |

- **FR-19 — Win condition. [COMMONLY OMITTED]** The game is won when **every cell without a mine is `REVEALED`**. Equivalently: `revealedCount == (width × height) − mineCount`.

  The win condition is **not** "every mine is flagged." Flags are advisory (FR-14) and play no part in it. A player may win having placed no flags at all, and must be able to. Checking flags instead is the most common way this rule is implemented wrongly, and it produces a game that cannot be won by a fast player and can be "won" by a lucky one.

- **FR-20** Evaluate the win condition after every reveal action, including after a cascade completes and after a chord completes — not only after single clicks.

### 5.1 On winning

- **FR-21** Stop the timer. Freeze the board. Auto-flag every remaining covered cell (all of which contain mines) as a cosmetic confirmation — this is display only and must happen *after* the win is determined, never as part of determining it. Show a win indicator (UX-5).

### 5.2 On losing

- **FR-22** Stop the timer. Freeze the board. Then reveal the full truth so the player can see what happened:
  - The mine that was clicked is shown distinctly from the others (e.g. a red background). The player needs to know *which* cell killed them.
  - All other mines are revealed as mines.
  - Every flag placed on a cell that does **not** contain a mine is marked as incorrect (conventionally a mine glyph struck through with an ✗). **[COMMONLY OMITTED]** This is the game's post-mortem: it shows the player exactly where their reasoning went wrong, which is the only teaching moment the product has.
  - Correct flags stay as flags.

---

## 6. Interface

### 6.1 Layout

```
┌────────────────────────────────────┐
│  ┌─────┐      ┌───┐      ┌─────┐   │
│  │ 010 │      │ ☺ │      │ 000 │   │   ← status bar
│  └─────┘      └───┘      └─────┘   │
│   mines      new game     timer    │
├────────────────────────────────────┤
│  ▓ ▓ ▓ ▓ ▓ ▓ ▓ ▓ ▓                 │
│  ▓ ▓ ▓ ▓ ▓ ▓ ▓ ▓ ▓                 │   ← board
│  ▓ ▓ ▓ ▓ ▓ ▓ ▓ ▓ ▓                 │
│  ▓ ▓ ▓ ▓ ▓ ▓ ▓ ▓ ▓                 │
└────────────────────────────────────┘
```

- **UX-1** The board is a grid of uniformly sized square cells. Cells must be visually distinct in their covered and revealed states — conventionally, covered cells are raised and revealed cells are flat/inset. The distinction must be legible at a glance across the whole board, since the player is constantly scanning for the covered/revealed boundary.
- **UX-2** Minimum cell hit target: 24×24 px. The game punishes misclicks with instant loss, so the target must be forgiving.

### 6.2 Status bar

- **UX-3 Mine counter** — displays `mineCount − (number of flags currently placed)`. It counts *flags*, not discovered mines; the game does not know what the player has discovered. It may go negative (FR-15). It does not tell the player whether their flags are right.
- **UX-4 Timer** — elapsed whole seconds. Starts on the **first reveal**, not on page load and not on new-game (FR-18). Stops on win or loss. Conventionally capped at 999 and displayed zero-padded to three digits.
- **UX-5 New game control** — a single always-available control that abandons the current game and starts a fresh one with the same settings. Conventionally a face that doubles as a state indicator: neutral while playing, sunglasses on win, dead on loss. The state indicator is decorative; the reset function is not.

### 6.3 Displaying a cell

- **UX-6** Rendering rules, in priority order:

| Condition | Display |
|---|---|
| `COVERED` | Blank covered tile |
| `FLAGGED` | Covered tile with flag glyph |
| `REVEALED`, `adjacentMines == 0` | Empty revealed tile — **no "0"** |
| `REVEALED`, `adjacentMines ≥ 1` | The digit, in its assigned colour |
| `REVEALED` mine (game over only) | Mine glyph |

- **UX-7 — [COMMONLY OMITTED]** A revealed cell with zero adjacent mines shows **nothing**, not the character `0`. Zero-cells are the visual negative space that lets the player read the board's shape; filling them with digits makes a solved region unreadable.
- **UX-8 — Number colours.** The digits 1–8 are colour-coded, and the specific palette is load-bearing: players scan for "the blue ones" faster than they read digits. Use the conventional assignment:

  | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
  |---|---|---|---|---|---|---|---|
  | blue | green | red | dark blue | dark red | teal | black | grey |

  Colour is an accelerator, never the sole carrier of meaning — the digit itself is always present, so the display remains fully usable without colour vision (NFR-5).

### 6.4 Instructions

- **UX-9** The rules must be reachable from the game without leaving it. First-time players cannot infer the flood fill, chording, or the win condition from the board alone. Minimum content: the goal, the three inputs, what the numbers mean, and the fact that the first click is always safe.

---

## 7. Non-functional requirements

- **NFR-1** Any single action resolves within 16 ms on the Expert board, including a worst-case cascade.
- **NFR-2** Custom boards up to 50×50 are supported without degradation.
- **NFR-3** Cascade implementation must not overflow the stack on a board where a single cascade reveals every safe cell (e.g. 50×50 with 1 mine). Prefer an explicit queue.
- **NFR-4** Full keyboard operation: arrow keys move a focus cursor, one key reveals, one key flags. The focused cell is visibly indicated.
- **NFR-5** The board is navigable and solvable without colour vision (UX-8) and exposes cell state to assistive technology (position, covered/flagged/revealed, adjacent count).
- **NFR-6** State is held in memory; no persistence is required in v1. A page refresh legitimately starts a new game.
- **NFR-7** The hidden board must not be discoverable through the rendered document — do not emit mine positions into the DOM, markup attributes, or class names for covered cells.

---

## 8. Non-goals

Stated explicitly so they are not implemented by reflex:

- **No difficulty scaling within a round.** Mine count is fixed at generation.
- **No hints, no undo, no lives.** A loss is final; the response is to start another round, which takes one click.
- **No guaranteed-solvable board generation in v1.** Some boards contain positions where no deduction determines the answer and the player must guess (§9).
- **No timing pressure.** The timer measures; it never constrains.
- **No touch-first design in v1.** Right-click and chord have no clean touch equivalents; solving that properly is its own scope.

---

## 9. Open questions

- **OQ-1 Guess-free boards.** Roughly 20–30% of Expert boards reach a position where two configurations are equally consistent with all visible information, forcing a coin flip near the end. Solvable-board generation exists (generate, solve with a constraint solver, regenerate on failure) but costs generation time and changes the board distribution. **Recommendation: ship without it, instrument how often players lose on a final guess, revisit.**
- **OQ-2 Question-mark state.** Some implementations add a third covered state (`?`) for "unsure", cycling `COVERED → FLAGGED → QUESTION → COVERED`. It is rarely used and it slows flag placement for experienced players, who then look for a setting to disable it. **Recommendation: omit.**
- **OQ-3 Timer cap.** 999 seconds is the convention, inherited from a three-digit display. Whether to keep the cap or count freely is a cosmetic decision.

---

## 10. Acceptance criteria

Each maps to the requirement it verifies. A build is complete when all pass.

| # | Test | Verifies |
|---|---|---|
| AC-1 | Generate 1,000 Beginner boards; each has exactly 10 mines, and mine frequency per cell is uniform within sampling error | FR-4 |
| AC-2 | Clicking the centre of a 3×3 mine-free region reveals the centre and all 8 neighbours | FR-11 |
| AC-3 | On a 10×10 board with 1 mine in a corner, the first click far from it reveals every safe cell in one action and wins | FR-11, FR-19 |
| AC-4 | A cascade reaching a flagged cell leaves it flagged and covered, and expands around it | FR-2, FR-11 |
| AC-5 | Across 500 new games, the first click never reveals a mine and always produces a cascade | FR-12 |
| AC-6 | Left-clicking a flagged cell does nothing, even when it has no mine | FR-2 |
| AC-7 | Revealing every safe cell with zero flags placed produces a win | FR-19 |
| AC-8 | Flagging every mine but leaving one safe cell covered does **not** produce a win | FR-19 |
| AC-9 | Chording a `2` with two adjacent flags reveals its other neighbours; chording it with one adjacent flag does nothing | FR-17 |
| AC-10 | Chording where a flag is misplaced reveals a mine and loses | FR-17 |
| AC-11 | On loss, the clicked mine is distinct, all other mines shown, all incorrect flags marked ✗ | FR-22 |
| AC-12 | Timer reads 0 until the first reveal, increments once per second thereafter, freezes on win and on loss | UX-4 |
| AC-13 | Mine counter reads `mineCount` at start, decrements per flag, increments per unflag, goes negative past `mineCount` flags | UX-3, FR-15 |
| AC-14 | Revealed zero-cells render empty; no `0` appears anywhere on the board | UX-7 |
| AC-15 | Custom input of 5×5 with 20 mines is rejected with a message; the game does not start | FR-5, FR-7 |
| AC-16 | A 50×50 board with 1 mine cascades fully without stack overflow, in under 16 ms | NFR-1, NFR-3 |
| AC-17 | Right-clicking the board never opens a context menu | FR-16 |
| AC-18 | Mine positions are absent from the rendered document for covered cells | NFR-7 |

---

## Appendix A: Reusing this as a template

The structure that makes this document buildable-from-zero, generalised:

1. **State the intent before the mechanics.** §1.1 says the game is about deduction, not reflex. Every later decision — first-click safety, no timing pressure, advisory flags — follows from that one sentence. An implementer who understands the intent resolves ambiguity the way you would. One that only has a rule list guesses.

2. **Define the vocabulary once, precisely, and then use it exactly.** §2 pins down "neighbour" as eight-way *with a diagram*. Every subsequent "adjacent" is now unambiguous. Most specification defects are vocabulary defects wearing a disguise.

3. **Mark the assumed knowledge.** The **[COMMONLY OMITTED]** tags flag the nine rules that a domain expert would never think to write down — flood fill, first-click safety, the real win condition, flags-block-reveals, no zeroes displayed, the loss post-mortem. **These are the whole reason the document works.** For your own domain, the test is: *what would I be annoyed to have to explain in review?* Write that down first; it is exactly what will be got wrong.

4. **Give worked examples with concrete data.** The cascade diagram in §4.1 conveys in twelve lines of ASCII what three paragraphs of prose leaves ambiguous. If a rule has a shape, draw the shape.

5. **Say why, not just what.** FR-12's rationale ("a loss on move one is unavoidable by any skill") lets an implementer who hits an edge case the spec missed reason from the same principle. Rules without reasons get followed literally into absurdity.

6. **Name the wrong implementations explicitly.** "The win condition is *not* 'every mine is flagged'." "Do not re-roll the board." Naming the attractive-but-wrong path is worth more than another paragraph describing the right one, because the wrong path is where the reader was already heading.

7. **Write non-goals down.** §8 prevents the plausible additions — hints, lives, difficulty ramping — that would each individually seem like an improvement and collectively be a different product.

8. **Separate open questions from requirements.** §9 marks the genuinely undecided, with a recommendation attached. Anything not in §9 is decided, which is what lets an implementer proceed without asking.

9. **Make every requirement testable, then write the tests as part of the spec.** §10 is the document's own definition of done. If a requirement can't be turned into a row of that table, it is a preference, not a requirement — either sharpen it or move it to non-goals.
