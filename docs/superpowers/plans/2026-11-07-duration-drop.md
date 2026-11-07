# Duration Drop Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a free, public, browser-based daily ordering puzzle in which players arrange five events by duration, scored exactly, with streaks kept in the browser.

**Architecture:** A static site with no backend. Pure JavaScript modules (scoring, date lookup, streaks, sharing) are tested with Node's built-in test runner. The browser layer (board, sound, app wiring) reads a single `puzzles.json` file and stores results in `localStorage`. No build step and no runtime dependencies.

**Tech Stack:** Plain HTML, CSS and ES modules; Web Audio API; Pointer Events and keyboard handling; Node.js 20+ with `node:test` for tests (dev-only).

**Spec:** `docs/superpowers/specs/2026-11-07-duration-drop-design.md`

## Global Constraints

- Five items per puzzle; 10 pairs per round; one point per correctly ordered pair.
- Equal durations count as correct in either order.
- Dates are keys in the form `YYYY-MM-DD`, using the player's local calendar date.
- Share line format: `Duration Drop, Nov 8: 8/10` (month abbreviation, day without leading zero).
- No accounts, no server calls, no personal data stored.
- No runtime dependencies. Dev tooling is limited to Node's built-in test runner.
- Sound is generated with the Web Audio API; no audio files.

## Review Focus

1. A puzzle with two equal durations must not penalize either order. Covered by Task 2.
2. A duration of zero or a negative number must be rejected by validation, not shown to players. Covered by Task 1.
3. Corrupt or non-JSON data in browser storage must not crash the page; it is treated as no saved state. Covered by Task 4.
4. A player who reloads after locking in must see the same results, not a fresh round. Covered by Task 6.
5. A duration under one second must display in a readable unit, not as "0 seconds". Covered by Task 2.

---

### Task 1: Puzzle data and validation

**Files:**
- Create: `puzzles.json`
- Create: `scripts/validate-puzzles.mjs`
- Create: `src/puzzle-check.js`
- Test: `tests/puzzle-check.test.js`

**Interfaces:**
- Produces: `validatePuzzles(puzzles: object) -> string[]` in `src/puzzle-check.js`. Returns a list of error messages; an empty list means valid. Checks: each key is a real calendar date in `YYYY-MM-DD`, `items` has exactly 5 entries, each item has a non-empty `name` and a `seconds` number greater than 0, and `note` is optional and a string when present.
- `scripts/validate-puzzles.mjs` reads `puzzles.json`, calls `validatePuzzles`, prints each error, and exits with code 1 if any exist.

- [ ] **Step 1: Write the failing tests**

```javascript
// tests/puzzle-check.test.js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { validatePuzzles } from '../src/puzzle-check.js';

const good = {
  '2026-11-08': {
    items: [
      { name: 'A blink', seconds: 0.3 },
      { name: 'A sneeze', seconds: 1 },
      { name: 'A day on Venus', seconds: 20997600 },
      { name: 'A flight across the Atlantic', seconds: 25200 },
      { name: 'A sunrise on Mars', seconds: 88620 },
    ],
    note: 'Venus takes longer to turn than to orbit the Sun.',
  },
};

test('a well-formed puzzle has no errors', () => {
  assert.deepEqual(validatePuzzles(good), []);
});

test('a puzzle with four items is rejected', () => {
  const bad = structuredClone(good);
  bad['2026-11-08'].items.pop();
  assert.equal(validatePuzzles(bad).length, 1);
});

test('a zero or negative duration is rejected', () => {
  const bad = structuredClone(good);
  bad['2026-11-08'].items[0].seconds = 0;
  bad['2026-11-08'].items[1].seconds = -5;
  assert.equal(validatePuzzles(bad).length, 2);
});

test('a date that is not a real calendar day is rejected', () => {
  const bad = { '2026-02-30': good['2026-11-08'] };
  assert.equal(validatePuzzles(bad).length, 1);
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `node --test tests/puzzle-check.test.js`
Expected: FAIL with "Cannot find module '../src/puzzle-check.js'"

- [ ] **Step 3: Implement `validatePuzzles` in `src/puzzle-check.js`**

Use `Date` with a round-trip check on the `YYYY-MM-DD` string to reject impossible dates such as February 30. Return one message per problem, naming the date key.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `node --test tests/puzzle-check.test.js`
Expected: PASS, 4 tests

- [ ] **Step 5: Add the initial `puzzles.json` and the validation script**

Create `puzzles.json` containing the single example day from the spec. Implement `scripts/validate-puzzles.mjs` as described in Interfaces.

Run: `node scripts/validate-puzzles.mjs`
Expected: no output, exit code 0

- [ ] **Step 6: Commit**

```bash
git add puzzles.json scripts/validate-puzzles.mjs src/puzzle-check.js tests/puzzle-check.test.js
git commit -m "feat: add puzzle data format and validation"
```

---

### Task 2: Scoring and friendly durations

**Files:**
- Create: `src/scoring.js`
- Test: `tests/scoring.test.js`

**Interfaces:**
- Produces: `scoreOrder(order: Item[]) -> number`, where `Item = { name: string, seconds: number }` and `order` is the player's arrangement from first to last. Returns the count of correct pairs out of 10. A pair `(i, j)` with `i < j` is correct when `order[i].seconds <= order[j].seconds`, which makes equal durations correct in either order.
- Produces: `formatDuration(seconds: number) -> string`. Under one second, show the value with up to two decimals and the unit "seconds" (for example "0.3 seconds"). From one second up to 120 seconds, show the number of seconds ("1 second", "45 seconds"). From 2 minutes to 2 hours, show whole minutes. From 2 hours to 2 days, show "about N hours". Above that, show "about N days". Use the singular form when the number is 1.

- [ ] **Step 1: Write the failing tests**

```javascript
// tests/scoring.test.js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { scoreOrder, formatDuration } from '../src/scoring.js';

const item = (name, seconds) => ({ name, seconds });

test('a perfectly ordered round scores 10', () => {
  const order = [item('a', 1), item('b', 2), item('c', 3), item('d', 4), item('e', 5)];
  assert.equal(scoreOrder(order), 10);
});

test('a fully reversed round scores 0', () => {
  const order = [item('a', 5), item('b', 4), item('c', 3), item('d', 2), item('e', 1)];
  assert.equal(scoreOrder(order), 0);
});

test('equal durations count as correct in either order', () => {
  const forward = [item('a', 2), item('b', 2), item('c', 3), item('d', 4), item('e', 5)];
  const backward = [item('b', 2), item('a', 2), item('c', 3), item('d', 4), item('e', 5)];
  assert.equal(scoreOrder(forward), 10);
  assert.equal(scoreOrder(backward), 10);
});

test('one swapped neighbour loses exactly one point', () => {
  const order = [item('a', 1), item('c', 3), item('b', 2), item('d', 4), item('e', 5)];
  assert.equal(scoreOrder(order), 9);
});

test('a sub-second duration shows decimals and seconds', () => {
  assert.equal(formatDuration(0.3), '0.3 seconds');
});

test('one second uses the singular', () => {
  assert.equal(formatDuration(1), '1 second');
});

test('a day on Venus shows as days', () => {
  assert.equal(formatDuration(20997600), 'about 243 days');
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `node --test tests/scoring.test.js`
Expected: FAIL with "Cannot find module '../src/scoring.js'"

- [ ] **Step 3: Implement `scoreOrder` and `formatDuration` in `src/scoring.js`**

Count pairs with a nested loop over positions. For `formatDuration`, follow the unit bands in Interfaces and round to the nearest whole unit above one second.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `node --test tests/scoring.test.js`
Expected: PASS, 7 tests

- [ ] **Step 5: Commit**

```bash
git add src/scoring.js tests/scoring.test.js
git commit -m "feat: add scoring and friendly duration formatting"
```

---

### Task 3: Daily picker

**Files:**
- Create: `src/daily.js`
- Test: `tests/daily.test.js`

**Interfaces:**
- Produces: `dayKey(date: Date) -> string`. Returns the local calendar date as `YYYY-MM-DD`, built from `getFullYear()`, `getMonth()` and `getDate()` (never from `toISOString()`, which uses UTC).
- Produces: `puzzleFor(puzzles: object, key: string) -> { key, items, note } | null`.
- Produces: `nextPuzzleKey(puzzles: object, key: string) -> string | null`. Returns the earliest key in `puzzles` that is after `key`, or null if none.
- Produces: `orderedKeys(puzzles: object) -> string[]`, sorted ascending.

- [ ] **Step 1: Write the failing tests**

```javascript
// tests/daily.test.js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { dayKey, puzzleFor, nextPuzzleKey } from '../src/daily.js';

const puzzles = { '2026-11-08': { items: [], note: 'x' }, '2026-11-10': { items: [] } };

test('dayKey uses the local date, not UTC', () => {
  const lateEvening = new Date(2026, 10, 7, 23, 30); // local 23:30 on 7 Nov
  assert.equal(dayKey(lateEvening), '2026-11-07');
});

test('dayKey pads month and day', () => {
  assert.equal(dayKey(new Date(2026, 0, 5)), '2026-01-05');
});

test('puzzleFor returns the entry for a known key', () => {
  assert.equal(puzzleFor(puzzles, '2026-11-08').key, '2026-11-08');
});

test('puzzleFor returns null when the day has no puzzle', () => {
  assert.equal(puzzleFor(puzzles, '2026-11-09'), null);
});

test('nextPuzzleKey skips the missing day', () => {
  assert.equal(nextPuzzleKey(puzzles, '2026-11-09'), '2026-11-10');
});

test('nextPuzzleKey returns null after the last puzzle', () => {
  assert.equal(nextPuzzleKey(puzzles, '2026-11-10'), null);
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `node --test tests/daily.test.js`
Expected: FAIL with "Cannot find module '../src/daily.js'"

- [ ] **Step 3: Implement the four functions in `src/daily.js`**

Compare keys as strings; `YYYY-MM-DD` sorts correctly that way.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `node --test tests/daily.test.js`
Expected: PASS, 6 tests

- [ ] **Step 5: Commit**

```bash
git add src/daily.js tests/daily.test.js
git commit -m "feat: add daily puzzle lookup using local dates"
```

---

### Task 4: Storage and streaks

**Files:**
- Create: `src/storage.js`
- Create: `src/streak.js`
- Test: `tests/streak.test.js`

**Interfaces:**
- Produces in `src/storage.js`: `loadState(store: { getItem(k), setItem(k, v) }) -> State`, and `saveState(store, state) -> boolean`. `State = { results: { [key]: { score: number, order: string[] } }, streak: number, lastKey: string | null, isMuted: boolean }`. `loadState` returns the empty state for missing keys, corrupt JSON, or any read error. `saveState` returns false on any write error and never throws. The `store` argument is `window.localStorage` in the browser; tests pass a fake object.
- Produces in `src/streak.js`: `recordResult(state: State, key: string, score: number, order: string[]) -> State`. Returns a new state. If `key` already has a result, the state is returned unchanged. The streak becomes `state.streak + 1` when `lastKey` is the calendar day immediately before `key`, otherwise `1`. `lastKey` becomes `key`.
- Date arithmetic for "the day before" uses `Date.UTC` on the parsed key components, so it is not affected by daylight saving time.

- [ ] **Step 1: Write the failing tests**

```javascript
// tests/streak.test.js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { recordResult } from '../src/streak.js';
import { loadState, saveState } from '../src/storage.js';

const empty = { results: {}, streak: 0, lastKey: null, isMuted: false };

test('the first result starts a streak at 1', () => {
  const s = recordResult(empty, '2026-11-08', 8, ['a']);
  assert.equal(s.streak, 1);
  assert.equal(s.lastKey, '2026-11-08');
});

test('a consecutive day extends the streak', () => {
  const s1 = recordResult(empty, '2026-11-08', 8, ['a']);
  const s2 = recordResult(s1, '2026-11-09', 9, ['a']);
  assert.equal(s2.streak, 2);
});

test('a skipped day resets the streak to 1', () => {
  const s1 = recordResult(empty, '2026-11-08', 8, ['a']);
  const s2 = recordResult(s1, '2026-11-10', 9, ['a']);
  assert.equal(s2.streak, 1);
});

test('a month boundary still counts as consecutive', () => {
  const s1 = recordResult(empty, '2026-10-31', 8, ['a']);
  const s2 = recordResult(s1, '2026-11-01', 9, ['a']);
  assert.equal(s2.streak, 2);
});

test('a second result for the same day changes nothing', () => {
  const s1 = recordResult(empty, '2026-11-08', 8, ['a']);
  assert.equal(recordResult(s1, '2026-11-08', 10, ['b']), s1);
});

test('corrupt saved data loads as the empty state', () => {
  const broken = { getItem: () => '{not json', setItem: () => {} };
  assert.deepEqual(loadState(broken), empty);
});

test('a blocked store loads as empty and fails to save without throwing', () => {
  const blocked = {
    getItem: () => { throw new Error('denied'); },
    setItem: () => { throw new Error('denied'); },
  };
  assert.deepEqual(loadState(blocked), empty);
  assert.equal(saveState(blocked, empty), false);
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `node --test tests/streak.test.js`
Expected: FAIL with "Cannot find module '../src/streak.js'"

- [ ] **Step 3: Implement `src/storage.js` and `src/streak.js`**

Store the whole state under a single key, `duration-drop-state`. Wrap every read and write in `try/catch`. Validate the parsed object's shape before returning it; fall back to the empty state if any field has the wrong type.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `node --test tests/streak.test.js`
Expected: PASS, 7 tests

- [ ] **Step 5: Commit**

```bash
git add src/storage.js src/streak.js tests/streak.test.js
git commit -m "feat: add browser storage and streak tracking"
```

---

### Task 5: Share text

**Files:**
- Create: `src/share.js`
- Test: `tests/share.test.js`

**Interfaces:**
- Produces: `shareText(key: string, score: number) -> string`. Returns `Duration Drop, <Mon> <day>: <score>/10`, where `<Mon>` is the three-letter English month abbreviation and `<day>` has no leading zero. Example: `shareText('2026-11-08', 8)` returns `Duration Drop, Nov 8: 8/10`.

- [ ] **Step 1: Write the failing test**

```javascript
// tests/share.test.js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { shareText } from '../src/share.js';

test('the share line matches the spec format', () => {
  assert.equal(shareText('2026-11-08', 8), 'Duration Drop, Nov 8: 8/10');
});

test('a single-digit day has no leading zero', () => {
  assert.equal(shareText('2026-01-05', 10), 'Duration Drop, Jan 5: 10/10');
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `node --test tests/share.test.js`
Expected: FAIL with "Cannot find module '../src/share.js'"

- [ ] **Step 3: Implement `shareText` in `src/share.js`**

Use a fixed array of month abbreviations rather than `toLocaleString`, so the output does not depend on the player's browser language.

- [ ] **Step 4: Run the test to verify it passes**

Run: `node --test tests/share.test.js`
Expected: PASS, 2 tests

- [ ] **Step 5: Commit**

```bash
git add src/share.js tests/share.test.js
git commit -m "feat: add share line"
```

---

### Task 6: Board, drag and keyboard, lock-in

**Files:**
- Create: `index.html`
- Create: `styles.css`
- Create: `src/board.js`
- Test: manual check in Task 9

**Interfaces:**
- Consumes: `scoreOrder`, `formatDuration` from Task 2; `puzzleFor` from Task 3; `loadState`, `saveState` and `recordResult` from Task 4.
- Produces: `createBoard(root: HTMLElement, options: { items: Item[], initialOrder: string[], onLockIn: (order: string[]) => void, playSound: (kind: 'pickup'|'slot'|'drop') => void, isLocked: boolean })`. Renders the cards and handles pointer drag and keyboard moves. Keyboard: a focused card is selected with Enter or Space; ArrowUp and ArrowDown move the selected card one slot, and each move calls `playSound('slot')`.
- Dragging uses Pointer Events. The dragged card follows the pointer, the others shift by one slot to show the landing spot, and on release the card animates to its slot with a short CSS transition. When `prefers-reduced-motion: reduce` matches, all transitions are removed via a `reduced` class on the root element.
- Initial order: the day's `items` in the order stored in `puzzles.json`, shuffled once with a seeded shuffle keyed on the day's date string, so the first view is the same for every player.
- Locked state: the board is read-only and shows the results section instead of the Lock in button.

- [ ] **Step 1: Write the page skeleton**

Create `index.html` with one `<main id="app">`, a `<section id="board">`, a `<button id="lock-in">Lock in</button>`, a mute toggle `<button id="mute" aria-pressed="false">Sound on</button>`, and `<script type="module" src="src/app.js"></script>`. Add `styles.css` with the `.reduced` rules and the card lift and shadow styles.

- [ ] **Step 2: Implement `createBoard` in `src/board.js`**

Build one `<li role="listitem" tabindex="0">` per card inside a `<ol>`. Each card shows the item name only; durations are hidden until the reveal. Use `aria-live="polite"` on a status line that announces each move, for example "A blink moved to position 2 of 5."

- [ ] **Step 3: Add the sound hook calls**

`createBoard` calls `options.playSound` at the three moments in the spec: pickup, each slot move, and drop. It does not create any audio itself; that lives in Task 7.

- [ ] **Step 4: Verify in the browser**

Serve the folder with `npx serve .` (or any static server), open the page, and check: a card can be dragged with the mouse, a card can be moved with the keyboard, the Lock in button fires `onLockIn` with the final order, and the board is read-only after lock-in.

Expected: all four behaviors work with no console errors.

- [ ] **Step 5: Commit**

```bash
git add index.html styles.css src/board.js
git commit -m "feat: add draggable board with keyboard support"
```

---

### Task 7: Sound

**Files:**
- Create: `src/sound.js`

**Interfaces:**
- Produces: `createSound(isMuted: () => boolean) -> { unlock(): void, play(kind: 'pickup'|'slot'|'drop'): void }`.
- `unlock()` creates the `AudioContext` on the first user gesture. Calls to `play` before `unlock` are ignored, so no sound plays until the player has interacted with the page.
- `play` produces a short tone with the Web Audio API: `pickup` is a quiet high tick (about 900 Hz, 40 ms), `slot` a soft click (about 500 Hz, 30 ms), `drop` a low thunk (about 220 Hz, 120 ms, with a fast decay). Does nothing when `isMuted()` returns true.

- [ ] **Step 1: Implement `src/sound.js`**

Use an `OscillatorNode` and `GainNode` per sound, with an exponential ramp to near zero at the end to avoid clicks. Keep the gain low (about 0.15).

- [ ] **Step 2: Verify in the browser**

On the page from Task 6, interact with a card and listen. Confirm each of the three sounds is distinct, and that nothing plays before the first tap.

Expected: three distinct sounds on pickup, move and drop; silence before the first tap; silence when muted.

- [ ] **Step 3: Commit**

```bash
git add src/sound.js
git commit -m "feat: add generated interaction sounds"
```

---

### Task 8: App wiring, results and edge cases

**Files:**
- Create: `src/app.js`
- Modify: `index.html` (add results and message containers)

**Interfaces:**
- Consumes: everything from Tasks 2 to 7.
- `app.js` reads `puzzles.json`, runs `validatePuzzles` and logs any errors to the console without blocking play, then picks today's puzzle with `dayKey(new Date())`.
- No puzzle today: shows "No puzzle today. The next one is on <formatted date>." using `nextPuzzleKey`, and does not render the board.
- Already played today (`state.results[key]` exists): renders the board read-only with the saved order, and shows the saved score and note.
- Lock in: computes `scoreOrder`, calls `recordResult`, calls `saveState`, reveals each item's `formatDuration` value, shows the note, the score out of 10, and a Copy button that writes `shareText(key, score)` to the clipboard.
- Storage blocked (`saveState` returns false): show "Your streak can't be saved in this browser." once per page load.
- Mute button toggles `state.isMuted`, saves it, and updates the button's `aria-pressed`.

- [ ] **Step 1: Implement `src/app.js`**

Wire the modules together in the order: load state, load and validate puzzles, pick today's puzzle, then render either the board, the already-played view, or the no-puzzle message.

- [ ] **Step 2: Verify the four paths in the browser**

Use the browser's developer tools to change the system date, or temporarily add entries to `puzzles.json` for today and tomorrow. Check: the normal round, the already-played reload, the no-puzzle message, and the blocked-storage message (block site data in the browser settings).

Expected: each path shows the right screen with no console errors.

- [ ] **Step 3: Commit**

```bash
git add src/app.js index.html
git commit -m "feat: wire daily round, results and edge cases"
```

---

### Task 9: Publish checklist and manual check

**Files:**
- Create: `README.md`

**Interfaces:**
- None; this task checks the finished game against the spec.

- [ ] **Step 1: Run every automated test**

Run: `node --test tests/`
Expected: all tests pass (4 + 7 + 6 + 7 + 2 = 26 tests)

- [ ] **Step 2: Run the puzzle validation**

Run: `node scripts/validate-puzzles.mjs`
Expected: no output, exit code 0

- [ ] **Step 3: Manual check on a phone and a desktop browser**

Check drag, keyboard, sound, reduced-motion (in the operating system's accessibility settings), results, the share copy button, and the streak across two days (change the system date to simulate).

- [ ] **Step 4: Write `README.md`**

Explain how to run the tests and the validation, and how to add a new day's puzzle to `puzzles.json`.

- [ ] **Step 5: Commit**

```bash
git add README.md
git commit -m "docs: add README and publish checklist"
```

---

## Self-Review Notes

- Spec coverage: puzzle file and validation (Task 1), scoring and friendly durations (Task 2), daily picker with local dates (Task 3), storage, streaks and blocked-storage handling (Task 4), share text (Task 5), drag/keyboard board, lock-in and fixed daily start order (Task 6), sounds with first-interaction rule and mute (Task 7), results, no-puzzle and already-played paths (Task 8), tests and publish checks (Task 9). Hosting and publishing workflow are operational and are covered in the README.
- Review Focus items each have a test or a verification step in the owning task.
