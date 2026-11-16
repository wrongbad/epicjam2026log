# Epic AI MegaJam 2026: dev log and author notes

Notes from our Unreal Engine entry to the [2026 Epic AI MegaJam](https://itch.io/jam/2026-epic-ai-megajam).

- [dev_log.md](dev_log.md): detailed timeline of the build, changes, fixes, pivots, etc.
- [repo_stats.md](repo_stats.md): top-level break-down of code and assets

## Repo Stats

- The private git repo has about 100k lines of engine/runtime code, and 75k lines of tests and tools.
- There are also 1,173 binary asset files totalling 817 MB
- All of it added to git and git-lfs in 1,381 commits over 4 days
- See [repo_stats.md](repo_stats.md) for a more detailed breakdown

## Before and During the Jam

Hi, I'm Kyle Finn, software engineer unrelated to games, and I really enjoyed playing intense multi-player shooter games like UT, CSGO, PUBG, etc.

I've thought for a long time it would be fun to make my own shooter game, but never made the time for it.

Seeing the games people have been vibe-coding lately, and this contest promoting vibe coding a game, it felt like the right time and time-box to just go for it.

In the spirit of the contest I'm trying to go as AI-native as possible. That means commanding an agent team, not looking at any code, and not touching any 3d or audio editor tools myself. I also wanted to see how far a modest budget could be pushed, so I bought \$100 month of claude subscription, and \$25 on OpenRouter to use on DeepSeekv4.1 Flash, as my only out-of-pocket expenses. Of course I claimed the contest sponsor credits as well (but missed the window for claude credits).

My agent team was structured with Fable at the top as my chat session, and Opus and DeepSeek workers spawned at its discretion, with Fable intervening when necessary if workers got off track or stuck. Fable was told to delegate everything to save credits and keep the main sessions tidy for longer horizon conversations. Still I found it filling up after 8-12 hours and I would start fresh sessions.

My workflow was mostly giving Fable 5-10 things to do at a time, and going back to my day job or sleeping for a few hours, and checking back. The worker agents liked to run lots of tests and simulations to validate things, so most feature requests took 1-2 hours, but they could be parallelized.

I had to intervene on audio fx / foley - I couldn't get ai code-synthesized or ai generated audio to come close to what I wanted. I found some sounds on Freesound, and recorded some others in my garage. And then I told the AI how I wamted it chopped and edited, which seemed to work quite well, so at no point did I open audio editing software.

Amusingly, like a throwback to early stable diffusion models, the character hands were one of the hardest aspects of the project. Somewhere between meshy and rigging, and aligning hands to gun poses... hands got mangled and deformed into a hot mess. I had agents build fixtures and processes to use image feedback before passing along to me to QA review, and then many iterations of me telling it which joints were bent the wrong way or curly-fried or whatever.

We ended up building a QA / ticketing app to keep my time efficient on reviewing all the meshes, effects, game mechanics, audio, etc everything one at a time on a big dashboard, where I could flag what's wrong with what quickly, and see which things agents were working on vs backlogged. It was easily a 10x productivity boost compared to end to end play testing.

I found myself often hit claude limits working this way, used my resets, started running cursor-grok sessions to use the free credits from the contest while claude was out, but it wasn't using my agent harness and didn't cooperate with the merge queue my main agent built, leading to som tasks reverting other tasks, and needing extra cleanup. So when my claud limits reset, I switched to Opus-Sonnet-DeepSeek to be more cost efficient. I don't notice much difference.

I did some multi-player testing with some old gaming friends from csgo/pubg days, and we were totally entranced by the fact that everything we didn't like about the game can be fixed within minutes, rather than month/years you get from a big production game. The mechanics and game loop arrived at a deeply satisfying place for FPS fans. We played until 6am EST the night before the deadline, and the game had new features, mechanics, and fixes every hour.

## Reflections (Nov 16)

I'm proud of the game after just 4 days of work. It's genuinely very fun to play online with competitive FPS players, with a game mechanic feature set I think is actually competitive with many of the most popular FPS launches from the past 10 years.

The development process going "full vibe" mode basically meant complex features land MVP versions within 10-30 minutes: prone, leaning, new game-modes, bot behaviors, etc. But this process still creates new bugs in the same ballpark as average humans building features. Agents can fix the bugs faster than humans, but you need to find them. Which meant for landing many features in a short time, the process of spotting and precisely describing defects needed to be a well oiled machine. Concretely that meant the QA app described above for shortcutting play testing, and describing defects very precisely with all the relevant technical jargon, so the agents could fix them quickly and reliably.

I also sent agents on missions to identify duplicated code, god objects, other anti-patterns, and perform refactoring passes. And I sent agents to work on running perf analysis on both cpu and gpu, which drove down p99 frame times by an order of magnitude. I also had agents optimizing the test and build time, pruning or using background workers for less-essential tests, building local merge-queue tooling to reduce testing overhead with 5-10 agents working concurrently on my PC. The funny part to me is that these were all trivial to ask for, agents did all the work without trouble, and the whole trick is remembering to ask for them.

Driving AI as a non-artist produced some plausible placeholder assets and maps that got the game ready for immersive play-testing almost instantly. It built ideas super quickly, enabling more things to be tried, and bad ideas to be pruned fast and cheaply. But I felt that I didn't really know what to do to make them look better. I don't really know the workflow and jargon of asset designers so it was hard for me to drive the AI to a higher level of quality than its baseline "one shot" quality. Now I know what I need to learn or team up with to fill the gap.

## Future Work

Taking the game to the next level would require more investment in visuals. Probably teaming up with someone talented at graphics would be a huge unlock. 

On another dimension, I think some larger battle royale maps with horses for travel and gunfighting would be amazing. There's a hundred other ideas one could take a shooting game, and having the power to build them quickly is amazing.

And third, a little investment in auto-scaling dedicated servers and player stats/elo databases would bring multiplayer experience to today's high standards.

And fourth, the game already supports peer-to-peer hosting, and I’d keep that first-class alongside public matchmaking rather than going all in on it. Unreal Tournament showed the world how cool it is to let the community decide how to play, by hosting games with their own rules and assets, and Counter-Strike’s “10 man” culture showed how well friends self-organize private matches. As people increasingly throw exploitation and automation at public leaderboards, many of us may turn to private and community matches to go back to the roots of just having fun in online gaming.
