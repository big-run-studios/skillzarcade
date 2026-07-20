# Titan Guard — Vertical-Slice Design Doc

> Display name **Titan Guard**. The folder / URL stays `titan-toll/` (like Unicorn Assault kept
> `unicorn-horde/`), so links and git history are undisturbed.

A skill-based **crash game** presented as a dramatic 1v1 JRPG boss battle. One low-poly
warrior with a **transforming pike** (melee / bazooka / staff) fights up to **three
increasingly deadly titans** in one ~2–3 minute run, accumulating score that stays **at
risk until cashout**. Fall before cashing = **zero**.

Everything lives in one self-contained file: [`titan-toll/index.html`](index.html) (three.js r160,
vendored in `lib/`). Deterministic from a run seed. Headless harness at `window.__game`.
All telegraph/attack/sequence timings run through a global `CFG.timeScale` (currently **2×**,
i.e. deliberately slow/readable).

---

## 1. Core fantasy & framing

- **Quarter-profile camera** (FOV 41, orbited ~22.5° off the lane axis, zoomed out): warrior
  **bottom-left**, mirrored stance, feet just above the Strike button; titan looming **upper-right**;
  the grassy field/lane runs diagonally between them so you read the **depth and distance**.
- Distant hills/mesas, bright sky, big billboard clouds — late-90s low-poly, original assets only.
- The pike **visibly transforms** through the character's animation: held for **Strike**, flipped
  muzzle-over-shoulder for **Shoot**, planted as a glowing staff for **Cast**. The energy tip
  recolours per form (amber / cyan / violet / gold-blitz).

---

## 2. Combat-state diagram

```
                         ┌─────────┐
                         │  MENU   │  (seed entry, ENTER BATTLE)
                         └────┬────┘
                              │ start(seed) → seed 3 RNG streams
                              ▼
        ┌──────────────────────────────────────────────┐
        │                  COMBAT  (per enemy)           │
        │                                                │
        │  PLAYER track            ENEMY track (director)│
        │  ┌───────────┐           ┌───────────────────┐ │
        │  │ ACTION BAR│  fills    │ IDLE (breather)   │ │
        │  │ fills 2.7s│──full──┐  │  ↓ chooseAttack() │ │
        │  └───────────┘        │  │ TELEGRAPH (windup)│ │
        │        │ commit()     │  │  ↓                │ │
        │        ▼              │  │ ACTIVE (hits land)│ │
        │  ┌───────────┐        │  │  ↓                │ │
        │  │  OFFENSE  │◄───────┘  │ RECOVERY (vuln)   │ │
        │  │ tap rings │           │  ↓                │ │
        │  │Miss/Good/ │  gets hit │ (back to IDLE)    │ │
        │  │ Perfect   │──────────►│ STAGGER (on Blitz)│ │
        │  └─────┬─────┘ interrupt └───────────────────┘ │
        │        │                                        │
        │   TURN-BASED: only ONE timing game is ever live.│
        │   Attacking→tap=hit; enemy attacking→tap=dodge  │
        │   (−0.32/+0.2s); commit during attack = QUEUED. │
        └───┬───────────────┬───────────────┬────────────┘
            │ enemy.hp≤0     │ cashout()     │ player hp≤0
            │ (foe 1&2)      │ (foe 1&2)     │
            ▼                ▼               ▼
    ┌──────────────┐  ┌────────────┐  ┌────────────┐
    │  DEFEATED    │  │  RESULT    │  │  RESULT     │
    │ 1.3s → next  │  │ CASHED OUT │  │  FALLEN     │
    │ enemy (carry │  │ score kept │  │  score = 0  │
    │ hp/mult/lv/  │  └────────────┘  └────────────┘
    │ score/bars)  │
    └──────┬───────┘
           │ enemy.hp≤0 on FOE 3
           ▼
    ┌──────────────────────────┐
    │ RESULT: TITANS FELLED     │
    │ auto-cashout + full-clear │
    │ bonus (+30%)              │
    └──────────────────────────┘
```

**One timing game at a time (turn-based):** there are never two timing games on screen.
While **you** attack, the enemy is paused (it can't start an attack). While the **enemy**
attacks, you dodge — and if you commit an action (the bar was full) it is **queued** and fires
the instant their attack ends (its button shows `QUEUED`). This removes the old
"abandon-your-combo-to-dodge" scramble in favour of a clean read: attack cleanly on your beat,
then dodge cleanly on theirs.

---

## 3. Player action system (v0.46: cooldown weave — no action bar)

| Action | When | Cooldown | Commit time | Damage | Niche |
|---|---|---|---|---|---|
| **Shoot** | own cooldown up | **4 s** | shortest | weakest (base 62) | the filler — always coming back |
| **Cast** | own cooldown up | **8 s** | medium | middle (base 112) | the mid hit; **Cliffhorn** vuln |
| **Strike** | own cooldown up | **13 s** | long | the payoff (base 200) | the big beat; ground ring shows it charging |
| **Blitz** | Blitz Bar full only | none (meter) | longest | **tops every chart** (base 160 ×1.35) | earned by dodging; staggers a *starting* attack |

**No action bar.** Every attack runs its own **flat cooldown** that starts when it fires and
**ticks at all times** — through your other attacks and through the enemy's turns — so skilled play
is a weave: strike on its beat, casts and shots stitched between. The enemy's turn clock also runs
through your offense (its attack still never *starts* mid-offense), so hyper-aggression can't
starve the dodge game. **Unified tap** unchanged: one timing game live at a time.

**Mobile HUD (diegetic-first).** The **ground ring** around the hero's feet now charges with
**Strike** (blue filling → strike-red pulse when the payoff hit is ready). Action selection is the
lower-right **radial wheel** — each button is a piece of **per-level icon art (L1–L9)**, so every
level-up visibly upgrades all three buttons at once (that art escalation IS the level indicator);
cooling buttons grey under a conic sweep + seconds countdown. **Blitz still has no bar** — the pike
tints gold as it charges. Bottom-left is the **hero plate** (`assets/hero-plate.webp`): portrait +
**Lv.** + the **HP bar** (with a delayed amber damage chip), replacing the old full-width bottom
bar; **Cash Out** sits just above it. Multiplier + score stay top-left; XP stays hidden.

### One global level & 3/6/9 evolutions
The player has **one level, 1–9** — every attack ramps together (+20% damage per level via
`lvDmg`, wider Good window `lvGood`, higher crit `lvCrit`). Milestones at **3 / 6 / 9** swap in
**new, longer timing sequences for all attacks at once** (see `SEQ`).

| Level | Strike | Shoot | Cast | Blitz |
|---|---|---|---|---|
| **1** | single thrust (1 ring) | 1 shot | 1 blast/charge | 5-input combo |
| **3** | 3-hit combo | 3 precision shots | 3-rune ordered seq | 7-input (extra melee+cast) |
| **6** | launcher + aerial finisher | barrage + charged shot | charge + detonation | 8-input, branching weak-points + launcher |
| **9** | extended combo + delayed heavy | multi-target + explosive weak-point | full 5-glyph + big detonation | ultimate 9-input, hard final window |

### XP (v0.46: skill events, not damage)
XP is granted **per graded event**: Perfect tap **+12**, Good tap **+4**, Perfect dodge **+20**,
Good dodge **+8**; misses and taken hits give **0**. Level-ups are fully deterministic (no seed
pick — the one global level just increments). **Max 8 level-ups** (`xpThresh` =
`[35,40,45,50,54,58,62,66]`), fitted to the perfect-run event budget so clean play reaches **L9
for the Sky Titan finale**. Bot-verified: perfect clear 51 s reaching L9 at ~80%; careful-but-
imprecise (±200 ms) clears ~103 s at L9; sloppy dodge discipline dies at the boss-3 wall.

---

## 4. Crash / risk systems

- **Scoring:** each resolved hit deals `dmg` to the enemy **and** adds `dmg × mult` to your
  score. Damage→score is 1:1 *before* the multiplier. Taking damage **never** removes score.
- **Multiplier (1×–5×):** meter fills from Good (+0.10) / Perfect (+0.20) offense, Good
  (+0.15) / Perfect (+0.32) dodges, and a clean-sequence bonus (+0.28). Each full meter = +1×.
  **Getting hit drops one full level and clears the meter.** Only affects future scoring —
  never the score already banked.
- **Blitz Bar (0–100), v0.46 — dodge-earned:** Good dodge **+8**, Perfect dodge **+15**; damage
  dealt only trickles it (`0.0015/dmg`, +2 on crit); **no charge from taking hits**. Blitz is the
  star move and you work for it: roughly one per boss for a clean dodger.
- **Cashout:** persistent button showing live score; ends the run immediately and banks the
  score. **Locked ~0.30 s (`resolveLockT`) whenever damage is resolving**, so you can't get a
  contradictory "cashed out AND died in the same frame" outcome.
- **Death:** hp ≤ 0 (only via a landed enemy hit) → run ends, **score = 0**.
- **Full clear:** third titan down → auto-cashout + **+30% full-clear bonus**.

---

## 5. Dodge & offensive timing

- **Reticle:** every timing target (dodge and offense) is a **thin ring shrinking onto a thick
  "success band"** — line the thin ring up with the band to hit (band centre = Perfect). The band's
  screen thickness *is* the success window, so fairness is legible.
- **Dodge window is ASYMMETRIC: −0.32 s / +0.20 s** around each hit's impact (anticipatory side is
  wider — dodging *ahead* of a hit is the natural instinct; 520 ms total). Perfect dodge = within
  **±0.09 s**. Outside both sides = the hit lands in full. **Every hit in a multi-hit set needs its
  own dodge.** A successful dodge cancels all damage, feeds mult + Blitz, pops a beam ring.
- **Offense grades:** Miss / Good (±`lvGood`, ~0.115 s +0.004/lv) / Perfect (±0.05 s). A
  **finisher** hit tapped dead-center (±0.022 s) is a **guaranteed critical**.
- **Crit:** deterministic roll from `rngCrit`; Good ~8% + lv, Perfect ~24% + lv; crit = ×1.9 dmg.
- **Vulnerability:** during an enemy's **recovery** it exposes a weak action (Cliffhorn→Cast,
  Gale→Shoot, Titan→shifts each opening, shown via eye/gem colour). Matching it = ×1.6 damage
  and bonus mult.

---

## 6. Enemy attack-set table (30 total)

Per-hit damage below is the **authored value** (% of a 120-HP bar at 1:1). Actual damage taken
= authored × **`enemyDmgScale`** = **E1 ×0.55, E2 ×0.68, E3 ×0.96** (the main difficulty knob).
Hits: `speed@impact_s:dmg`. Speeds: **s**low / **m**ed / **f**ast (telegraph leads 0.90 / 0.58 / 0.34 s).

### Enemy 1 — Cliffhorn Juggernaut  (HP 850 · weak: **Cast** · slow, huge tells)
| # | Set | Hits (sp@t:dmg) | Rec | Track | Diff | Telegraph | Notes |
|---|---|---|---|---|---|---|---|
| 1 | Boulder Overhead | s@0.95:32 | 1.5 | – | 1 | rears both fists overhead, foot plant | teaches delayed slow dodge |
| 2 | Horn Gore | m@0.70:20 | 1.1 | ✓ | 1 | lowers head, paws ground | |
| 3 | Twin Stomp | m@0.65:15, m@1.05:16 | 1.2 | – | 2 | lifts one foot then the other | repeated dodges |
| 4 | Shoulder Charge | s@1.05:34 | 1.6 | ✓ | 2 | rears back, digs a heel | long recovery = Cast window |
| 5 | Rockfall | s@0.90:24, m@1.28:14 | 1.3 | – | 2 | slams ground, debris | mixed speed |
| 6 | Quake Clap | m@0.62:15, m@0.98:17 | 1.0 | – | 2 | hands wide → clap | |
| 7 | Triple Hammer | m@0.60:13, m@0.92:13, m@1.28:14 | 1.3 | – | 3 | three overhead blows | |
| 8 | Gore & Toss | m@0.65:18, s@1.40:30 | 1.5 | ✓ | 3 | gore, then heaves overhead | delayed finisher |
| 9 | Ground Sunder | s@1.15:38 | 1.7 | – | 3 | giant two-hand smash | biggest single hit; huge opening after |
| 10 | Avalanche | m@0.60:14, m@0.96:14, s@1.66:30 | 1.4 | – | 4 | enraged flurry → slow crusher | **HP<40% only** |

### Enemy 2 — Gale Reaper  (HP 1500 · weak: **Shoot** · fast, airborne)
| # | Set | Hits (sp@t:dmg) | Rec | Track | Diff | Telegraph | Notes |
|---|---|---|---|---|---|---|---|
| 1 | Twin Slash | f@0.40:10, f@0.75:10 | 0.8 | ✓ | 2 | crosses twin blades | |
| 2 | Dash Cut | f@0.50:13 | 0.7 | ✓ | 2 | blurs sideways, wing flick | |
| 3 | Scythe Flurry | f@0.40:9, f@0.72:9, f@1.04:9 | 0.9 | – | 3 | rapid three-hit fan | |
| 4 | Feint Slash | f@0.45:11, m@1.05:18 | 1.0 | ✓ | 3 | quick jab, **pause**, heavy | baits early dodge |
| 5 | Rising Talon | m@0.62:15, f@0.97:10 | 0.9 | ✓ | 3 | crouch → leap airborne | goes airborne (Shoot it) |
| 6 | Gale Combo | f@0.40:8, f@0.72:8, m@1.15:14, f@1.50:9 | 1.0 | – | 4 | four mixed-speed cuts | tempo shift |
| 7 | Wind Shear | f@0.45:12, f@0.80:12 | 0.7 | ✓ | 3 | spins, wind ring | |
| 8 | Reaper's Dance | f@0.40:7, f@0.70:7, f@1.00:7, f@1.30:7, m@1.75:15 | 1.1 | – | 4 | four fast → heavy finisher | |
| 9 | Shadow Cross | m@0.62:14, m@1.00:14, f@1.35:10 | 0.9 | ✓ | 3 | two mediums → fast snap | |
| 10 | Tempest | f@0.40:8, f@0.70:8, f@1.00:8, m@1.45:16, s@2.15:26 | 1.2 | – | 5 | storm → slow crusher | **HP<45% only** |

### Enemy 3 — Sky Titan  (HP 2800 · weak: **shifts** each opening · all speeds)
| # | Set | Hits (sp@t:dmg) | Rec | Track | Diff | Telegraph | Notes |
|---|---|---|---|---|---|---|---|
| 1 | Sky Smite | s@1.00:36 | 1.1 | – | 2 | raises fist, light gathers | beam column |
| 2 | Comet Jabs | f@0.40:9, f@0.72:9, f@1.04:9 | 0.7 | – | 3 | three rapid punches | short recovery |
| 3 | Feint Crush | m@0.65:16, s@1.55:32 | 1.2 | ✓ | 4 | jab, long pause, crusher | wide gap baits you |
| 4 | Star Barrage | f@0.40:8, f@0.70:8, f@1.00:8, f@1.30:8 | 0.8 | – | 4 | four-shot star volley | **punishes long Cast** |
| 5 | Gravity Slam | s@0.95:30, m@1.35:18 | 1.0 | – | 4 | slow slam → fast follow | |
| 6 | Tempo Trap | m@0.60:14, f@0.90:10, s@1.70:30 | 1.0 | ✓ | 4 | med, fast, delayed slow | tempo whiplash |
| 7 | Meteor Rain | f@0.40:8, m@0.80:14, f@1.15:8, m@1.55:14, f@1.90:9 | 0.9 | – | 5 | five alternating-speed meteors | |
| 8 | Titan Fury | m@0.62:13, m@0.97:13, m@1.32:13, m@1.67:13, s@2.35:28 | 1.0 | – | 5 | four mediums → slow finisher | metronome then delay |
| 9 | Judgement | f@0.35:11, f@0.65:11, f@0.95:11 | 0.6 | ✓ | 4 | snap volley at casters | **punishes long Cast** |
| 10 | Cataclysm | s@0.90:26, f@1.30:9, f@1.60:9, m@2.00:16, s@2.85:34 | 1.1 | – | 5 | everything, five hits | **HP<40% only** |

**Director rules** (`chooseAttack`, seeded via `rngDir`): never the same set >2 in a row;
`castPunish` sets (Star Barrage, Judgement) weight ×3.4 while you're casting (and ×2 on a cast
streak); a 3rd consecutive *fast* set is suppressed (×0.35); difficulty is paced toward the
enemy's remaining-HP fraction; low-HP enrage sets gated by `maxHpFrac`. With the asymmetric window
(0.32+0.20 = 0.52 s) adjacent windows CAN overlap on tight gaps — `tapDodge` resolves the tap to the
**nearest unresolved hit**, which disambiguates chains correctly (a resolved hit never eats a tap).

---

## 7. Balance table (all data-driven values, `CFG` + data)

| Group | Value | Setting |
|---|---|---|
| **Player** | HP | 1000 |
| **Enemy HP** | E1 / E2 / E3 | 4250 / 7500 / 14000 |
| **Enemy dmg scale** | E1 / E2 / E3 | 5.5 / 6.8 / 9.6 |
| **Enemy breather** | E1 / E2 / E3 (s) | 3.4 / 2.6 / 2.0 (turn clock runs through player offense) |
| **Cooldowns** | shoot / cast / strike | 4 / 8 / 13 s (flat, always ticking; blitz = meter) |
| **Dodge** | early / late / perfect | −0.32 / +0.20 / ±0.09 s (× speed scale) |
| **Offense** | good / perfect / center | ±0.115 (+0.004/lv) / ±0.05 / ±0.022 s |
| **Grade dmg** | miss / good / perfect | ×0.18 / ×1.0 / ×1.55 |
| **Crit** | mult / good% / perfect% / per-lv | ×1.9 / 0.08 / 0.24 / +0.012 |
| **Vulnerability** | dmg / mult bonus | ×1.6 / +0.14 meter |
| **Multiplier** | max / good / perfect | 5 / +0.10 / +0.20 |
| | dodge / perfect-dodge / clean | +0.15 / +0.32 / +0.28 |
| **Blitz** | max / per-dmg / crit | 100 / 0.0015 / +2 |
| | dodge / perfect dodge / on-hit | +8 / +15 / 0 (dodge-earned) |
| **Action base dmg** | shoot/cast/strike/blitz | 62 / 112 / 200 / 160×1.35 (× weights × lvDmg × grade × crit) |
| **Level ramp** | lvDmg | +20%/level (×2.6 at L9), one global level |
| **XP** | perfect tap / good tap | +12 / +4 |
| | perfect dodge / good dodge | +20 / +8 |
| | thresholds (8) / cap | 35,40,45,50,54,58,62,66 / L9 |
| **Cashout** | resolve lock / full-clear bonus | 0.30 s / +30% |
| **Feel** | hit-pause / seq tail / blitz stagger | 0.07 / 0.35 / 1.4 s |
| **Target curve** | perfect / careful / sloppy | ~51 s clear · ~103 s clear · dies at boss 3 (bot-verified) |

---

## 8. Placeholder assets still needed for production

Everything renders from three.js primitives + procedural animation — intentionally
placeholder-grade. For final production:

**Animation**
- Player: proper rigged skeleton + mocap-style clips for idle, the three pike transforms
  (grip→shoulder-cannon→planted-staff transitions are currently pose-lerps), strike lunge,
  shoot recoil, cast channel, dodge sidestep/hop, hurt recoil, victory.
- Blitz: a real choreographed multi-form combo animation (currently a pose loop).
- Enemies ×3: rigged windup/strike/recovery/stagger/enrage clips per attack set so the
  **telegraph reads from the body first** (currently arm-raise + eye-flash approximations).
  Airborne (Gale) and shifting-vuln (Titan) states need bespoke poses.

**VFX**
- Pike energy trails per form; muzzle flash + projectile tracers (Shoot); rune/glyph seals and
  detonation (Cast); per-hit impact bursts; crit flourish; the Blitz ultimate sequence.
- Enemy telegraph particle build-ups, ground shadow/danger decals, per-attack signature FX
  (rockfall debris, wind ring, meteor streaks, gravity crater).

**Audio** — battle music + win/fail stingers are now real authored tracks
(`battle-music.mp3` loops during combat, `success.mp3` on cashout/full-clear, `fail.mp3` on
death), routed through `<audio>` tags for iOS Silent-Mode. Still placeholder: the in-combat
**SFX** (synth tones) — replace with authored cues for per-**speed** attack startups,
Good/Perfect/Crit hits, dodge whoosh, per-form pike sounds, Blitz, and level-up.

**UI/Art** — real timing-ring/reticle/glyph art, a rendered **hub tile** (`tile.webp`, currently
a CSS placeholder), enemy portrait/nameplate art, cashout/result screen polish.

**Environment** — hand-modelled cliffs/mesas, foliage, sky, and per-enemy arena dressing.

---

## 9. Does it create the "attack again vs cash out" tension?

**Yes.** Validated with a deterministic headless auto-player (`window.__game`) across skill
tiers and many seeds (tactical bot = attacks only in safe windows, one dodge decision per hit):

| Skill (dodge success) | Result | Reaches | Avg HP left | Read |
|---|---|---|---|---|
| Perfect (100%) | **full clear** 8/8 | Foe 3 | 120 | mastery is rewarded |
| Good (93%) | full clear ~7/8 | Foe 3 | ~50 | excellent play wins, but **barely** |
| Competent (83%) | 0/8 clear | dies **partway through Foe 2** | 0 | the crux decision point |
| Sloppy (70%) | dies on Foe 1–2 | Foe 1–2 | 0 | |
| Weak (52%) | dies on Foe 1 | Foe 1 | 0 | luck can't save a poor player |

The tension is structural, not scripted:

1. **Score is only ever at risk.** It never drops when you're hit — so a big banked total is
   pure temptation. The multiplier (built by dodging + perfect timing) makes *late* hits worth
   far more, so the longer you survive the more each additional action is worth **and** the more
   you stand to lose.
2. **Getting hit is severe but not score-destroying** — it interrupts your combo, drops your
   multiplier a full level, and chips a no-heal HP bar. So a mistake doesn't take your money; it
   raises the odds that the *next* mistake will (death = everything).
3. **The competent-player wall is Foe 2→3, exactly where the pot is already juicy.** A player who
   cleared Cliffhorn and is mid-Gale with a healthy score faces the real question every crash
   game wants: *bank ~6–15k now, or push into the Sky Titan for the full-clear +30%?* The Sky
   Titan's short recoveries, feints, and Cast-punish volleys mean pushing without excellent
   execution usually ends in zero.
4. **The cashout lock** (0.30 s during damage resolution) removes the cheese of banking on the
   exact frame a lethal hit lands, keeping the risk honest.

Determinism means a given seed has a fixed enemy order, upgrade order and crit rolls, so players
can compare runs on the same seed — a skilled player reliably outperforms an average one on the
same seed, while some seeds naturally produce swingier high-crit runs. Same seed + same inputs =
identical outcome (verified: score, attack order, and level-up order match exactly across repeat
runs).

---

## 10. Debug panel & headless harness

**Debug panel** (⚙ button, or `` ` `` key): pick/restart seed, pause, slow-mo slider, force a
specific enemy attack, enemy −50% HP, set all ability levels or +1 random, fill Action/Blitz,
set multiplier, heal, god-mode, and a live **timing log** (per-input error in ms).

**`window.__game`** (headless): `s` snapshot, `step(dt)` (deterministic sim advance),
`start(seed)`, `commit(a)`, `tapTargetAt(err)` / `dodgeNearestAt(err)` (explicit-grade inputs),
`curTargetErr()` / `nextHit()`, `forceAttack(id)`, `setLevel/setMult/fillAction/fillBlitz`,
`cashout()`, `hitLog()`, `slow(f)`, plus framing helpers (`setCam/setFov/setPlayerHome/…`).
Same seed + same input sequence ⇒ identical enemy order, upgrade order, and crit rolls.

## 11. Battle camera (v0.31.0, expanded v0.45.0)

State-driven **camera director** (`CAMS` literal + `directCamera()` in `frame()`), cosmetic only —
reads sim state, never writes it. Four laws:

1. **Both fully framed** — hero + entire titan on screen during any live-input beat.
   Exempt (pure cinematics): intro, enemy death, end game.
2. **Never off the art** — the painted backdrop lives at −z; every shot aims into the valley.
   No reverse angles.
3. **Cut or crawl** — fast changes are hard cuts; glides are slow + eased; FOV static per shot;
   horizon always level (no roll, no FOV animation — dolly punches instead).
4. **Additive layers only** — dolly-punch spring (hits), breathing sway, legacy shake; all small,
   all decaying.

Shots: **intro** titan reveal (v0.45: reveal shortened, hero idles through it, the stance starts
WITH the pull-back at 1.25x and ends exactly on the 6.1s gate) · **home** + tighten drifts (telegraph / vuln window / rage phase / at-risk
bank, slew-capped, total ≤ 0.12) · **blitz** hard-cut low-angle hero shot, static through the
combo (per-foe pull-back in `CAMS.blitz.perFoe` keeps Gale's wings + the Sky Titan framed) ·
**kill** cut-in push on the death dissolve · **victory / crash / cashout** end shots ·
**attack** (v0.45) glide low/screen-left of PHOME on commit, static through the timing game,
glide back out on resolve (per-foe fixups for Gale's span + Sky's halo) · **imposing** (v0.45,
melee wind-ups = Juggernaut only) dolly toward the titan on the walk-in, settle low-right of the
hero before the tap window — verified with him at MID_PT. Projectile wind-ups stay in standard
framing with a stronger push. **Cinema budget:** attack/imposing fire at most 2x per rolling 15s
(sticky per beat); blitz/intro/kill/endgame exempt.

Ring closure is FLAT 1.25x at every level as of v0.45 (`CFG.closureBase`; `closurePerLv` removed —
level-ups grant damage and grade windows, not speed), giving predictable first-target leads the
attack-shot glide can rely on. This supersedes the "+25% ring-closure speed per level" line in §7.

Harness: `camOn(v)`, `camShot(name[,t])`, `camTight(v)`, `camDbg()`, `camCheck(margin)`
(projects both real bounding boxes against the live camera; raw boxes overhang the visible
silhouette — screenshots are the acceptance test, camCheck is for regressions).
