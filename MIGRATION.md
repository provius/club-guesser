# Club Guesser migration

Status: draft 2, 18 November 2026, owner decisions applied
Platform API: 1
Registry entry: id `club-guesser` in interactiv-games `docs/registry/games.json`
Written against interactiv-games `docs/PLATFORM-SPEC.md` draft 1. Spec changes needed are listed in interactiv-games `docs/SPEC-FEEDBACK.md`.

Read for this draft: README.md, `functions/index.js`, `functions/src/api.js`, `functions/src/game.js`, `firestore.rules`, `firebase.json` and the file list. Not read in full: `scoring.js`, `public/app.js`, `scripts/build-dataset.js`, the tests.

## Owner decisions, 18 November 2026

- Quiz scoring shape, S7 and O16: a game can offer one free play each day that earns points, and further plays cost points. For this game a play is a game of rounds. The payout, the cost of an extra play and any earn cap stay null until decided.

## 1. Summary and target model and tier

Players from one club are revealed one at a time and the player guesses the club, with a score that falls as hints, time and wrong guesses add up.

- Session model: `settle-once`.
- Tier: Functions.
- Exclusive: `true`, the default. Rounds are short, with a 30 minute round time to live.
- Front end: plain web UI.

The game is already on the right side of the platform's main rule. Rounds are stored in Firestore with the answer and start time, and clients cannot read Firestore at all.

## 2. What stays as it is

- Round building and the hint ordering in `functions/src/game.js`.
- Pure scoring in `functions/src/scoring.js`, and the test that fails if the browser copy drifts from it. The browser copy is for the live points available display only and never decides a result.
- The deny-all Firestore rules.
- The dataset build script and the three difficulty levels.
- The scoring formula in the README. It is a game design and stays with the game.

## 3. What is rewritten

- **Transport.** `functions/index.js` uses a hand written router over `onRequest`, anonymous, with Hosting rewriting `/api/**`. Replace it with callable functions that require the platform user and the `over18` claim, like the rest of the suite.
- **Identity.** Rounds have no owner today. Every round must carry the user id so a user can only act on their own rounds.
- **Anti repeat.** The client sends an `exclude` list of clubs already seen. That is client supplied state. The server should keep the seen clubs per user instead.
- **Region.** The function runs in us-central1. The platform project is europe-west2.
- **Data location.** Rounds move under `games/club-guesser/`.
- **Hosting.** `public/` becomes a game module.

## 4. What moves server side

- The seen clubs list, from the client to the server.
- Opening and closing the platform session. One session covers a game of rounds. The number of rounds in a game is a game design number and is null for now.
- The mapping from score to platform points. Not decided. The formula gives up to 1000 times factors per round, which is a game score and not a points amount. See S7 and O16. The payout stays null until decided.
- Timing already uses the server start time. Keep it, and read the server clock only.

## 5. Data to carry over or drop

Assume no real user data, O7.

- `rounds` in Firestore are short lived, with an optional time to live policy. Drop them.
- The dataset is the important part. The repo currently ships a mock set of 19 clubs and about 90 players. `build-dataset.js` has not been run against a live Sportmonks token. Building the real dataset is a prerequisite for beta and is separate from the migration.
- Sportmonks has no player fame rating, which drives hint order and difficulty. The default is 3 for everyone, with a per player override file. This needs a real source before the difficulty levels mean much.

## 6. Front end plan

Plain web UI. The browser app is a small vanilla script of about 105 lines. Rebuild it as a game module with the shell providing the points display, errors and dialogs, and use design tokens once real tokens land. No Phaser. Easy mode is multiple choice and the others are typed, so DOM input is the right fit.

## 7. Manifest draft

```json
{
  "platformApi": 1,
  "id": "club-guesser",
  "name": "Club Guesser",
  "version": "0.1.0",
  "status": "draft",
  "kind": "dom",
  "entry": "/g/club-guesser/",
  "minAgeLevel": 1,
  "sessionModel": "settle-once",
  "exclusive": true,
  "stake": { "min": null, "max": null },
  "caps": { "maxWinPoints": null, "maxWinMultiple": null, "maxActionsPerSecond": null },
  "service": { "tier": "functions", "name": "clubGuesser", "recoveryHook": "clubGuesserRecover" },
  "dataNamespace": "games/club-guesser",
  "leakTests": true
}
```

## 8. Game ledger design and recovery hook

- One ledger per session. Entries: round start with the club id kept server only, each hint revealed, each guess with its result, each skip, and the round end with the score.
- The hash chain wraps each entry. The ledger holds the answer, so it is under a server only path.
- Recovery hook `clubGuesserRecover`: ends any open round as unanswered, settles the session on the score so far and never refunds.

## 9. Tests to add

- Keep the existing game tests and the scoring drift test.
- Leak tests: no round response contains the target club id or name before the round ends, including hints, errors and the config response.
- Rules tests for the namespaced paths.
- Abuse tests: act on another user's round, replay a guess, reveal hints faster than the client could, send a client crafted exclude list, open two sessions.
- Recovery test for an abandoned round.

## 10. Steps in order

1. Build the real dataset and settle the fame source. This can start now.
2. After Phase 1, move the function and data code into the platform repo.
3. Replace the router with callables and add the user id to rounds.
4. Add session open and close and the game ledger wrapper.
5. Rebuild the browser app as a game module.
6. Run the tests above and promote to `beta`.

## 11. Risks and open questions

- Score to points mapping and earn caps, O16.
- The number of rounds in a game and the lease length, S1.
- Fame ratings are a guess today. A weak difficulty curve makes the points inflow unpredictable.
- Easy mode shows four choices from elite clubs, so a bot with the dataset can win it. Rate limits and timing floors are the only defence. Accept and monitor.
