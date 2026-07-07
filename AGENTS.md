# Eras London Timer — Functional Spec

This is a single-page web app (works as a static file, e.g. on GitHub Pages) that helps a
player of the mobile/RTS game **"Eras of London"** track two things during a match:

1. When **Capital City**'s cooldown will finish (so the player can act on it in-game).
2. When each of 3 **Trainers**' buff pulses will next go active — and, since those buffs
   don't stack, when a newly-enabled Trainer will *actually* be able to fire without
   overlapping another one that's already buffing.

The app does not connect to the game in any way. Everything is driven manually by the
player tapping buttons as they take actions in-game, and the app's job is to do the
timekeeping and shout reminders so the player doesn't have to watch a clock.

## The in-game mechanic being modeled

- **Capital City** has a cooldown (in-game description: 5 minutes). While on cooldown it
  cannot be used. It has **no "active/buff" phase of its own** — it's just cooldown, then
  ready, repeatedly forever (in practice the player re-triggers it each time it's ready).
- **Trainers** (3 of them, capturable independently) are abilities the player enables once
  they're able to (roughly around the 2:30 mark in a match, but this varies). Once
  enabled, a Trainer **loops forever automatically**: it goes "active" (its buff is live)
  for a fixed duration, then "cooldown" for the remainder of its cycle, then active again,
  forever — the player never has to re-press anything for it once it's enabled.
- **While a Trainer is active**, it makes Capital City's cooldown tick faster (a speed
  bonus, confirmed at **15%** — see "Known-good, empirically confirmed values" below for
  how the description-vs-build-screen discrepancy was resolved).
- **Trainer buffs do not stack.** Only one Trainer's buff can be providing its speed bonus
  to Capital City at any given moment, confirmed by in-game testing (forcing two Trainers'
  buffs to overlap did not produce a bigger speed-up than a single Trainer). If a second
  Trainer is enabled while another Trainer's buff is currently live, the second Trainer's
  *first* buff pulse is delayed until the first one's buff ends — it does not start
  immediately just because the player tapped the button. After that first pulse, each
  Trainer keeps looping on its own fixed cycle independently.
  - With the default numbers (35s active / 180s cooldown), 3 Trainers can be staggered by
    the player so that collectively there is *always* exactly one Trainer buffing Capital
    City (35 × 3 ≈ close to the 180s cycle) — this "perfect rotation" is a deliberate
    in-game strategy the app needs to support cleanly.
- **Temporal Manipulator** is a separate, independent speed boost the player can toggle
  on/off for Capital City. It is not tied to any Trainer and does not have its own
  cooldown in this app — it's just an on/off switch. Its bonus stacks with whatever
  Trainer bonus (if any) is currently active.
- **Bonus stacking model: multiplicative**, not additive. Total speed multiplier =
  (1 + Temporal%) × (1 + Trainer%), applied only for whichever single Trainer (if any) is
  currently mid-buff, since Trainer buffs don't stack with each other either. See
  "Known-good, empirically confirmed values" below for the test data behind this.
- **Game clock ratio:** the game's own on-screen match clock runs faster than real-world
  time — confirmed by the player timing 30 in-game seconds against a real stopwatch twice
  (21.06s and 21.10s), giving a ratio very close to **1.4×**, matching StarCraft II's
  "Faster" game speed setting. All duration/cooldown values the player enters into the app
  (e.g. "300" for Capital, "35"/"180" for a Trainer) are **in-game seconds, exactly as the
  game describes them** — the app is expected to run its own internal clock at that same
  accelerated 1.4x rate (not real-world wall-clock speed), so that a value like "35s" for
  a Trainer buff corresponds directly to what the player would see on the game's own
  clock. This ratio is fixed/hardcoded and intentionally not user-editable (to avoid
  confusing the player with a second thing to tune).

## What each screen element does, functionally

### Capital City panel (the large "hero" element, top of the page)
- Shows a countdown ring/timer to when Capital City's cooldown will finish.
- **Enable** button: starts the cooldown countdown *right now* (used once, when the match
  starts — Capital City is already on cooldown in-game from the start of the match, so the
  player taps Enable the moment they open the app / minimize the game).
- **Stop loop**: appears in place of Enable while running; stops the countdown (e.g. if
  the player made a mistake).
- **Temporal Manipulator toggle**: on/off switch, independent of Trainers, adds its own
  speed bonus while on.
- Shows a live readout of the *current total speed bonus* being applied (sum of Temporal
  Manipulator, if on, plus whichever Trainer is currently mid-buff, if any).
- Editable settings (gear icon): base cooldown, Temporal Manipulator's bonus %, and the
  spoken alert phrase.
- **Cycle log** (small panel inside this same card, to the right): every time Capital
  City's cooldown completes a full cycle (goes from Enable/previous-ready back to ready
  again), the app records how long that cycle actually took — in the same in-game-second
  units the player reads off the match clock — so the player can compare the app's
  prediction against what they observe in the actual game. Newest entry on top, numbered
  by cycle count. Cleared by "New game."

### Trainer panels (3 identical cards below Capital City)
- Each represents one Trainer, independent of the other two.
- **Enable**: the moment the player is ready to use that Trainer's ability in-game, they
  tap this.
  - If no other Trainer's buff is currently active, this Trainer's buff starts
    immediately (right now).
  - If another Trainer's buff *is* currently active (or will become active soon enough to
    overlap), this Trainer's first buff pulse is delayed until it can start without
    overlapping any other Trainer's buff window — checked not just for "right now" but
    for that other Trainer's upcoming pulses too, since it auto-loops.
  - After that first pulse, the Trainer keeps auto-looping its own active/cooldown cycle
    forever, independently.
- **Stop loop**: same meaning as on Capital City, per-Trainer.
- Editable settings: active-buff duration, total cycle length ("cooldown from use"),
  the Capital-City speed bonus % this Trainer grants while its buff is active, and the
  spoken alert phrase.
- Visual status: READY (not yet enabled) / ACTIVE (buff currently boosting Capital City) /
  COOLDOWN (waiting for the next pulse) / ALMOST READY (final few seconds before the next
  pulse, per the global "warn before" setting).

### Global controls (top of page)
- **Sound: ON/OFF** toggle — on by default. When on, alerts are audible (a loud
  attention-grabbing beep followed by the timer's spoken alert phrase, e.g. "Trainer 1").
  Sound needs one click/tap anywhere on the page to actually unlock playback the very
  first time, due to browser autoplay rules — this is expected, not a bug.
- **Volume slider** — can go above 100% (the audio is compressed/limited so this makes it
  louder without distorting, up to the ceiling of what a browser will play; the spoken
  voice alert itself is capped at 100% by the browser regardless of this slider).
- **Warn before (s)** — a single global setting (not per-timer) for how many seconds
  before something is about to happen that the alert should fire. Applies to Capital City
  and to all 3 Trainers alike. This is the *only* thing that triggers a sound — there is
  intentionally no separate "just went active" sound; exactly one alert per cycle, timed
  by this setting.
- **New game** — resets every timer (Capital City + all 3 Trainers) back to not-running/
  READY, turns off Temporal Manipulator, and clears the Capital City cycle log. Does not
  change any of the configured durations/percentages/names — only running state and
  history.

## Known-good, empirically confirmed values (as of latest testing)
- Temporal Manipulator bonus: **50%** (in-game test: Temporal alone gave exactly 3:20 for
  a 5:00/300s base cooldown → 300/1.5 = 200s = 3:20, exact match).
- Trainer buffs genuinely do not stack (confirmed by a deliberate overlap test).
- Game clock ratio: **1.4×**, matching SC2 "Faster" speed (confirmed via real stopwatch
  vs. in-game clock, two trials averaging ~1.423, consistent with 1.4 given manual
  stopwatch reaction-time error).
- Trainer active-buff duration: **35 in-game seconds** (stated directly by the player from
  testing; also implied by their note that ~5 Trainers, or 4 with a small gap, would be
  needed to fully tile the ~180s cycle).
- **Trainer's Capital-City speed bonus: 15%** (matches the in-game flavor description, not
  the 11.5% shown on the build/stats screen). Confirmed by an in-game test of "perfect
  rotation" (Trainers only, no Temporal Manipulator) taking 4:23/263s: 300/263 = 1.1407 →
  ≈14.07% bonus, close enough to 15% (small gap attributable to imperfect human timing
  when staggering the rotation) that 15% is taken as the correct value.
- **Stacking model between Temporal Manipulator and a Trainer's bonus: multiplicative, not
  additive.** I.e. total speed multiplier = (1 + Temporal%) × (1 + Trainer%), not
  1 + Temporal% + Trainer%. Confirmed by combining the two tests above: "perfect rotation"
  with Temporal Manipulator ON measured 2:56/176s. The multiplicative model with 50%
  Temporal and 15% Trainer predicts 300 / (1.5 × 1.15) = 300/1.725 ≈ 173.9s ≈ 2:54, very
  close to the measured 2:56. The additive model (1 + 0.50 + 0.15 = 1.65) predicts
  300/1.65 ≈ 181.8s ≈ 3:02, which is clearly further off. Multiplicative is the accepted
  model going forward.

> **Implementation note:** the app's code now uses the multiplicative model and a 15%
> Trainer bonus default, matching the conclusion above.

## Open questions / not yet resolved — do not "fix" these without new test data
- Whether "Warn before" should be interpreted in real-world seconds or in-game seconds
  (currently: in-game seconds, i.e. subject to the 1.4x ratio like everything else) has
  not been explicitly questioned by the player, but is worth keeping in mind if the alert
  ever seems to fire "too early/late" relative to real reaction time.
- The 15%/multiplicative conclusion above rests on two in-game tests; the player intends
  to keep testing in-game to further confirm before it's fully locked in.

## What "correct" looks like, functionally
Given the above, a cycle of Capital City with **zero** active bonuses (Temporal off, no
Trainer buff active for the whole cycle) should always take exactly the configured base
cooldown (300 in-game seconds by default) — no more, no less, ever. Any bonus present
should only ever make a cycle *faster* than that baseline, never slower. If the Cycle log
ever shows a cycle longer than the configured base cooldown, that is a bug.

> **Note:** there used to be a Resync (⟲) button on Capital City and each Trainer, meant to
> force a running timer to restart counting from *right now* if the app's timing drifted
> from the real game state. It was removed: it doesn't respect the "Trainer buffs don't
> stack" rule (it could force a Trainer active while another Trainer's buff was already
> live), and in practice the player just clicks Enable/Stop loop accurately enough that
> a manual resync isn't needed. Don't re-add it without also re-solving that overlap
> problem.
