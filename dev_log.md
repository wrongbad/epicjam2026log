# Dev log — 2026 Epic AI MegaJam

Turn-by-turn, high-level log of how this game is built, maintained by the orchestrator agent
(see [`AGENTS.md`](AGENTS.md)). The jam runs Nov 12–15, 2026.

## Pre-jam (declared setup, no game content)

### 2026-11-11 22:53 EST — Tooling and repo setup
- Installed Unreal Engine 5.9.0 Preview 1 on Linux (RTX 4090, Vulkan). Verified a throwaway C++ First
  Person template project builds and runs.
- Connected Claude Code to the editor through Unreal MCP, with all toolsets enabled.
- Seeded this repo: mirrored the contest page (`contest.md`), set up Git LFS for assets, and added
  agent instructions (`AGENTS.md`) and this dev log.

### 2026-11-11 23:05 EST — UE 5.9 feature notes
- Condensed the UE 5.9 release notes into `docs/ue59.md` so agents know the new engine features
  (MCP toolsets, Python authoring of Blueprints/StateTrees, Canvas, AI animation, Lumen World Cache, Web target).
- Looked at Hyper3D Rodin for AI 3D asset generation through its official Claude Code plugin. Setup is pending an API key.

### 2026-11-11 23:26 EST — Agent team: Archer (Fable / Opus / DeepSeek)
- Set up "Archer", a cost-tiered orchestrator on Omnigent. **Claude Fable** leads: it talks with us, shapes
  designs and writes short specs. **Claude Opus** handles design depth, tricky code and reviews. **Pi on DeepSeek**
  (via OpenRouter) does the bulk work: implementation, tests, builds, code reading and repo chores.
- Shared skills for the team: cross-review, fan-out and investigate.
- Hyper3D Rodin is now connected through its hosted MCP server (OAuth) rather than the plugin and API key.

### 2026-11-11 23:50 EST — Windows build pipeline
- Goal: get Windows builds out of a Linux-only dev machine, so a remote playtester on Windows can join.
- Set up a Windows 10 build VM (QEMU/KVM) with the matching engine build and Visual Studio Build Tools.
  QEMU runs inside a rootless Fedora toolbox container, because installing it on the host would have
  pulled in a partial distro upgrade the night before the jam.
- Packaged the throwaway template project for Win64 in the VM, driven over SSH from Linux: about two
  minutes per warm build, 438 MB zipped without debug symbols.
- Added a vendor-neutral agent skill, `.agents/skills/windows-build/`, that packages a project in the VM
  and names the zip and its debug symbols by the 8-character git commit, so a crash report maps to a commit.

## Jam

### 2026-11-12 00:01 EST — Game idea
- Theme is "Director's Cut". After brainstorming a few directions, the human picked a straight western:
  round-based outlaws-versus-law team fights in scenarios from classic western films, networked, with
  bots on both teams so it also plays solo.
- Theme link: our "director's cut" keeps the punishing gun mechanics a studio would cut, and each round
  is a film scene with a cinematic intro and outro. The gameplay itself stays straight and immersive.
- Two living documents hold this: `idea.md` for the game and `impl_ideas.md` for how to build and test
  it. Specs are generated from them and the orchestrator sequences the work from the specs.
- Planned an unattended bot-versus-bot test loop to find bot and map problems. Bots get human-level
  reactions and aim, fixed, so they can only improve through better decisions, not by becoming aimbots.

### 2026-11-12 01:00 EST — Art trial, foundation specs, project created
- The human went to sleep and asked the agent team to keep building overnight, committing as it goes.
- Art trial: a Hyper3D Rodin character, auto-rigged, plays the template mannequin animations unchanged.
  A generated lever-action came back as one fused mesh, so first-person guns will be built from separate
  parts and animated in code instead.
- Wrote foundation specs: one gun state machine for all four guns, human-capped bots in plain C++, a
  grey-box town built at runtime from a text layout, and bot matches that run without graphics.
- Created the real project (codename `Western`) from the engine's first-person template by script, with
  build and test scripts. The first automated test passes.
- Problem: the DeepSeek worker stalled three times on one task without writing anything; that task
  moved to Opus.

### 2026-11-12 01:55 EST — Core systems land overnight
- Merged the first town map (three outlaw routes, checked by tool for cover and sightlines), the gun state
  machine for all four guns, the character, round and match flow, and server-side shooting with
  black-powder smoke that blocks bot sight. 73 automated tests pass.
- Each piece written by the cheaper model is reviewed by Opus against the spec. Reviews caught real
  problems before any human played: a reload interrupted to fire never fired, queued shots used ammo
  without firing, bots skipped recoil, and bots would have seen over cover while crouched.
- Headless runs each started the editor's agent server on the same port and collided; every run now gets
  its own port.

### 2026-11-12 02:55 EST — Bots fight on the town; network play; first editor look
- Milestone: eight bots play full 4v4 matches on the grey-box town with no graphics, about 20 times
  faster than real time. Twenty matches take under two minutes and feed an analyser.
- First numbers from 82 rounds: the law wins 74%, 42% of deaths are in the open, and bots bunch at the
  vault. Next is working out which of that is the map and which is the bots, then a bot revision that
  uses cover, measured against the first.
- Network play passes a two-process test with 107 ms simulated ping and 2% packet loss: shots, hits,
  kills and deaths agree on client and server. Friendly fire is off for now.
- First look in the editor (run off-screen): iron sights line up on all four guns and the lever, hammer,
  cylinder and break-open motions work. Other players showed in an A-pose and crouching did not lower
  the visible body, both now being fixed.
- Problems solved: the town's navmesh only covered the spawn ends because boxes were registered before
  they existed; and builds silently went stale whenever another Unreal process was running, so tests
  passed on old code. The test script now fails if fewer tests run than are defined.

### 2026-11-12 04:28 EST — Menu, Windows build, cover-using bots, first sounds
- A main menu (play solo, host, join by IP), a pre-round gun picker and an end-of-match panel replace
  the console. The first Windows build was packaged in the VM and ran a bot match and a host-and-join
  there; packaging exposed four defects a Linux editor run never showed (the town's layout file
  unreadable inside the package, meshes loaded by path not included, 39 MSVC-only compile errors).
- The second bot revision (rev 1) uses cover. Against the first (rev 0) over 100 matches it wins 57% of
  rounds and dies in the open 31% of the time instead of 50%, with identical reaction and aim limits. Triage of the recordings
  says the law's 74% win rate is mostly bot behaviour (outlaws arrive one at a time, defenders always
  take the same two spots), so the map is unchanged for now.
- Other players now animate, crouch to match their hitboxes and carry a visible gun. The town got
  colours and interior lighting after a look in the editor.
- All sound is synthesised by our own script from numbers in a text file: four distinct gunshots with
  distance variants, lever, hammer and reload sounds, impacts, footsteps and wind. No third-party audio.
- Mistake and fix: a merge that broke the build was pushed because a filter hid the failing exit code.
  Merges now go through a script where any failing check stops the push.

### 2026-11-12 06:31 EST — A second map, the film framing, visible smoke
- The theme now shows: each round opens with a letterboxed title card, slate and establishing shot that
  settles into the player's eyes, and closes on an outro and "Cut." Nothing film-related appears during
  live play. Teams got distinct colours, hats and a badge or bandana.
- Second map, a snowed-in mountain pass with a stagecoach lodge, built from the same text layout format
  and balanced by bot batches alone: the law's win rate went 84%, 67%, 56%, 54.5% over four layout
  iterations. A "Feature" match plays the bank, then the pass, as two scenes of one film with a running
  score; connected players follow the host across the map change.
- Black-powder smoke is now visible and matches exactly what blocks a bot's sight. Before this, a human
  saw no smoke at all while bots were blinded by it.
- The third bot revision (rev 2) varies its positions each round, attacks together and answers a
  teammate's death. It ties rev 1 on win rate once bots can hear footsteps, and one tested change to
  the town made things worse and was rejected. Both kept as recorded negative results. We also found
  that human players had been fighting the weakest bots because of an unread setting; the fix is in
  progress.
- A strict build mode compiles the code in three groupings so Windows-only errors show up on Linux.

### 2026-11-12 08:13 EST — First human playtest: a crash and a missing lesson
- The human played for the first time. The game segfaulted 53 seconds into the first round, and he could
  not work out how to fire a second shot: nothing on screen says the lever has to be worked by hand.
- The crash was ours: loaded sounds were cached where the engine's garbage collector could not see them,
  so about a minute in it freed them and the next sound played from freed memory. Every automated run
  overnight had sound off, so none hit it. Reproduced in 5 of 6 rendered runs with random input, fixed,
  then 5 of 5 clean. A rendered soak test is being added to the merge checks.
- Real-session scripting before the playtest had already caught problems one machine never shows:
  joining clients with no map data, a shot fired after the round ended, and duplicate player ids.
- Human decisions: bots should have individual personalities (flanker, pusher, cautious) so teammates fan
  out instead of leaving spawn in a line, with no campers; dry humour is welcome around the rounds; and
  each round ends with awards in the spirit of The Good, the Bad and the Ugly.
- On-screen control prompts that fade once learned are in progress. Auto-cycling stays out by design.

### 2026-11-12 09:05 EST — Awards on screen, real guns as reference, cheaper agent sessions
- The game has a title and tagline: Polvere, "Ashes to ashes, gunpowder to dust." Polvere is Italian
  for both gunpowder and dust. (A first pick, Print the Legend,
  turned out to be taken by an existing game.) Instead of one clever title carrying the film theme,
  the game will be dotted with small unannounced references to western films, on signs, props,
  headstones and in award citations. The theme no longer stops dead when a round starts: it fades
  during the action, and the director yells "Cut!" when the round is decided.
- Human idea, started as a parallel track: a single-player bank scene where you cut between a lawman,
  townsfolk and a hostage the way a film cuts between shots, against outlaws who would win any fair
  fight. Each outlaw has a temper that rises with fidgeting hostages and noises and shows in his body
  language; anyone he sees armed gets shot at once. Both modes are built side by side until playtests
  pick a lead.
- Each round's outro now hands out The Good, The Bad and The Ugly with dry one-line citations, and the
  credits end on "Print it."
- Control prompts teach the hand-worked action and fade once learned. Bots fan out from spawn instead of
  filing out in a line. Loads from disk during a live round are gone.
- Art track started: characters with Hyper3D Rodin and guns with Meshy. Human direction: model the guns
  on photographs of real period guns, action closed and open, not on our box model. Only public-domain
  and CC0 photos go into generation.
- Process: long agent sessions were burning the usage limit, because each step re-reads the whole
  history. State now lives in short done/todo files under `wip/` and lessons in `AGENTS.md`, so
  sessions can be restarted often and cheaply.

### 2026-11-12 11:20 EST — Playtest feedback fans out into parallel tracks
- Human playtest verdicts: the new character bodies are nearly right but their arms twist "like curly
  fries" around the rifle; the game feels slow on foot; audio needs a lot of love (the human will lead
  that with his own recordings and a fast audition loop, parked for now).
- Started in parallel, each in its own worktree: an arm fix, hold-to-sprint, period props, the
  single-player bank scene, and a lighting and atmosphere pass (sun shafts, dust, clouds, distant
  mountains, weather). The solo scene is already playable end to end on its branch.
- Props are built to the footprints of the grey blocks they replace. Human decision: what you see is
  what you hit, so bullets and bot sight use the prop's real shape, characters collide with simple hulls,
  and the block survives only as the navigation body. One server-wide switch flips a match between
  block and art. Simple props (crates, barrels, hay, fences) are generated by script; wagons and the
  stagecoach will come from Meshy.
- Bots with personalities (a pusher, a flanker, cautious ones, each with an epithet) are now the default
  opponents after batches showed faster first contact and no balance loss.
- Process: Claude credits ran low mid-session, so the expensive model now only does the parts that need
  judgement and writes briefs; a cheaper model (DeepSeek, through the Pi agent) does the bulk and the
  long check runs.

### 2026-11-12 16:05 EST — Six tracks land on main; clips replace playtests for art feedback
- Merged and playable: hold-to-sprint, the arm fix, the single-player bank scene (four characters to cut
  between, outlaws with tempers, "Cut!" and Take 2 on a loss), all four guns as art, the town dressed
  with period props, and a first lighting and atmosphere pass (low sun, fog, sun shafts, clouds, distant
  mountains, weather). Each new look can be switched off from the console.
- The curly-fry arms had two causes: the generated bodies rest in a different arm pose than the
  animations assume, and their twist bones carried no skin. Fixed in the importer and at runtime with no
  new generation. A review of sprint caught what the automated checks missed: sprinting while aiming was
  faster and quieter than running, and the server never enforced "gun down while sprinting".
- Props: a script-built kit (crates, barrels, hay, fences, stairs, an outhouse, a well) plus a wagon,
  stagecoach, market stand and water cart from Meshy, textured with CC0 materials from ambientCG. Human
  decision: open props are see-through and shoot-through cover. The generated vehicles did not match
  their blocks' proportions; they keep their real shape rather than being squashed to fit.
- Human direction on process: merge early and fix forward on main; no interest in grey-box comparisons,
  show the proposed meshes in one batch. That became named, permanent QA clips (turntables and
  first-person gun actions, stitched with ffmpeg) and a small local review page where comments typed
  under a clip append to a feedback log the agents read.
- Problem: the cheaper model cannot see its own renders, so visual work went through blind rework (props
  that rendered near-black, untextured generated meshes) and several of its sessions died mid-task.
  Trial under way: the stronger model acts as art director, judging images and writing one small task at
  a time, while the cheaper one does the editing and rendering.

### 2026-11-12 18:48 EST — Art direction by relay; recoil, powder smoke and honest clips
- The stronger model now leads a visual stream as art director while the cheaper one does the editing
  and rendering, one small task at a time. Four rounds on the look cost 19 cheap turns and found what
  blind measurement had missed: the desert looked like snow because the backdrop terrain's triangles
  faced down and were never drawn. Result: golden-hour town, adobe and timber walls, a readable bank
  interior, trodden snow on the pass.
- From the human's clip comments: guns now kick and spring back per gun, and black-powder smoke shoots
  forward and mushrooms instead of puffing as smooth spheres. Firewood and other round props got more
  sides.
- The first batch of QA clips was misleading: gun turntables showed parts unassembled, action clips
  started recording after the shot, and an auto-run setting leaked from one clip into the next so
  characters walked into walls. Human call: every clip owns its own controls. Clips now run as isolated
  sessions and fail their check unless the logged action happens inside the recorded window.
- The props that rendered near-black in the town were a material multiplying a photographed texture by
  a dark tint, not a lighting problem.

### 2026-11-12 22:24 EST — The maps are the problem: buildings as kits, and a tornado idea
- Human verdict after a day of polish: the town is boring. Buildings are hollow boxes with two doors and
  a window on a flat grid, and no amount of bot or balance tuning fixes that. Bot and balance tuning is
  stopped. A wave-survival pivot was considered and rejected as overdone.
- New direction: buildings are assembled from small script-built parts (plank walls, door and window
  panels, see-through stud frames, floors, stairs, roofs) into ten types with distinct roles in a
  fight, from a two-storey saloon to a bell tower. The human reviews one building or one room at a
  time as clips, the way props are reviewed; towns are then generated from them, scored on sight
  lines, routes and choke points, and he picks from a shortlist.
- Human idea, written into the design: Twister, a last-one-standing round where tornadoes tear the
  town apart piece by piece until no cover is left and a dust storm finishes the stragglers. It is why
  every wall and roof is a separate removable part. The game also has a tagline now: "Ashes to ashes,
  gunpowder to dust."
- Generated props took four rounds with a reviewer who can see: the wagon had no wheel rims because we
  shipped Meshy's reduced remesh instead of its original mesh, and every generated prop looked like
  clay because our own material blended them 45% towards white. The sack pile was replaced by a
  script-built one. The vehicles did not match their blocks' proportions; human call: the blocks change
  to fit the props.
- Clips now cover both character bodies and every gun action on an empty stage, and they exposed a
  game bug: other players' bodies never visibly aim up or down. The review page gained comments in
  the page, before/after tabs, a size slider, and it only loads the clips on screen.

### 2026-11-13 01:45 EST — The gold run lands on main with a real train
- Human ask for the night: the gold run fully playable, with all its mechanics, good looks, varied
  buildings and tested bots. Five parallel streams started: mode, buildings, tornado, HUD, shop.
- The extraction mode is merged: loot, safe, gold dropped on death, two trains on a timetable, the
  closing dust ring, a stash between rounds and bot raiders. The placeholder boxes became the generated
  4-4-0 locomotive, tender and boxcars with sliding doors, on track tiled along the whole line.
- The one red network check was a test that ended the round 1.4 s before the train left; gold only
  counts when the train departs. The handoff's diagnosis pointed elsewhere: read the newest log first.
- The destruction runtime gained its support rule (nothing floats), and the buildings branch was
  reconciled with it so both can merge.

### 2026-11-12 18:48 EST — Art direction by relay; recoil, powder smoke and honest clips
- The stronger model now leads a visual stream as art director while the cheaper one does the editing
  and rendering, one small task at a time. Four rounds on the look cost 19 cheap turns and found what
  blind measurement had missed: the desert looked like snow because the backdrop terrain's triangles
  faced down and were never drawn. Result: golden-hour town, adobe and timber walls, a readable bank
  interior, trodden snow on the pass.
- From the human's clip comments: guns now kick and spring back per gun, and black-powder smoke shoots
  forward and mushrooms instead of puffing as smooth spheres. Firewood and other round props got more
  sides.
- The first batch of QA clips was misleading: gun turntables showed parts unassembled, action clips
  started recording after the shot, and an auto-run setting leaked from one clip into the next so
  characters walked into walls. Human call: every clip owns its own controls. Clips now run as isolated
  sessions and fail their check unless the logged action happens inside the recorded window.
- The props that rendered near-black in the town were a material multiplying a photographed texture by
  a dark tint, not a lighting problem.

### 2026-11-13 03:10 EST — Tornado, buildings and ladders join the gold run
- The tornado is a game feature now: it follows a networked path, tears kit buildings apart module by
  module (anything left unsupported falls), throws the boards each wall is drawn with, and pulls and
  throws players. The gold run storm runs it once per round.
- The building kit grew paint schemes, false fronts, porches, gable roofs and furniture; saloon, general
  store and bank are the first three templates, each with turntable and walk-through clips.
- Gold run gained its HUD, title and results cards and seven procedural sounds (whistle, chuff, doors,
  safe, coins, dust wind). Ladders merged.
- First 20-seed bot batch: stable and gold conserved, 41 of 160 raiders got out. It also showed six
  deaths with no killer at the moment the early train arrives, and the safe cracker almost never
  escaping; both are being chased.
- Lesson: file-scope names need a unique prefix, since unity builds put files from different streams
  into one translation unit; main briefly failed to build because two streams both had a `Hidden`.

### 2026-11-13 05:55 EST — A real town for the gold run
- The gold run now plays in a town generated from the building kit by one script: a main street that
  widens into a square and bends, back lanes, a depot whose porch is the platform, a bank vault with
  one door. The tornado takes it apart board by board (717 modules in one round), at 7 ms a frame
  for the whole town and about 13 ms with the tornado in it on our RTX 4090.
- The kit reached eleven templates: church with a spire, livery stable with a hayloft, two-storey
  hotel, sheriff and jail, a frame house under construction, small house, cabin and shed joined the
  saloon, store and bank. Signs are lettered from per-letter modules; the first render read "YREVIL"
  because the whole kit is mirrored in the world.
- Bot batches found a bug a player would have hit: the boxcar steps kept their collision while the
  train moved and dragged raiders waiting at the platform edge out into the dust. Fixed, with a
  ride-out camera for those who make the train.
- The tornado's "dark column" was an unlit material writing no emissive; it now has a lit, spinning
  funnel, and the light darkens through the round as the storm builds. Third-person bodies got their
  shoulders back (span 0.79 to 1.06 of bind pose) and a left hand that holds the gun.
- Three worker sessions died mid-turn on an API sign-in error; because each stream keeps its state in
  `wip/` and commits often, restarting them cost minutes.

### 2026-11-13 07:00 EST — Sack, kegs and a first walk through the whole round
- Carried gold is now a burlap sack on the raider's back in four sizes, so who is worth robbing is
  visible across the street; it drops as the same sack. The safe went from three-quarters of the
  town's gold to a third, so looting rooms matters.
- Powder kegs: shoot one and it blows out walls, throws boards and raiders, and chains to its
  neighbours. Fourteen stand around the town, one against the bank's vault wall.
- Bot batches on the new town (20 seeds per change, one change at a time): stuck bots and abandoned
  loot went from 19 and 18 to 0 and 1; a tuning change that did nothing was written down and dropped.
  Bots had been jamming in the one-door vault behind whoever was cracking the safe; now they shoot him.
- A scripted first-time-player walkthrough, judged frame by frame, found what the checks could not:
  a tornado that outlived its round and wrecked the town under the results card and the next shop,
  and loot that was a brown box on a brown floor. Three rendered soak rounds ran clean.
- The tornado's frame-time spike was blamed on destruction; measured, destruction costs 0.4 ms and
  the cost was the funnel's see-through dust cards filling the screen.

### 2026-11-13 11:00 EST — First human playtest, and the review page becomes a ticket board
- Human played the gold run: "a massive improvement in fun factor", with a list of changes. Decided:
  one train instead of two, and it comes at once for the last raider standing on the platform; sound
  carries the news (train horn, a loud ratchet at the safe) instead of HUD banners; dying pulls the
  camera back to third person, fades, then spectates; the tornado has a survivable warning zone and a
  core that kills outright and throws the body about 30 m.
- The clip review page was rebuilt around how the human actually reviews: only renders from main, one
  card per scene, unread badges, and each scene a ticket (in review, change requested, in progress,
  closed) that he can also open for a scene with no render yet. UI screens are captured as stills.
- From his comments on clips: the tender's missing body was our own prep script cutting it away; the
  muzzle plume had never drawn at all (a fade that was always zero) and the clip stage hid effects;
  the storm now strips walls board by board, and a stricter support rule took floating structures
  after a blast from 59 to 0 in a randomized test. Kegs got a fireball, lingering smoke and a sound.
- Human rule after a morning of confusion: DeepSeek workers are sandboxed and never merge; Opus lands.
- The menu and HUD got one design language: big text links in a left column over the kit town, a film
  clapper for the end cards, a paper railroad ticket for the timetable, a coin stack for gold, a kill
  feed. Blurry HUD text was the engine's small built-in font scaled up 2-3x. New third-party assets:
  the Rye and Old Standard TT fonts (Google Fonts, SIL Open Font License).

### 2026-11-13 18:50 EST — Winner takes all, stores in town, and a collision bug found by measuring
- After a playtest the human reshaped the gold run: the bolt-on menu shop is gone and you now spend the
  gold you carry at counters in town (mercantile, dry goods, saloon); when the train leaves, the richest
  raider aboard takes it all and everyone else loses; seconds later a mega tornado wipes the town.
  Bot batches still end mostly as last man standing, so the town is being re-planned to break up
  sight lines (buildings offset like paving stones, no open main street), scored by a ray-cast metric.
- "I fell through the floor at the top of the stairs" and "there's a wall inside the boxcar" were one
  bug: our importer wrote collision box sizes as half lengths, so every kit mesh, prop and train car
  had half-size collision in the art build, and the test meant to catch it assumed the same wrong
  convention. A capsule sweep up every stair and through every car is now in the gate.
- Hands: both views now share one mannequin hand, so how a hand sits on a gun is being moved to one
  grip definition per gun with a contact-sheet check, instead of posing first and third person twice.
- Also in: shooter-only hit sounds and hit flinches, a walk-in bank vault that survives the storm, a
  wood material that no longer repeats, the Polvere title on the menus, and a code audit whose fixes
  cut a shot from 429 allocated objects to 6 and a building collapse from 8 ms to 2 ms on the CPU.
- Process: review clips moved into the repo under one folder per category; re-rendering an unfixed
  scene no longer counts as progress; the cheaper worker handled mechanical tool fixes on its own branch.

### 2026-11-14 01:16 EST — A night of two-player playtesting, turned into fixes as it happened
- The human played the gold run with a friend over a forwarded port and reported problems as they hit
  them; each became its own small branch, landed on main the same evening. The friend joined as a
  spectator because round 1 started eight seconds after the map loaded: a hosted game now waits in town
  until the host presses Enter. Spectators now see the watched player's up/down look and sprint pose
  (the engine only applies those on the owner's machine) and stay on a dying player's ragdoll.
- New ways to play: lean (Q/E, so Use moved to G), a melee strike (V, and the trigger on an empty gun),
  a dead raider's best gun left on the ground beside his gold, and a window vault. A loot menu for
  bodies was built and then dropped the same hour for the simpler ground pickup.
- Measuring before fixing paid off twice. 42 of 62 doors and 112 of 113 windows in the town were a few
  centimetres too tight for the player's capsule, and porch decks sat 15 cm above the room floor right
  under the door header; town generation now fails on any opening that is too tight. The first tornado
  came at 190 s, later than most real rounds last; small tornadoes now cross from 30 s.
- Ladders are walked like stairs (the old climb turned the player 180 degrees on every grab), anyone on
  the platform calls the train in 30 s with a distant whistle, the HUD shows rounds in the gun and in the
  pocket, the town has three gun stores and visible cartridge boxes, and there is a volume setting.
- New tool: Suno (suno.ai). The human made the menu, intro and outro track "Whistle in the Dust" with
  it; it is being wired in now.
- Friends can join by a short code over Epic Online Services (device-id sign-in, P2P): proven on this
  machine against the real backend; still needs a two-network playtest and a Windows package check.

### 2026-11-14 05:16 EST — Overnight: recorded sound, a Windows package, and bugs only a second machine shows
- The human dropped six Freesound recordings and a Suno track into the repo and went to bed asking for
  polish. Each recording was cut by measurement, since no agent can hear: the shot found among two pump
  clicks by its energy, footsteps split at their onsets, the train split by spectrogram into whistle
  blasts, an 18-beat engine loop and a bell. One gunshot is pitched four ways for four guns. A master
  compressor and limiter keep footsteps audible under gunfire. New third-party assets: recordings by
  Shades, Nox_Sound, C-V, KVV_Audio, Mark_MPT and theshaggyfreak (freesound.org; licences in ASSETS.md),
  now credited on an in-game Credits screen because three of the licences require attribution.
- Verifying last night's features under simulated lag found that the new window vault had never worked
  for a joining player: the jump never reached the server. It now travels with the predicted move.
  "Bots get stuck" turned out to be bots freezing over bodies, holding the Use key the new gun pickup
  needed released; humans had the same problem, so Use now takes what you look at.
- First Windows Shipping package since the early zip (390 MB): it failed to compile until built as
  Shipping, and three materials drew grey in a package. It runs in the Windows VM without a GPU.
- Finger joints bent backwards on every gun's grip because a bend axis flipped past 90 degrees; fixed,
  with an anatomical-limit check on every joint.
- Problem: several rendered runs at once hung the graphics driver at 03:40, so nothing since has been
  seen on screen. The owed looks are listed for the morning.

### 2026-11-14 09:33 EST — A morning playtest, turned into a dozen fixes
- The human played a gold run and reported as he went: unreadable intro text, synth drums in the safe
  crack, his own footsteps trailing behind him, a death to the tornado inside the vault, and a town
  wiped in a minute. Each report was traced in code before anything changed.
- Landed so far: the intro slate and title card now play in order and the host's start key no longer
  skips them; the safe crack is the recorded ratchet alone (the "realistic" sound had a synthesized
  ring and clunk baked in, louder than the recording); own footsteps play centred instead of from
  where the foot landed.
- Decided and in progress: the storm becomes straight fly-bys that scrape the outskirts and close in
  on the train station, with the big funnel kept and a smaller damaging core that hurts fast instead
  of killing outright; the vault is safe and a wrecked building gives a few seconds to run. Looting
  is being reworked so a dead player leaves a gun and a sack with everything else, boxes never swap
  your gun, and ammo comes in three types with .44 common and .45-70 and shells scarce.
- Found by measuring: the storm removed 84% of building pieces and none of 1,662 props, which is why
  rows of crates and fences were left standing in a flattened town.
- The looting rules landed within the hour: a dead raider's gun keeps only the rounds in it and his
  sack carries his gold and every other round, one hold takes both; boxes never swap your gun, the
  guns they used to hold now lie beside them to take on purpose; every life starts with 60 gold and
  full guns, so killing is a way to earn. Review found that 8 of 21 of those free guns were never
  placed, because the search for a spot started on top of the shop counter instead of at the floor.

### 2026-11-14 10:42 EST — The Gold Rush, plain numbers, and fewer switches
- The human kept playing and reporting. On main since the last entry: the mode is now called The Gold
  Rush; the pistol is carried raised and muzzle-up in a sprint; the safe's clicking falls off with
  distance and is gone by 70 m (it had been as loud at 50 m as beside it); gun hints sit in a fixed
  slot over the resting gun; and ammo is a plain "6 + 12" instead of bullet icons.
- Two rules from the human changed how we work: a fix gets no switch back to the broken behaviour,
  and related tasks are queued on the worker already in that code so they share one test gate.
- Shelter from the storm landed: the bank vault is safe from tornadoes, a building's first storm hit
  gives the people inside four seconds to get out, the core hurts fast instead of killing outright,
  and crate walls, fences and hay now go with the buildings. Review found the vault's safe box
  reached a metre outside its walls, and that the network tests ran on a map with no vault at all.
- Controls: Use moved to F and cocking to the mouse wheel, lean is always on, and a cocked gun now
  ignores the action key, so spinning the wheel cannot waste a round. Keys are rebindable in Settings
  and saved, with a reset. Review found the wheel still cocked whatever it was bound to.
- The human recorded the revolver's hammer and loading himself, the game's first first-party sounds;
  an inventory found 42 of 60 sounds still synthesized, and they are going onto the review page to
  be judged by ear one at a time.
- In progress: the fly-by storm, three ammo types with sacks that drain at a fixed rate
  for several looters at once.
- Boulders: the playtest's "geometric crystals" were chamfered boxes drawn with the metal material.
  Five rounded Meshy desert rocks replace them, varied from one to the next, with the same cover
  footprints (120 credits). Four of the five generated meshes came partly inside-out and were repaired.

### 2026-11-14 11:32 EST — Three cartridges, and robbing a body takes time
- The human asked for ammo types: .44 for the revolver and carbine, .45-70 for the rifle, shells for
  the coach gun. Each raider carries spares per type; .44 is plentiful and cheap, the other two are
  scarce, dear and limited per store per round, so the strong guns are good for a fight or two. The
  carbine became the common gun (cheaper, and most of the free guns lying out).
- Sacks now drain instead of being taken in one go: 200 gold and 8 rounds a second to everyone
  holding Use on it, so 1,000 gold takes five seconds and can be contested. Let go and the rest
  stays. Two lagged clients on one 1,000-gold sack ended with 502 and 498.
- Bots fire too little to show whether a rifle can be kept fed for a round; that one needs a human.
- The storm no longer flattens the town in a minute. A full-size funnel now flies past on lines
  tangent to a shrinking circle round the station, scraping the outskirts pass by pass: about 8% of
  the town gone in the first minute (it was 26%) and two thirds by the train, with the station
  untouched in 50 of 50 test rounds. Only a small core under the big funnel hurts. The human's call:
  the storm exists to shrink the play area and force fights, not to kill.

### 2026-11-14 13:30 EST — Hits you can see, and a revolver you can fan
- With the hit-confirm sound removed at the human's request ("no magic audio feedback"), a hit has
  to show at the victim: a cloud of dark red mist now bursts from the body, bigger for head shots,
  one merged cloud for a coach-gun load, and sized by each viewer's distance so it reads across the
  street. Before, frames from the shooter's position showed nothing at any range.
- Single-action shooting, the human's design over several messages: aiming cocks the gun as it
  rises, an aimed shot drops the gun so each shot is fire, aim, fire, aim, and holding fire while
  spamming the hammer fans the revolver from the hip.
- The human recorded sounds himself (revolver hammer and loading, lever, a body being hit, spent
  cases) and picked spaghetti-western ricochets; the first batch is in the game. Every bullet now
  ricochets, the victim's body makes the hit sound, cases clink on the ground, and a raider taken
  by the tornado gets a Wilhelm-style scream. Review found nobody could hear the body impact at
  all, and that two of the new sounds had been cut but never played.
- The .45-70 now kicks like a cannon: the muzzle snaps up about 42 degrees and settles slowly.
- Bots had quietly stopped firing from the sights under the new aim rule (kills per round fell
  from 5.3 to 3.7); a bot batch before and after caught it.
- The round-start camera was redone to the human's sketch: the bank from the street, a flight
  locked on the bank to ten metres above your own spawn, then down into your eyes, and the card now
  says what the safe holds. The wall close-up he reported was a shot aimed at the other game mode's
  spawn points, followed by a 90 m slide through walls.

### 2026-11-14 16:13 EST — Jumps with weight, silly names, and a tornado that drowns everything out
- Jumping was floaty and lost its momentum the moment the key was released (two lines of code were
  braking the player in the air). Jumps now carry their speed, steer almost not at all, and fall
  faster; drops from roofs hurt (5 m costs 30 health).
- Every machine gets a stable silly western name (Gumshoe Dunmore, Kerosene Bramblewood), renamable
  from Settings even mid-match, and a session scoreboard totals gold extracted across rounds and
  survives a rejoin. Review found the engine's login was sending each player's computer name to
  the server; fixed before it landed.
- Second audio pass from the human's garage recordings: coins pouring while a sack drains, dry
  fire, menu ticks, a boards recording layered like particles while buildings come apart, and a
  tornado whose bass, saturation and level rise as it closes until the limiter muffles everything
  else. A slow limiter release makes a gunshot briefly deafen the mix. Only three sounds in the
  game are still synthesized, down from 42 this morning.

### 2026-11-14 17:22 EST — A five-second change must not take an hour
- The human's complaint, and a fair one: reverting one sound took 35 minutes to reach a build he
  could play. Small changes had been going through three agent hand-offs, one serial landing queue,
  the full test gate, and then waited for him to close his game.
- New workflow for the last day: small changes land by a fast lane (one worker, under two minutes
  of tests, straight to main, reviewed afterwards); he plays from a copy that a launch script
  refreshes from the last finished build, so nothing waits for him to quit; a background builder
  keeps main built. The first copy would not start: the game module finds the engine by a relative
  path, so a copy has to sit exactly as deep in the filesystem as the repo.
- Correction to this morning's entry: the storm does kill. Falling hurts in a storm too, and a
  tornado's fling is credited to the tornado. The human's point was only that bot simulations need
  not show storm deaths, since bots run away.
- By ear, the human sorted the sounds: synthesized drums, clicks and thuds are bad; synthesized
  wind is good. The library wind an agent had downloaded came out again ("a kid making a whoosh
  into a pvc pipe"), and no agent downloads a sound file without his approval from now on. The keg
  blast is now the guns' own gunshot slowed down; his recordings were re-cut where the cutter had
  kept one click of a double click. Nothing percussive in the game is synthesized any more.
- First landings through the fast lane, minutes each: the round intro now says it once, on the
  clapper ("Gold Rush - Get the gold from the bank, catch the last train out of town", held five
  seconds), instead of a clapper, a title card and screen text all saying the same thing. And the
  bank safe clicks again while it is cracked: the afternoon's sound cleanup had deleted it by
  misreading "silence the safe opening", and the review had passed it; the human heard it missing.

### 2026-11-14 18:26 EST — Short sessions, one landing queue, a dozen playtest fixes in an hour
- The human found the long-running worker sessions cost more and did worse, so the rule changed: one
  task, one fresh short session, closed when its commit is on main. To keep many of them from racing,
  every worker now lands through one queue script (one landing at a time: rebase, build, the tests that
  touch the change, push); a background builder turns each landing into a playable build about a
  minute later.
- Through that queue, from the human's playtest notes: alt-tabbing back no longer needs a click that
  wastes a shot (the engine's Linux layer only woke the game on a click); aiming an uncocked gun now
  costs about 0.4 s more than aiming a cocked one, so cocking ahead pays; loot only what you look at
  with a clear line of sight; kill feed with gun silhouettes drawn from the game's own models and the
  outlaw's hat for a headshot; left and right mouse cycle the spectated player; 38 short western names
  instead of long generated ones; an fps readout with a settings toggle; looting plays the pocket and
  the coin jingle together.
- The review page was brought up to date: 101 stale clips re-rendered from main, 9 retired, and a card
  for every sound the game builds from other sounds, with its recipe and a play button per ingredient,
  so the human can say whether the recipe or the source clip is wrong.
- Problem hit: the clip renderer had been running two engines at once and swapped frames between two
  clips; it renders one at a time now.

### 2026-11-14 21:01 EST — Slower crouch, rematch that keeps friends, and a beat before Cut
- From the human's play: crouch felt snappy, rematch and the film's map change kicked the join-by-code
  client, and a round-winning shot cut to the outro before the body fell.
- Crouch squat and the first-person eye now ease over about a quarter second (capsule still snaps for
  hits). Rematch and scene travel use seamless ServerTravel so the EOS connection stays up. After the
  deciding kill the round stays Live for 1.8 s so you see them go down before "Cut."

### 2026-11-14 22:34 EST — Lying down, a gentler revolver, and a build that stopped following main
- The human asked for a prone move, a crouch hold/toggle setting and a rebalance: revolver 40 body / 90
  head (no more one-shot head kills), melee 30 a swing, the .45-70 rifle scarce and 500 gold.
- Prone is the engine's crouch at a third, lower height rather than a new movement mode, so it replays
  and replicates the way crouch does; the window vault taught us what happens otherwise. Z toggles it;
  the gun drops for half a second on every change so nobody drop-shots. The body is posed in code and
  still looks like a plank: a crawl and a proper lie-down are in progress.
- His hitbox preview frames showed the body pill standing upright while the torso bent forward to aim
  down. Capsules along the spine, legs and arms are being built, placed by calculation like the head,
  because a server without a screen never animates anyone.
- Problem hit: to match a Windows package for a two-player test, the main checkout had been parked on
  an old commit, and the play build silently fell 16 commits behind for hours. The builder now puts a
  clean checkout back on main and says so loudly; a matched Windows + Linux pair of zips from one
  commit is being built to replace the trick. Unlanded playtest edits found there (one second on your
  own body, then your killer's view in Gold Rush) were reviewed and landed.

### 2026-11-15 01:04 EST — Rancher colours and bun seam
- Human review of the gold-rush Rancher: bright clown colours on walk, and a white gap around the
  hair bun on the crouch turntable.
- Dirtied the diffuse toward dusty ranch colours (shirt, hat, pants, hair, skin) and painted pale
  nape texels into the ginger bun so it no longer floats on a bald ring; FP sleeve/hand look colours
  follow. All five `char.rancher.*` clips re-rendered for review.

### 2026-11-15 02:56 EST — EOS Connect auth refresh
- Human play session: join-by-code P2P died after exactly one hour with `EOS_InvalidAuth`.
- Cause: Connect tokens expire hourly and we never registered AuthExpiration / refreshed login.
- Register the notify and re-login with the same device id so a long session keeps the connection.

### 2026-11-15 04:37 EST — Wall-banging
- Human ask: shoot through buildings and props, with damage loss and a limit on wall thickness.
- Each gun now has a budget of wood it can punch through (rifle 55 cm, carbine 32, revolver 24, shotgun
  pellets 8). Every layer measures its own thickness and costs damage (a kit plank wall is 18 cm: the rifle
  keeps ~63 % of its damage, the revolver ~47 %); stone, rocks, iron (safe, vault, stove) and the ground stop rounds.
- The entry hole gets a wood impact, and a remote player's hit claim through a wall is checked and scaled
  by the server's own walk through the same wood.

### 2026-11-15 06:46 EST — Last-morning playtest pass on the gold rush
- Playtest feedback turned into a run of small fixes: a 3-minute train with a live countdown on the ticket, a 15 s ending
  with a scoreboard that updates the moment a rider extracts, an alive count that includes those who got out, and the
  train called automatically when one raider is left.
- Placeholder cube costume bits (neckerchief, bandana, badge) removed; uniforms alone tell the sides apart. Beds got cloth.
- Combat feel: kill-feed icon for wall-bang kills, shotgun accuracy no longer punished for running or jumping,
  gunshots audible across the map, powder-keg kills now fling bodies (the launch was lost when the body went ragdoll).
- The square of odd floor around town turned out to be drawn dunes with no collision over a flat invisible floor; the
  desert is now one flat surface out to 500 m with storm damage beyond 200 m.
- Spectating online: the watched player's look arrived in packet-sized steps; it is now smoothed for the spectator view.
- Decision (human): the gold rush becomes a 7-round match of plain gold-maxxing, scoreboard between rounds, awards at the end.

### 2026-11-15 09:21 EST — Match flow, awards and a new front door
- The 7-round match got its finish: the end-of-match awards now visit the raider who rode out (standing in the boxcar
  doorway), and two new superlatives join The Good, The Bad and The Ugly: Dead Eye (most headshot kills) and Boot Hill
  (most deaths). The scoreboard gained a deaths column.
- Hosted matches no longer restart on their own: after the awards the host gets "Play again?", players stay connected
  and newcomers can drop in during the wait.
- The main menu was rebuilt around mode tabs, with the hosting choice (address or Epic lobby code) on top and a single
  join box that recognises either.
- Playtest fixes: spectators see the watched player's health, fall damage starts at twice the height, sloped roofs no
  longer turn into oversized stepped blocks under storm damage, the start sign is a clean two-liner, the fake station
  clock is gone.
- Problem: the disk filled up mid-morning (about 230 old agent worktrees with builds, 26 GB of packages in one); every
  build failed until they were cleared, and agents now remove their worktree as soon as their work lands.

### 2026-11-15 11:07 EST — Last-day polish: modes, menus, props and a proper slate
- The 4v4 shootout moved onto the gold rush town with the bank vault as its objective; the original maps live on in a
  "Deleted Scenes" menu section. Sides are random, players can switch teams between rounds, and guns are dealt and
  rotated each round so nobody camps the .45-70. The 4v4 got the gold rush's award visits, dead winners included.
- The scoreboard became a real film slate (mirrored striped sticks, hinge, ruled cells, chalk lettering), and every
  translucent box behind HUD text was removed in favour of outlined, shadowed type (human's rule).
- Placeholder cubes for loot, counter goods and the gold sack were replaced by Meshy models (concept image first,
  then smart-topology mesh); unopened loot now glows like a candle and goes dark once looted. The bank safe's door
  actually swings open. New perks: Sneaky boots (silent crouching) and Cowboy coffee (+15% speed).
- Fonts: Gochi Hand and Barlow Condensed (Google Fonts, OFL) for the slate.
- Lessons: a placeholder in code is not always on screen (all characters already had modelled hats, and we paid for
  replacement hats nobody sees), and a game launched mid-build kept getting an hour-old copy until the build loop
  started preparing the play copy itself.

### 2026-11-15 17:00 EST — Heist, a quieter frame and a menu that makes sense
- Playtests showed nobody ever played the 4v4 objective: every round ended in a plain shootout. The human redesigned
  it as a new mode, **Heist**: the outlaws start inside the bank and crack the safe, grab the whole gold bag in one
  go, and must get it onto a train that waits two minutes; the newly named Marshals ride in on that train and win by
  killing every outlaw or running out the clock. Wiping out the Marshals no longer ends a round, so the gold has to
  move. The old Shootout lives on in Deleted Scenes, now its own top-level menu entry.
- The menu was folded into one Start Game screen (Offline, Lobby Code or Raw IP Address, with the same round settings
  everywhere), and every saloon in the gold rush town now serves healing whisky.
- A frame-time pass found the tornado's cost was not the number of flying boards but the lighting recomputing
  every time one moved; taking debris and small moving props out of that lighting cut the twister's worst frames
  from 23 ms to about 9 ms with no visible change.
- The day's red network checks turned out to be six stale tests (two-word names, the new 7-round flow, the raised
  fall height) and one real bug (perk drops landing on the gold sack); the build loop also learned to restart itself
  after it had silently died on an edit to its own script.
- First Windows packages came out of the build VM after two compile errors that only MSVC catches; the strict Linux
  build now catches shadowed names as well.

### 2026-11-15 19:15 EST — Final evening: Heist polish from live playtests
- Evening playtests with bots and friends drove a run of small fixes to Heist: the safe now works exactly like the
  gold rush's (same sound and prompt, look-at to crack, gold to the cracker at once), the gold beacon shows only to
  outlaws and only while the bag lies on the ground, killing the carrier aboard cancels the train's departure, and the
  train waits on the gold rush's own ticket with live team counts.
- Bots stopped jamming doorways: one explicit cracker, the other outlaws fan out to a perimeter in nearby buildings,
  marshals scatter from the train, and carriers walk deep into the boxcar. Dead players can take over a living
  teammate bot with the Use key.
- Bugs found in play and fixed: a rematch that grew teams to 7 v 7 (bots carried over by seamless travel), a client
  crash on rematch (a HUD read during travel), first-person arms floating over cutscenes, floating guns during the
  train's arrival, crouched players stuck at ledges, and stepped roof collision.
- The clapperboards became plain chalk on a dark slate, and the end-of-match board sizes to its content.
