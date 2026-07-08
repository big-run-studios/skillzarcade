# Spy Runner — Handoff / Status

> Read this first when picking up the project in a new session. It captures the current state of the web prototype, how the code is organized, the tunables, how to run/host it, and the non-obvious gotchas. Companion docs: **UNITY_MIGRATION_PLAN.md** (the plan to rebuild on Skillz Arena) and **ARENA_QUESTIONS.md** (open questions for the platform). Long-term project memory also lives in the Claude memory file `crasharcade-car-game.md`.

## What this is
- Workspace: `/Users/bigrunandrew/Desktop/CrashArcade`. Owner: **Andrew, Big Run Studios**.
- The active deliverable is the **car game** = `spy-runner/index.html`, a single-file Three.js prototype (Three r128 from unpkg CDN). `go-chicken-go/` is a separate copied game, not the focus.
- Goal: evolve the web prototype, then **rebuild in Unity for the Skillz Arena platform** (see UNITY_MIGRATION_PLAN.md).
- Current version tag is in the footer `.brand` text (currently **v0.9.17**). Bump it on every change.

## The game (mechanic)
Skill-based **crash / cash-out** endless driver (sweepstakes-style):
- 4 driving lanes + a right **shoulder** (5 columns total). You auto-drive; steer between lanes.
- Left lanes pay a higher **multiplier** (`MULT = [4,2,1.5,1,0]`) and drive faster (`SPEED_MUL`). Start in lane 2.
- **Score = laneMultiplier × heat × time.** Heat (1×→5×) is filled ONLY by **near-misses** (`HEAT_BUMP` per near-miss); no passive build.
- **Near-miss** = you commit a lane change (opens an `NEARMISS_WINDOW` dodge window) and brush a car in the lane you entered/left without hitting it. Rewards dangerous weaving.
- **Cash out** (veer onto the green shoulder, or hold the Cash Out button) → banks **score × 10**. **Wreck** (hit traffic) → banks **score × 1** (raw reached score, never 0 — this avoids leaderboard ties). Both update the persisted **personal best** (`localStorage sr_best`).
- Fair wave spawner: the open "gap" lane drifts ≤1 per wave, so traffic is always dodgeable by skill.

## Code map (all in index.html)
- **Config constants** (~lines 256–281): `LW, NLANE, MULT, SPEED_MUL, START_LANE, SHOULDER`, heat/near-miss consts, intro-timing consts, curve consts. This is the main tuning surface.
- **SKINS** object: `neon` (default), `spyhunter`, `bootlegger`. Each has palette + `setupLights/buildTrack/playerCar/vehicle`. Neon uses hot-pink rails; lanes L→R are electric-neon **red, orange, blue, yellow**; **shoulder = neon green** (cash-out). Visuals are skin-isolated; game logic is skin-agnostic.
- **Curve** module (deterministic, distance-sampled): cosmetic OutRun-style hills+bends. Gameplay stays on the straight logical track — curves are visual only.
- **buildRoadRibbon**: the road is ONE continuous deformable BufferGeometry mesh with **lanes baked in as vertex colors** (unlit MeshBasicMaterial). This is what eliminated tile tearing/black seams on bends. Only thin dashed dividers/rails/chevrons remain as tiles on top.
- **buildPerson / buildGuy / buildGal / seatPerson / standGalWaving / buildCop**: the intro characters/cars.
- **STATE** = MENU / INTRO / PLAY / CRASH / CASHED. `startIntro → (updateIntro) → beginPlay`; `crash()`, `cashOut()`, `backToMenu()`.
- Main loop: `loop→frame→update(dt)`; there's a `setInterval` safety-net that also calls `frame()` (see gotcha on rAF throttling).

## Intro cutscene (~3.6s, plays on every START / PLAY AGAIN)
Tuxedo **driver** runs in toward camera → front-flip-twist over the windshield into the **left/driver seat**; blonde **partner** waits in the driver seat waving, then slides to the passenger seat as he lands. **Two cop cars** chase down the adjacent lanes then **turn away and peel off-screen** (dropping toward camera so they never overlap the car). Then the car "takes off": it drives away over a **still road** (camera parked, `dist` does NOT scroll during the hang) and when the cops turn the **camera catches up** — the road scroll is coupled so the car never appears to stop/reverse, ending matched to play speed (no lurch). Timing consts: `RUN_END, JUMP_END, COP_PEEL, CAM_CATCH, INTRO_DUR`.

## Audio
- Per-skin loop tracks: neon→`neon-pulse.mp3`, spyhunter→`spyhunter.mp3`, bootlegger→`bootlegger.mp3` (swap files to reskin, no code change). `music.mp3` is the old unused original.
- **Wreck stinger** `wreck.mp3` (~3.5s) plays once on crash. `syncMusic()` plays the loop during INTRO/PLAY/CASHED and pauses it on CRASH/MENU; music resets to 0 on each start.
- **Mute** toggle top-right (`#muteBtn`), persists `localStorage sr_muted`, mutes both tracks.
- **Wreck-screen does NOT auto-return** (removed) — stays until CONTINUE.

## Controls
- Mobile: **swipe left/right** (fires on crossing ~26px) OR **tap a side** (fires on release). Desktop: **arrow keys / A-D**, or click a side. `C` = cash-out hold.
- The two end screens show the score **math** (`raw × 1` on wreck, `raw × 10` on cash-out).

## Run / host it
- Preview server: `.claude/launch.json` config named **arcade** runs `node .claude/serve.js` (autoPort; tries 8778, falls back). Use the preview_* tools. `serve.js` 302-redirects `/` → `/spy-runner/index.html`, honors byte-range (media), and reads `PORT` env.
- **Play on phone (same Wi-Fi):** a background server is/was run via `PORT=8080 node .claude/serve.js`; open **http://10.0.0.115:8080** on the phone (this Mac's LAN IP — reconfirm with `ipconfig getifaddr en0`). QR generator available via the `segno` pip package (installed).
- No tunneling tool installed (no public URL / off-Wi-Fi access yet).

## GOTCHAS (important)
1. **Mobile audio unlock**: iOS/Safari only lets an `<audio>` play programmatically if it first played inside a user gesture. `music` is unlocked because it plays on the START tap; **`wreck` must be "primed"** (played silently then paused) inside `startIntro` (the START tap) or it's blocked on phones — this was the "wreck music broke on phone" bug (fixed v0.9.16, `wreckPrimed`). If adding new SFX, prime them the same way. Also: iOS **hardware silent switch** mutes Safari `<audio>` (would need Web Audio API to bypass) — rule that out before debugging code.
2. **Preview rAF throttling**: when the automation tab isn't focused, `requestAnimationFrame` throttles and the game runs at ~1/5 speed via the setInterval safety net — so automated tests may "never reach PLAY" or "never crash." This is a test artifact, not a bug. Verify timing-sensitive things at real speed / on device.
3. **Fog color = background color** per skin, so the road dissolves into the distance with no hard far edge. Don't change fog color without matching bg.
4. **Curves are cosmetic only** — collision/lane logic uses `laneX(lane)` on the straight track; traffic/road just get a visual x/z offset. `FLAT_LEN` forces a flat ~10s opening (clear orientation + keeps the intro/cops on flat ground). High hill/bend severity can facet slightly.
5. Synthetic touch tests must dispatch on a real element (not window/document), or `e.target` lacks `.closest`. `isUI()` is guarded against this now.
6. **This file is edited by multiple sessions.** Recent external additions you may see: impact **juice** (`hitstopT`, `shakeT`/screen-shake, `spawnDebris`/`spawnCoins`, `flashScreen`/`#fxFlash`, `crashRevealTimer`/`cashRevealTimer` that delay the result card ~0.6–0.7s after impact), and skin `hover` variants. Re-read the relevant function before editing; don't assume the code matches this doc exactly.

## Key tunables (current values)
`HEAT_MAX=5, HEAT_BUMP=0.5, NEARMISS_BONUS=18, NEARMISS_WINDOW=0.8` · hitboxes (v0.9.17, more forgiving) `PLAYER_HALF=11, PLAYER_HALFW=6` (wreck zone; smaller = harder to wreck) and near-miss grace shell `NEARMISS_X=20, NEARMISS_Y=40` (bigger = near-miss registers from further out) — collision test at ~line 949 is `dx<halfW+PLAYER_HALFW && dy<halfLen+PLAYER_HALF`; near-miss is the shell just outside · intro `RUN_END=0.85, JUMP_END=2.1, COP_PEEL=2.6, CAM_CATCH=1.0, INTRO_DUR=3.6` · curve `SEG_LEN=2200, FLAT_LEN=5000, CURVE_MIX=0.25, HILL_RISE=130/HILL_MAX=230, BEND_SHIFT=70/BEND_MAX=220`.

## Status / open threads
- **Wreck music on phone**: fix applied (v0.9.16 priming); **needs confirmation on a real phone**. If still silent, check mute toggle + iOS silent switch first.
- Original 4-phase plan: (1) gameplay depth ✅, (2) feel & controls (mostly done: swipe, juice), (3) visual direction (neon art pass done), (4) **Unity/Arena port** — see UNITY_MIGRATION_PLAN.md.
- "Score to beat" HUD target is still in raw-score units (not ×10 scaled); left as-is intentionally.
