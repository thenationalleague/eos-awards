# MOTM widget — handover

Brief catch-up so a fresh session can pick this up clean. Pair this with
`widgethandover.md` (the family-level invariants doc) — that's the "why
the shell exists"; this doc is "what's done, what's left, what we decided".

## Where we are

- **MOTM widget** built at `embeds/motm.html` (originally in
  `thenationalleague/eos-awards`, moved to `tools/embeds/motm.html`).
- **Firebase project: `nl-widgets`** is the new shared home for the
  whole NL widget family. RTDB live, Anonymous Auth on, authorized
  domains added, rules deployed from
  `embeds/nl-widgets.rules.json` (covers `users/`, `predictions/`,
  `motm/`).
- **MOTM is live on a test page** and confirmed working signed-in and
  signed-out.
- **Score Predictor still points at the old `nl-score-predictor`
  project.** Migration is the next job (one config swap; data is a
  deliberate fresh start, not migrated).

## Pending work

1. **Swap `FIREBASE_CONFIG` in `score-predictor.html`** to `nl-widgets`
   (snippet below). Leave `APP_NAME = 'nlPredictor'` alone — the
   named-app guard is what keeps the two widgets from clashing on a
   page that embeds both. Re-paste into the CMS, verify signed-in
   registration + a test prediction land in the `nl-widgets` RTDB
   console.
2. **Delete the old `nl-score-predictor` project** after ~a week of the
   predictor running cleanly against `nl-widgets`. Console → Project
   settings → General → Delete project. 30-day grace period before
   permanent. Optional: export the RTDB JSON beforehand as a backup.

## Decisions made (don't relitigate)

- **Voting window: KO + 2.5h → KO + 24h.** Constants:
  `VOTE_OPEN_MIN = 150`, `VOTE_CLOSE_MIN = 1440`. Match states are
  `pre` → `live` → `voting` → `closed` (plus `postponed` / `abandoned`).
- **Only own-team matches are votable.** The matchday navigator is
  filtered to dates where the user's registered team plays. Off-team
  matches simply don't appear — no "this isn't your match" filler.
- **Votes can go to either side.** Player picker groups players by team
  via `<optgroup>`, starters before subs, with shirt number prefix.
  Fan can vote for an opposition player.
- **Registration UX is two-path:**
  - JWT `favourite_team` matches a current National / North / South
    club → **confirm-only** card showing team crest + name + single
    button. Caption links to `https://signin.thenationalleague.org.uk/`
    for fans who want to change their team (they update it there, then
    sign out / sign back in to refresh the JWT claim).
  - No match (empty claim, or a Premier League team etc.) → full
    grouped dropdown fallback so they can still play this season.
- **Tally is deliberately count-less.** Top 5 players ranked by votes
  (sorted internally), no numeric counts, no bars. "Leading" pill on #1
  **only** when there's a strictly clear leader — ties at the top get
  no pill, on purpose, to avoid "everyone's leading with 1 vote each"
  during low turnout. "Your pick" pill highlights the user's choice.
  Header reads "Man of the Match — Live" during the window and "Man of
  the Match — Final" after close.
- **`users/` tree is shared between predictor and MOTM.** A fan
  registers once, in whichever widget they hit first, and that team
  applies to both. The rule `auth != null && !data.exists()` makes it
  immutable. Intended behaviour per the family handover.
- **Players are lazy-fetched** from `/v2/matches/{matchId}` per match
  when its state enters `voting` / `closed` / `live`. Cached in
  `state.players[matchId]`. Player name resolution chain:
  `knownName → customKnownName → firstName + ' ' + lastName`. Subs use
  `playerSubPosition` when `playerPosition === 'Substitute'`. See
  `NLSMatchDataMOTMReference.md` for the API shape and gotchas.
- **RTDB write shape:** `motm/{jwtId}/{matchId} → { playerId,
  playerName, teamId, submittedAt }`. Re-voting is allowed and just
  overwrites — no "vote history" tracked.

## Firebase config (nl-widgets)

```js
var FIREBASE_CONFIG = {
  apiKey:            'AIzaSyAOePUiyfACJ546b08Z7oGWahAEYzEadMo',
  authDomain:        'nl-widgets.firebaseapp.com',
  databaseURL:       'https://nl-widgets-default-rtdb.europe-west1.firebasedatabase.app',
  projectId:         'nl-widgets',
  storageBucket:     'nl-widgets.firebasestorage.app',
  messagingSenderId: '440054238126',
  appId:             '1:440054238126:web:349a1aeaf3c65ff281563f'
};
```

Rules in `embeds/nl-widgets.rules.json` — covers all three subtrees.
Per-`$matchId` write granularity on `predictions/` and `motm/` (so any
reset / overwrite must hit individual children, not the parent — the
predictor already does this).

## Test plan (when re-verifying)

Use the sim datetime bar at the bottom of each widget — it's the
fastest way to simulate the lifecycle.

MOTM:
- **Pre-KO:** match shows kick-off time + "Voting opens HH:MM".
- **KO → KO+2.5h:** "Live" pulse dot, "Voting opens in Xm".
- **KO+2.5h → KO+24h:** player picker enabled. Pick from either side,
  save, see "Your pick" pill. "Change" button reopens the picker.
- **KO+24h+:** locked summary, no change button.
- **Tally:** appears below the row during `voting` and `closed`.
- **Postponed / abandoned:** voided meta strip, no picker.

Predictor (after the config swap):
- Signed in, no registration in `nl-widgets` yet → registration card.
  Confirm → main view, blank predictions.
- Pick scores on a future match → submit bar → bulk save.
- Sim forward past KO+105m → row locks, scores show, points calculate.

## Files

- `embeds/motm.html` — single-file embed, paste into CMS custom HTML
  block. CMS strips external `<script src>` so Firebase loads
  dynamically via `loadScript`.
- `embeds/motm.rules.json` — MOTM-only rules subset (reference; not
  what's deployed).
- `embeds/nl-widgets.rules.json` — **deployed rules**, covers
  predictor + MOTM.
- `embeds/motm-handover.md` — this file.

## Gotchas worth remembering

- `renderSponsor` / `renderFooter` read `state.registration` and call
  `userCompId()` — must be invoked from `boot()`, not at script-load
  time, because `state` is `undefined` while `var` declarations are
  still being hoisted but unset.
- `position: fixed` inside the embed fights the host page's sticky
  nav. Don't.
- The matchday navigator's click-drag has a 12px deadzone before drag
  engages, and only suppresses the synthetic click if `scrollLeft`
  actually moved. Don't simplify this — the alternative breaks pill
  clicks.
- `users/` immutability means **a fan who registers the "wrong" team
  cannot change it inside the widget.** They go to NL+ and update
  their favourite there, then sign out / in. If you ever need to
  unstick a specific fan, delete their `users/{jwtId}` node in the
  RTDB console manually (rule allows create-only, not update — so they
  re-register fresh).
- Sponsor centre title in MOTM is "Man of the Match" (mixed case, not
  ALL CAPS). The header CSS uppercases it via `text-transform`.
- The "Leading" pill only renders when `topVotes > secondVotes`. If
  you change tally logic, preserve that — it's the whole point of the
  count-less design.
