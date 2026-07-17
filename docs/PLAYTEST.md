# CONSOLIDATED OWNER PLAYTEST — feat/containment wave (2026-07-15)

ONE pass gates the whole wave (61 commits, bc4445e…HEAD). No agent had a
browser; every check below was flagged by a builder in a commit body or by the
audit. **Chrome is PRIMARY** (this branch family exists because of the
Chromium/ANGLE compile ceiling) **+ one Firefox sanity lap at the end.**
This pass also SUBSTITUTES for the null SHADER-BUDGET report — §1 is the
budget checkpoint.

Record failures as KNOWN_BUGS.md entries; tuning notes go to the flagged
commit's dial. Devtools console OPEN the whole session (the fatal trap +
`__bootPhase` live there).

---

## §0. Setup

- Chrome, this worktree's `index.html`, devtools console open.
- Debug console handy: `radBurn`, `crowdCap`, `viewDist`, `rayQuality`,
  `maxEnemies` are the dials this script touches.
- A save with a few raids on it AND a fresh NEW GAME both get used.

## §1. Chrome/ANGLE shader budget — THE GATE  (KNOWN_BUGS #2 · d73840b · 9f5ada6 · 654881b · 833460b · 8ea4d3c)

- [ ] Cold boot: loading screen → PREVIEW renders → full shader hot-swaps in.
      Watch the console: both full programs must LINK (fatal trap quiet), no
      sustained black screen. If AUTHORED fails to link, **revert d73840b
      (gas fog) alone first** — it is the wave's one real GLSL delta; the burn
      field stays honest via the HUD without it.
- [ ] Brief-black correlation: note `__bootPhase` (bootDiag) if any black blink
      appears — is it at BOOT (preview→full) or at the FIRST hideout↔valley
      swap? The prewarm (654881b) should have removed both. If it persists,
      report the phase; fallback dial is `RLIM.ENEMIES` 20→12-14 (~L737).
      NOTE: the goggles CROWD dial does NOT cut compile cost — only
      RLIM.ENEMIES does.
- [ ] Cap-20 arena: deploy to area1, pull the horde — 20 bodies render, no
      black screen, material band sane (nobody shading as a tracer blob).
- [ ] Walls + gate BOTH states: perimeter renders + collides; gate LOCKED =
      solid wall (walk into it), gate OPEN = notch carved in BOTH render and
      collision (walk through it).
- [ ] Menu-prop edit (9f5ada6) compiles clean in BOTH full variants — visit
      the hideout (authored) AND the valley, open/close the bag in each.
- [ ] P6 geometry: area1 renders + collides the five new crate clusters; two
      consecutive raids show DIFFERENT dark (rotated-out) crates; no bake
      throw on deploy.
- [ ] VALLEY + hideout render exactly as before; night still readable.

## §2. The P2 loop — keycard → gate → gauntlet  (c80d099 · 867217d · 4f00146 · 24facc5 · d4a1989)

- [ ] Card hunt: gateCard seeds in a VARYING unlocked crate raid to raid (not
      the same box every time).
- [ ] Unlock: GATE TERMINAL prompt names the card; card-less E refuses with
      HORDE KEYCARD REQUIRED; carded E unlocks; card NOT consumed.
- [ ] Walk out (any side — the trigger is position, not the gate): spawns
      start ~3s, ramp toward 0.45s with bursts, **OUTSK_MAX=16 cap holds**
      (never more live out there).
- [ ] Outskirts gas bites while you linger (see §8) — the clock is the tension.
- [ ] Return inside: interior density INTACT ~30s later (the gauntlet must not
      have eaten the arena roster); next trip out re-arms fresh.
- [ ] Exfil home, redeploy: gate reads LOCKED again, a card is seeded again.
- [ ] With the gate OPEN, items at the wall are grabbable (no interact dead
      zone); once P4 cards are in hand, the WRONG card refuses the horde gate.

## §3. DO-NOT-REGRESS movement chain  (16318d6 · 5640e5a · 58b66f2 · 0ec21fd · 1de141f)

The sacred chain FIRST, on flat ground:
- [ ] **sprint → jump → air-crouch → land-slide → hop-out** replays exactly as
      it always has. Then the same across lips and slopes.
- [ ] Every lip type mounts at walk + sprint + slide-hop: crate step, ruin
      wall base, truck slabs.
- [ ] Valley wall REFUSES at walk and sprint, from several angles — the 72m
      rim is containment again. Tunables if a legit lip refuses: raise the
      0.55 gradient factor toward (never past) 0.62, or shrink the 0.15 probe.
- [ ] Idle on 30-45° valley-side dirt: PLANTED — no skate, no bounce, no
      sawtooth. Crate/physgun shoves while idle still displace you.
- [ ] 20m+ drops onto the watchtower deck, roofs, truck slabs — at LOW fps too
      (FAST preset or a throttled tab): no tunneling. Jump/hop/rope feel
      unchanged.
- [ ] Rope: swing → release → mid-air crouch → landing SLIDES at swing speed
      and hops back out (the chain closes). Same run WITHOUT arming lands dead
      as before. Landing while still HOOKED still kills the slide.
- [ ] Enemies: soldiers no longer mount hip-high cover (they path around or
      hold); zombies still clamber ~0.9m crates AND the clamber completes (the
      pogo latch must not interrupt it); nobody face-pogos against 0.93-1.1m
      faces; chase across rough terrain still tracks.
- [ ] Flagged edge (balance-law-legal, owner may veto): a glue glob stuck to a
      steep wall smooth-mins the local gradient — the wall may read mountable
      NEAR the glob. Paid ammo = priced verb; confirm it reads as a feature.

## §4. Desk terminal lifecycle  (8f3fc88 · 49d7ffb · 2a989cf)

- [ ] Bag/manual/debug OVER the awake terminal: no double-drive — E sorts the
      bag only; two E presses mid-bag can no longer DEPLOY. Terminal re-wakes
      when the menu closes.
- [ ] Blind E while facing AWAY from the panel: nothing confirms (up/down/back
      stay live; Esc still exits).
- [ ] CLOSE row / Esc-at-home / pad-Circle: the desk goes dark AND STAYS dark
      inside the wake radius; stepping past ~1.4m lifts the latch.
- [ ] Walk past the desk mid-haul with a rope-carried item: no wake, nothing
      drops, pad-look never seizes (wake requires facing now).
- [ ] No stale interact prompt frozen on screen while the terminal owns E; no
      floating cursor glyph when the panel is off-camera; no open-flash on E.
- [ ] Desk framerate: no dip at the terminal (the reflow is dead); row
      hover/click still tracks the crosshair (one frame of hover latency is
      the accepted cost).

## §5. Bug #7 — the gun-raise replay  (9f5ada6)

- [ ] Close bag/rack/crate with B / SELECT with a tool worn: the gun is
      SIMPLY THERE — no raise replay, no rack-prop settling down-screen.
- [ ] Crate view shows the worn gun (the old code drew a phantom rack gun
      behind it).
- [ ] Tool-rack close: no double-draw.

## §6. Audio  (4842abb + the pre-wave spatialOut)

- [ ] **Pan SIGN first**: a shot from your LEFT sounds LEFT. spatialOut's own
      comment says flip the sign if reversed — this is the one listen nobody
      could do without ears.
- [ ] Distance gain curve: a cross-map firefight is quiet-and-placed, not
      "right on me"; enemy fire AND enemy reload localize.
- [ ] Localized: die() thud, world grenade tick (from where it lies), carve
      spark, LURE ping, ricochet + fuse-stick clicks, item-drop bounce.
- [ ] STILL CENTERED by design: your hit-confirmation sounds + helmet crack
      (hitmarker UX, owner call 14), onDeath/onBreach, warp arrive.
- [ ] Owner feel call (flagged, unbuilt): enemy melee wind-ups are silent —
      want an audible tell?

## §7. Render / perf  (b4f82c3 · 947d3f2 · 0c40d2d · 6d9e498 · 77e42a5 · c22a4ac · e55b2e9 · ac45862 · 4704ea5)

- [ ] Auto-res recovery on a 60Hz panel: induce a hitch (terminal visit /
      combat spike) — resolution sheds, then CLIMBS BACK within a few seconds
      (pre-fix it stayed shed until reload).
- [ ] Automatic fire beside a hideout lamp at night: NO strobe (the lamp must
      not blink out while muzzle flashes die).
- [ ] Walk a lamp edge light→shadow: the hands/tool dim RAMPS (no 8Hz steps);
      stealth detection behavior unchanged (AI still reads the stepped field).
- [ ] FAST preset: judge the structure-edge shimmer (#6c) — preset tradeoff or
      worth a jitter experiment? Owner call, note it.
- [ ] Pin 12+ items with the physgun, then fire: BULLETS STAY VISIBLE; the
      farthest pin needles are what drop.
- [ ] VIEW=CLOSE in an item-dense camp: nearby loot always renders (items
      behind the fog wall no longer starve the 16 slots).
- [ ] Outskirts gauntlet frame-time/GC before-vs-after feel: the 36-body
      roster should run visibly lighter (dormant 4:1 + alloc sweep + cached
      ranking). Watch the dormancy boundary in the 20-body horde: no enemy
      blinking rendered↔invisible at the cap edge.
- [ ] HUD: pips advance on fire/cycle-reset, EMPTY HANDS clears them, red-dot/
      holo glass still swaps tint with the mounted optic.

## §8. P3 — gas, masks, filters  (47fdb31 · b65bb23 · 2229606 · d73840b)

- [ ] Hazed POIs read GREEN from across the map; the gate notch shows the
      outskirts glow through it; VALLEY/hideout show no haze anywhere.
- [ ] Unmasked in a hazed POI: burn pool grows, ceiling scars follow, screen
      tint + the toxic symbol (red pulse = bare lungs); the dose is priced
      survivable-maskless per the BALANCE LAW (dial: `radBurn`).
- [ ] Masked with 2 live filters: −90% — a trickle only; symbol green.
- [ ] Filter % drains ONLY while gassed; a spent filter announces itself;
      save/load round-trips a part-lived filter.
- [ ] Both masks equip (Filter Mask rides the face slot UNDER a helmet);
      [X] opens filter seats; a mask refuses non-filters; a broken Respirator
      Helm SPILLS its seated filters.
- [ ] Masks + filters drop from crates and from masked zealots.
- [ ] Soldiers: an unmasked squad lured toward a hazed POI holds the RIM and
      fires — never wades in; masked zealots stand in it; a Believer chases
      through and COOKS (very them); the roving brute's aura parts a knot and
      bites you up close; step past the gate notch — outskirts gas ramps over
      ~6m.
- [ ] LURE in a cloud: undead answer, soldiers do not. (LURE on undead
      generally: stick one near idle shamblers — they walk to it; fired past a
      mid-chase zombie after breaking LOS — it diverts.)
- [ ] Hazed POIs feel meaningfully THICKER with undead than the zombie zones
      (double round-robin weight).

## §9. P4 — locks, unique cards, lockboxes  (fdedc19 · 73a7499 · b7c6a1d)

- [ ] Card-less lab/testing gates read sealed AND collide sealed on BOTH
      sides; the prompt names the lock and tints the required card in its
      color (card art matches its lock).
- [ ] Carded unlock opens render + collision TOGETHER, same frame; locks
      re-seal every camp raid; cards persist.
- [ ] The exTesting exfil now sits BEHIND the range lock — confirm it reads as
      reward, not trap (two open exfils remain).
- [ ] Lab/testing cache crates render + collide inside both yards; loot rolls
      match tier character (lab gear/rare/cores, testing mods/common/rare);
      guaranteed cards NEVER seed behind a lock.
- [ ] Lockbox loop: kill the brute until an epic drops → bag it → extract →
      the box STANDS in the hideout + INSTALLED toast → stow loot → die in the
      field → the loot SURVIVES. Save/load with boxes owned and with none;
      START OVER clears ownership.

## §10. P5 — hitboxes, headshots, spawns, morale  (2da355e · 0c0dc37 · 2a9d5b5 · 054e5ed · c398faa · 1c06127 · ec43ecd · 77541a6)

- [ ] Grazes that visibly MISS a body stop paying; the feel edge to watch is
      leg/foot shots at the capsule floor (y+0.20).
- [ ] Zombie bare-head shots ONE-TAP; a helmeted zealot takes 2 (light) / 3
      (heavy); the brute head pays ×4 and its enlarged head registers at its
      drawn size.
- [ ] A lucky enemy round on YOUR bare head kills — confirm it reads FAIR, not
      cheap (they aim center-mass; spread makes the lucky one). A worn helm
      eats it with HELMET CRACKED, shatters when spent, spills seated filters.
- [ ] A beam held on a head strips a helm in 1-2 crack cadences — priced and
      aimed; confirm it reads as skill, not exploit.
- [ ] Raid's first minute: the two soldier knots form up, the swarm crowds the
      hazed POIs, first contact happens ON THE MOVE, not at spawn (60m
      faction separation).
- [ ] Nobody spawns on raised ground/walls (yMax 3.0); watch a full chase —
      the vertical-gradient stepOver means enemies can no longer walk up
      steep faces mid-chase either. **Owner read: is the chase-up-slope
      asymmetry acceptable now?** (The audit left this to your eye.)
- [ ] Morale: pressure a knot hard — SOME zealots visibly bolt and regroup,
      SOME dig in; nobody flip-flops frame to frame; Believers still walk
      their bomb straight in; believer squads STILL FIRE (the suppressor slot
      exclusion).
- [ ] Crowd a horde at CROWD=SPARSE: no damage lands from empty air (the
      dormant lunge gate).
- [ ] Brute roam: watch its commitment across the map — owner call 10 kept the
      current roam; the far-trek variant stays flagged if this reads samey.

## §11. P6 — crates + rotation  (833460b · 8ea4d3c)

- [ ] Every POI has its loot layer; the once-crateless north half included.
- [ ] Two consecutive raids: a DIFFERENT crate sits dark per rot cluster; the
      breach caches / hideout clutter / valley never rotate (paid-for rewards
      and home storage stay as-described).
- [ ] Loot volume feels ~constant per raid (46 live seedable crates vs 43
      pre-P6 — spread wider, not richer).

## §12. P7 — ADS free-aim  (b9ccf28 · 8318dfb · b0af3a8 · f4cb1bf)

- [ ] FREEAIM row on the goggles toggles it; default ON; the dial persists.
- [ ] Bare tool + iron sights: the AIM LEADS, the camera TRAILS (~100ms glide);
      the gun leans with the lead; leftover lead drains smoothly when ADS
      drops.
- [ ] **The hit-test follows the CROSSHAIR, not screen center** — fire with the
      marker deflected to the ring edge and confirm impact lands at the marker.
- [ ] Recoil rides the LEAD (marker jumps, camera glides after); total climb
      feels identical to the rigid path.
- [ ] Iron-glyph handoff: while the DOM post-and-notch owns the screen the
      shader glyph sleeps — no double-draw, no glyph pop at the ADS threshold;
      marker fades over the same band the shader used.
- [ ] Glass optics (dot/holo/2x/3x): CENTER-LOCKED as before — free-aim must
      not touch them (extending it to glass is a flagged ~2-line shader
      change, deliberately unspent — owner call).
- [ ] Physgun/heldWorld still forces rigid aim.

## §13. Economy / UI reads  (24facc5 · 0c676bd · 5afd480 · 95126b5)

- [ ] No duplicate compass/photometer after exfil arrivals (the two-deep
      ownership walk).
- [ ] PERSONNEL RECORD: legible at the FAR half of the wake band (the k=0.8
      floor renders meta ~11px — **eyeball the slab look; push the floor
      higher if it still squints**, flagged in 95126b5).
- [ ] PR counts ONLY your undead kills — park near soldier-vs-zombie crossfire
      and confirm their kills don't pad your score.
- [ ] Enum dials: debug-set `viewDist 0.9` → the widget shows the RAW value,
      not CLOSE; first nudge snaps it to a preset.
- [ ] Drop loot mid-knot-fight and fight for 2+ minutes: do drops vanish
      before you can loot? (KNOWN_BUGS #13 is OPEN — age/frozen guards didn't
      land; this read calibrates its priority.)
- [ ] Hideout floor drops: NOTE the 120s reap still runs at home (owner call 5
      approved the exemption, unbuilt — KNOWN_BUGS #13). Don't read vanishing
      home-floor loot as new.

## §14. Firing-model browser pass — owed since the pre-wave combat rework  (3c2430b…bc4445e, KNOWN_BUGS #3)

- [ ] Two-pool: a boltless mag throws plain lead PARALLEL to the energy cores
      (pools/counts drift by design); a bolt core puts ammo IN the cycle.
- [ ] Powder mag: any caliber feeds; with a bolt core its relay mods cost 0⚡;
      REFUND waives base ⚡ ONLY — lead is always spent.
- [ ] burstCam + POWDER + BEAM mint gate: mag −3 per pull, no free shots.
- [ ] Hitscan pulse feel/converge; drum beats land at the dot; thread
      degradation draws where it damages (damage-follows-draw); grapple swing
      feel; whip takedown reads.
- [ ] Blast parity flavor (the #3 survivor): a point-blank CONTACT/TIMER
      delivery deals direct + splash — confirm it reads as flavor, not error.
- [ ] BEAM + mag lane shapes (post-rework regression fix): a lane shaped
      [non-laser core, …, BEAM→core] THREADS its beam (this was dead);
      [BEAM→core] + mag threads too; the mag emptying dry-clicks the bolt
      without touching the thread.
- [ ] R-rack a mag into an empty slot mid-combat (menus closed): plain lead
      resumes IMMEDIATELY — no rack-menu visit needed.
- [ ] Wing mags: an emptied WING mag dry-clicks its lane (never siphons the
      base reserve) — including relay extra-casts and CONTACT drum beats. NOTE
      (owner call): wing mags have no hot-reload path while a base mag exists —
      refill via the rack menu; flag if that feels bad mid-fight.
- [ ] A REFUND bolt shows no FREE tag (it pays lead + the hard surcharge); a
      mag-only wing lane's tooltip reads "BOLT ⚡0·1rd", not "—".

## §15. Firefox sanity lap

One lap, nothing Chromium-specific: boot clean → one full raid loop (card,
gate, gauntlet, exfil) → gas visuals → free-aim → desk terminal → save/load.
Firefox ran clean when Chromium black-screened before; it is the control.

---

## NOT in this build — don't chase these as regressions

Audit items queued but NOT landed this wave (all open in KNOWN_BUGS.md):
hideout crate faucet still reseeds every deploy (#9) · save-restore integrity
guards + corrupt-LOAD rollback (#10) · four unobtainable items + settings
gadgets not in the hideout guarantee (#11) · the horde gate still has NO
physical terminal body (#12) · worldItem age/frozen/hideout-reap guards (#13)
· the starter kit is unchanged (#14) · uniform batching (#15).

---

# CREATURE WAVE addendum (2026-07-16, 22 commits, 3b1075f…HEAD)

The P8 enemy-AI overhaul + idle ecosystem. Same rules as the pass above:
Chrome PRIMARY, devtools console open, failures become KNOWN_BUGS.md entries,
dial notes go to the flagged const (ZAI/BRT/HORDE/CRIT blocks ~L6910). The
SwiftShader smoke gate passed after every commit — these checks need a REAL
GPU and eyes. Deploy area-1 unless a line says valley.

## §C1. Compile budget FIRST  (KNOWN_BUGS #2 — the standing gate)

- [ ] Real-window Chrome load after the GLSL-touching commits (bodies
      3b1075f/a85b87d, gas+glass ee7631e, gibs d594882): boot clean, no black
      screen beyond the known brief flash, `__bootPhase` sane. Wave shader
      delta: a 12-prim pose rig REPLACING the old enemy block (~+5-10% net
      compiled, executed 12/10/10/6 prims by kind), critter/gib itemSdf
      branches, D.w bit strips 128→2, gas billow + DOSED glass post.
      `RLIM.ENEMIES` untouched at 20.

## §C2. New bodies on all factions

- [ ] Soldier: robe+torso cones, bent limbs, hood/helm/plates still read; arms
      RISE in alarm (state 2). Zombie: shamble slump + head loll. Brute:
      knuckle bulk. Charging brute: lean + hand plow + heat tint (+32).
- [ ] Believer still renders a gun (KNOWN — owner-flagged render tell).
- [ ] Critter: small quadruped, trot legs, tail wisp, beady eyes; its corpse
      lies over (pitch 1.45) while it lingers.
- [ ] Headshots land on every kind — the JS head/body twins moved WITH the
      geometry (slump/loll/charge/flinch offsets; critter head sphere
      +0.40/r0.12, horizontal spine capsule r0.19).

## §C3. Brute charge

- [ ] Stand ~10m off in the open: growl + body glow is the 0.6s tell; the
      sprint is a straight 3x lane — a sidestep beats it clean.
- [ ] Eat one on purpose: ~30 dmg + a real view kick. Bait it into a
      truck/wall: 1.4s stagger window, boom + 20m noise ping. Re-arms 7-11s.

## §C4. Zealot combat AI

- [ ] Alarm a knot: the cover role tucks, leans ±1.1m out to fire ~0.9s,
      tucks 1.4-2.2s — the lean is the only exposure. Shooting one forces a
      duck (any role; reloading ducks too).
- [ ] Wound one below 35%: it breaks for cover and wraps the wound 2s
      ('reload' cue, 'pickup' on completion); shooting it interrupts. ~35%
      of carriers drop a spare bandAid.
- [ ] Backup shout: a pressed soldier rallies the camp from 40m — the
      converge jogs (1.15x). The LAST soldier standing always runs — toward
      an ally ≥12m or home, never through your guns.
- [ ] Suppression: while a squadmate repositions/heals and you're out of
      sight, the suppressor hoses your last-known cover BLIND — stand behind
      a truck and hear rounds crack into it.

## §C5. Horde + food chain

- [ ] Find the wandering brute: the escort wheels a golden-angle ring, the
      gait ripples around it; every 9-18s a moan rolls outward in staggered
      answers with a short lunge pulse.
- [ ] Critter huddles (2 groups of 3, seeded off the hot camps): graze and
      nibble; over minutes the herd DRIFTS together (anchors creep, knit past
      6m, fenced off gas/walls/rim). Walk inside 7m: an 8m startle ripple
      bolts the huddle around the hazards.
- [ ] Let a zombie hunt one: 1.35x chase, bite, ~4s head-down feed → the
      sickly green pulse (+64). Stand inside 4.5m masked then bare: the burn
      tick scales with distance AND mask — radiation dread, the glow IS the
      zone. Geiger cadence + DOSED glass + rad motes agree with the field.
- [ ] Critters never alarm soldiers and never tick the HUD kill counter;
      seeker/chaos threads DO read them as warmth (owner call — flag it).

## §C6. Idle ecosystem — watch a calm camp do NOTHING for 2 minutes

- [ ] Two posted zealots drift together (POI cohesion), stop in arm's reach,
      face off and sway ~4-7s, then part (~25-45s cadence). Believers join
      the circle too. Any alarm/damage/distance drops it mid-word.
- [ ] Calm zombies rotate habits on a 6-12s clock: mostly drift, some stand
      head-down swaying (stillness is the tell), some twitch-dash; feed rolls
      moan about half the time.
- [ ] Kill something near calm zombies and wait one idle roll: a zombie walks
      to the gib pile (≤12m) and mouths it — piles last 14-20s; reaped
      mid-walk it worries the stain instead.
- [ ] Fireflies at night: warm dots only OUTSIDE the gas (the dark line marks
      the field). SPRINT through the swarm — flies dart dark off your body
      and re-light ~1s later; a plain WALK leaves them lit. A chasing zombie,
      charging brute, or bolting critter punches the same dark hole.

## §C7. Gibs + respawn feel

- [ ] Kills burst faction-tinted chunks (zombie green-black · soldier
      red-brown · brute slabs · critter pale), never lootable, never a
      prompt; the 14-meat cap self-evicts quietly.
- [ ] Hold a camp 6+ minutes: dead soldiers return on the undead 5-10 min
      clock — the 45s revolving door is gone. The brute keeps its 15-min
      session clock.

## Dials if it plays wrong (all consts, one line each — see KNOWN_BUGS
Backlog "CREATURE-wave owner calls + dials" for the full ledger)

#radTint alpha .38→.30 (over-green) · FLY_VIS / duty gate 0.08 (firefly
density) · gib counts + meat cap 14 · chat cadence · zombie idle mix
(0.62/0.24/0.14) · ZAI/BRT/HORDE/CRIT ~L6910.
