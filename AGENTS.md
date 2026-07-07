# Eras London Timer — Functional Spec

This is a single-page web app (works as a static file, e.g. on GitHub Pages) that helps a
player of the mobile/RTS game **"Eras Zombie Invasion"** track two things during a match:

1. When **Capital City**'s cooldown will finish (so the player can act on it in-game).
2. When each of 5 **Trainers**' buff pulses will next go active — and, since those buffs
   don't stack, when a newly-enabled Trainer will *actually* be able to fire without
   overlapping another one that's already buffing.

The app does not connect to the game in any way. Everything is driven manually by the
player tapping buttons as they take actions in-game, and the app's job is to do the
timekeeping and shout reminders so the player doesn't have to watch a clock.

## The in-game mechanic being modeled

- **Capital City** has a cooldown (in-game description: 5 minutes). While on cooldown it
  cannot be used. It has **no "active/buff" phase of its own** — it's just cooldown, then
  ready, repeatedly forever (in practice the player re-triggers it each time it's ready).
- **Trainers** (5 of them, capturable independently) are abilities the player enables once
  they're able to (roughly around the 2:30 mark in a match, but this varies). Once
  enabled, a Trainer **loops forever automatically**: it goes "active" (its buff is live)
  for a duration, then "cooldown" for the remainder of its cycle, then active again,
  forever — the player never has to re-press anything for it once it's enabled.
- **A Trainer's active-buff duration depends on Temporal Manipulator**, tested in both
  Capital City ("London") and Obelisk: **55 in-game seconds** if Temporal Manipulator is
  off, but only **35 in-game seconds** if Temporal Manipulator is on for that same
  building — apparently a genuine in-game bug/quirk, not a design choice, but the app
  models it as-is since the goal is matching observed behavior. The duration for a given
  pulse is locked in from Temporal's on/off state at the exact moment that pulse starts;
  toggling Temporal mid-buff does not retroactively change a pulse already in progress,
  only the next one. This is hardcoded (`TRAINER_DURATION_NO_TM` / `TRAINER_DURATION_WITH_TM`
  in the code), not a per-Trainer editable setting, since it's a fixed game fact.
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
  - With Temporal Manipulator **on** (35s active / 180s cooldown), 5 Trainers can be
    staggered by the player so that collectively there is *always* exactly one Trainer
    buffing Capital City (35 × 5 = 175s, close to the 180s cycle) — this "perfect
    rotation" is a deliberate in-game strategy the app needs to support cleanly. 5 is also
    the in-game cap on how many Trainers can help reduce Capital City's cooldown at all.
    With Temporal Manipulator **off** (55s active), 5 × 55 = 275s no longer tiles a 180s
    cycle without overlap — "perfect rotation" as originally described assumes Temporal is
    on. This hasn't been re-tested with Temporal off; treat gapless off-Temporal rotation
    math as unconfirmed until it is (see Open questions).
- **Temporal Manipulator** is a separate, independent speed boost the player can toggle
  on/off for Capital City. It is not tied to any Trainer and does not have its own
  cooldown in this app — it's just an on/off switch. Its bonus stacks with whatever
  Trainer bonus (if any) is currently active.
- **Bonus stacking model: multiplicative**, not additive. Total speed multiplier =
  (1 + Temporal%) × (1 + Trainer%), applied only for whichever single Trainer (if any) is
  currently mid-buff, since Trainer buffs don't stack with each other either. See
  "Known-good, empirically confirmed values" below for the test data behind this.
- **Game clock ratio:** the game's own on-screen match clock runs faster than real-world
  time — confirmed by three independent game-clock-vs-real-stopwatch trials of increasing
  duration and precision: 30 in-game seconds → ~21.08s real (ratio ≈1.423), 120 in-game
  seconds → 84s real (ratio ≈1.4286), and 450 in-game seconds (17:40→25:10 on the match
  clock) → 315.28s real (ratio ≈1.4271). These three converge tightly on **≈1.427×** —
  notably *not* StarCraft II's "Faster" 1.4× exactly, even though that was the original
  working assumption from the first, shortest (and noisiest) trial. Eras of London only
  resembles SC2 in flavor/naming; there's no reason its engine must reproduce SC2's tick
  rate precisely. All duration/cooldown values the player enters into the app (e.g. "300"
  for Capital, "35"/"180" for a Trainer) are **in-game seconds, exactly as the game
  describes them** — the app is expected to run its own internal clock at that same
  accelerated rate (not real-world wall-clock speed), so that a value like "35s" for a
  Trainer buff corresponds directly to what the player would see on the game's own clock.
  This ratio is fixed/hardcoded and intentionally not user-editable (to avoid confusing
  the player with a second thing to tune).

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

### Trainer panels (5 identical cards below Capital City)
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
- Editable settings: total cycle length ("cooldown from use"), the Capital-City speed
  bonus % this Trainer grants while its buff is active, and the spoken alert phrase.
  Active-buff duration is *not* editable — it's derived automatically each pulse from
  Temporal Manipulator's on/off state (55s / 35s, see above).
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
  and to all 5 Trainers alike. This is the *only* thing that triggers a sound — there is
  intentionally no separate "just went active" sound; exactly one alert per cycle, timed
  by this setting.
- **New game** — resets every timer (Capital City + all 5 Trainers) back to not-running/
  READY, turns off Temporal Manipulator, and clears the Capital City cycle log. Does not
  change any of the configured durations/percentages/names — only running state and
  history.

## Known-good, empirically confirmed values (as of latest testing)
- Temporal Manipulator bonus: **50%** (in-game test: Temporal alone gave exactly 3:20 for
  a 5:00/300s base cooldown → 300/1.5 = 200s = 3:20, exact match).
- Trainer buffs genuinely do not stack (confirmed by a deliberate overlap test).
- Game clock ratio: **≈1.427×** (superseding the earlier 1.4/SC2-"Faster" assumption).
  Confirmed via three real-stopwatch-vs-in-game-clock trials of increasing duration/
  precision: 30s→1.423, 120s→1.4286, 450s→1.4271 — a tight, convergent cluster. A
  separate indirect test (timing a full 300s/5:00 Capital City cooldown end-to-end with
  no bonuses active) still showed a small ~4s residual gap even after applying 1.427,
  down from ~8s under the old 1.4 — attributed to ordinary human reaction-time margin
  when starting/observing the test, not a further ratio correction (see Open questions).
- Trainer active-buff duration: **55 in-game seconds normally, 35 if Temporal Manipulator
  is on** for the same building (tested in Capital City and Obelisk) — apparently an
  in-game bug, modeled as-is regardless. The original "35s" figure (still stated directly
  by the player from testing) turned out to only hold for the Temporal-on case; at 35s
  each, 5 Trainers (175s) almost exactly tile the ~180s cycle, which is what makes
  "perfect rotation" work and is why 5 is both the practical and in-game-enforced max.
- **Trainer's Capital-City speed bonus: 15%** (matches the in-game flavor description, not
  the 11.5% shown on the build/stats screen). Confirmed by an in-game test of "perfect
  rotation" (Trainers only, no Temporal Manipulator) taking 4:23/263s: 300/263 = 1.1407 →
  ≈14.07% bonus, close enough to 15% (small gap attributable to imperfect human timing
  when staggering the rotation) that 15% is taken as the correct value. **Caveat added
  after discovering the Temporal-dependent duration above:** this test is described as
  "no Temporal Manipulator", which per the duration finding means each Trainer's buff
  should have been 55s, not 35s — under which 5 Trainers (275s) can't gaplessly tile a
  180s cycle the way "perfect rotation" describes. Whether this test's setup, the 15%
  figure, or the "no gaps" framing needs revisiting is unresolved; flagged in Open
  questions rather than guessed at here.
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
  (currently: in-game seconds, i.e. subject to the game clock ratio like everything else)
  has not been explicitly questioned by the player, but is worth keeping in mind if the
  alert ever seems to fire "too early/late" relative to real reaction time.
- The 15%/multiplicative conclusion above rests on two in-game tests; the player intends
  to keep testing in-game to further confirm before it's fully locked in.
- The ~4s residual gap left in the Capital City cooldown test even after correcting the
  ratio to 1.427 (see above) isn't fully explained. Current best guess is ordinary
  human timing margin, not a further code/ratio issue — a tick-accumulation-drift bug was
  ruled out mathematically (simulated: max ~20ms drift even under heavy jitter), and no
  other code-side cause was found on review. If a future clean test reproduces a similar
  multi-second gap consistently, revisit this rather than assuming it's settled.
- The Temporal-dependent Trainer duration (55s off / 35s on) was discovered *after* the
  15%-Trainer-bonus "perfect rotation, no Temporal" test was already taken as confirmed.
  That test's framing (gapless rotation, no Temporal) doesn't obviously square with 55s
  Trainers not tiling a 180s cycle gaplessly at 5 Trainers. Needs a fresh in-game test
  specifically re-checking rotation behavior with Temporal off before trusting either the
  15% figure or the no-Temporal "perfect rotation" framing further.

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
