# Duration Drop: Design

Status: approved.

## Summary

Duration Drop is a free, public, browser-based daily puzzle. Each day, a player sees five everyday events or things and puts them in order from shortest to longest duration. The game reveals the real durations and scores the round out of 10 (each correctly ordered pair is one point). The goal is a daily habit for players of all ages.

## Confirmed decisions

- Public website, no accounts, no sign-in, no server-side storage.
- Curated daily schedule: a reviewed batch of puzzles, about a month at a time.
- Launch category: Duration Drop (events ordered by how long they take).
- Streaks and results are saved in each player's browser only.

## Architecture

A static website with no backend:

1. **Puzzle file** (`puzzles.json`): one entry per day, keyed by `YYYY-MM-DD`.
2. **Daily picker**: reads the player's local date and shows that day's puzzle.
3. **Game board**: five cards the player reorders by drag or keyboard, then locks in.
4. **Results and streaks**: reveal, score, share text, and streak tracking, stored in browser storage.

Assumptions:
- A new puzzle appears at local midnight, so each player sees the day matching their own clock.
- Each day's puzzle is the same for everyone in the same time zone.

## Puzzle data

Each entry has five `items` (`name`, `seconds`) and an optional `note` shown after the reveal.

```json
{
  "2026-11-08": {
    "items": [
      { "name": "A blink", "seconds": 0.3 },
      { "name": "A sneeze", "seconds": 1 },
      { "name": "A day on Venus", "seconds": 20997600 },
      { "name": "A flight across the Atlantic", "seconds": 25200 },
      { "name": "A sunrise on Mars", "seconds": 88620 }
    ],
    "note": "Venus takes longer to turn than to orbit the Sun."
  }
}
```

Validation (run before every publish): exactly five items, all durations positive numbers, dates are real calendar days, no duplicate dates.

## Scoring

- Five cards produce 10 pairs; each correctly ordered pair scores one point.
- Pairs with equal durations count as correct in either order.
- After the reveal, each card shows its duration in friendly units (for example, "0.3 seconds" or "about 7 hours").

## Gameplay and feedback

- Cards start in a fixed order per day, so every player's first view matches.
- Drag on touch and mouse; arrow keys on keyboard.
- Dragging lifts the card with a shadow, the others slide to show the landing spot, and the drop snaps with a small settle motion.
- Generated sounds (Web Audio, no audio files): a tick on pickup, a click on each slot move, a thunk on drop. Sound starts after the first user interaction.
- A mute toggle in the corner, remembered in browser storage.
- With reduced-motion set on the device, lift and slide animations are removed; sounds still play.
- A round locks in once. Reloading after lock-in shows the same results.

## Results and sharing

- Results reveal the real durations and the score out of 10, plus the day's note.
- A share line such as "Duration Drop, Nov 8: 8/10," copied with one tap.

## Streaks

- A streak counts consecutive local calendar days with a completed round.
- Missing a day resets the streak to 1 when the player next plays. No grace days.

## Edge cases

- No puzzle today: the page says so and shows the date of the next puzzle.
- Browser storage blocked: the game still works; the streak does not persist; the page says so once.
- All controls have text labels and keyboard support; the layout works on small screens.

## Hosting

- Plain static files on a free static host (GitHub Pages, Netlify or Cloudflare Pages); the host is chosen at publish time.
- No server, database or accounts.

## Publishing

- Puzzles are written in monthly batches and reviewed before going live.
- Fixes to live puzzles are redeployed; players keep their saved results for that day.

## Testing

- Automated tests for scoring, date lookup and streak logic.
- The validation script runs before every publish.
- Manual check of drag, sound and results on a phone and a desktop browser before launch.

## Out of scope (first version)

- Accounts, cross-device sync, leaderboards, or any server-side data.
- Categories other than Duration Drop.
- Ads, payments or user-submitted puzzles.

## Next step

After review of this written spec, the next step is to create the implementation plan with the writing-plans skill.
