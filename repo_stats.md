# Repo stats

Snapshot of `epicjam2026` at commit `f90af30` (Nov 15, 2026), the end of the jam.

The repo has about **175,000 lines of code**, not counting blank lines. Roughly 57% is game code, 18% is tests for the game, and 26% is tooling. There are also **1,173 binary asset files totalling 817 MB**. All of it came from **1,381 commits** over 4 days, around 345 a day.

## Code

Non-blank lines, comments included.

| Area | Files | Lines |
|---|---:|---:|
| Game: C++ runtime | 225 | 99,255 |
| Game: C++ tests | 95 | 30,759 |
| Tooling: Python/shell | 171 | 40,613 |
| Tooling: tests | 33 | 4,885 |
| Config (.ini, build rules) | 9 | ~540 |

About 3,000 of the runtime lines are one generated table of building boards (`WesternBoardTables.inl`). Without it, hand-directed game code is about 96K lines, and the C++ tests come to about a third of that.

### Game runtime by subsystem

| Subsystem | Lines |
|---|---:|
| Core | 29,525 |
| GoldRun mode | 15,615 |
| Weapons | 10,007 |
| Destruction | 7,909 |
| UI | 7,517 |
| Bots | 6,170 |
| Map | 4,563 |
| Audio | 3,487 |
| Twister (the tornado) | 3,015 |
| Solo | 2,671 |
| VFX | 2,315 |
| Match | 2,132 |
| Look | 1,834 |
| Telemetry | 1,074 |
| Net | 1,000 |

### Tooling by folder

| Folder | Lines |
|---|---:|
| Props | 6,858 |
| Clips review-page system | 6,604 |
| Procedural building generator | 6,237 |
| Top-level build/land/play/test scripts | 6,228 |
| Audio synthesis | 3,599 |
| Layout generator | 3,283 |
| Town | 2,089 |
| Train | 2,047 |
| Analyze | 1,620 |
| Agent skills | 1,117 |
| GoldRun | 1,076 |
| Smaller folders (look, art, guns, VFX, loot, maps, …) | ~4,700 |

## Not counted as code

- **Generated level data: ~903K lines of JSON** in `Content/Layouts`. These are the town layouts and the collision boxes made from them. Counting them would swamp everything else, so they're left out.
- **Tool data and fixtures:** ~14K lines (prop catalogue, building archetypes, test fixtures).
- **Docs and process:** ~6.1K lines of markdown. The work-in-progress files are 2.3K, the specs 2.0K, and the README, dev log, ideas and AGENTS.md 1.7K.

## Assets

| Type | Files | Size |
|---|---:|---:|
| .uasset (imported engine assets) | 831 | 425 MB |
| .glb (source meshes) | 185 | 188 MB |
| .png / .jpg | 67 | 120 MB |
| .wav / .mp3 | 80 | 83 MB |
| Fonts | 6 | 1 MB |
| .umap | 4 | 0.1 MB |

The biggest asset groups are game content at 280 MB, characters at 125 MB, Meshy-generated loot at 124 MB and props at 95 MB. Two things stand out:

- **The maps are almost all code.** The four `.umap` files together are about 100 KB, because the town is built from the JSON layouts by the generator.
- **The buildings are code too.** The 97 building mesh files total only 3 MB because a Python script generates them.

Per `ASSETS.md`, the art comes from Meshy (guns, props, train, loot, boulders) and Hyper3D Rodin. Textures are from ambientCG, music from Suno, and sound from Freesound plus our own recordings. Some sounds are synthesized by the game's own audio scripts, and a little is Epic template content.

## Method

Lines were counted with a small script over `git ls-files`, so comments are included in the totals. Asset sizes come from the Git LFS records, so they're the real file sizes.
