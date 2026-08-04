# Appendix: the \navworld{} world

Everything in this appendix is computed from the 225 scored conversations of the realistic channel,
45 per system. Configuration is fixed throughout: user simulator `gpt-5.6-luna` at temperature 0,
decision model `claude-haiku-4.5`, and the five voice agents as the only variable. The corpus is
56.2 simulated hours, median 168 turns and 1053 simulated seconds per conversation.

---

## A.1 What the world asks

A voice copilot guides a pedestrian who cannot see a map. The copilot can query the map; the walker
cannot. The walker can see the street; the copilot cannot. Neither alone can complete the task, so
every block walked is the product of a spoken instruction and its execution, and every failure is
attributable to one of the two.

This is the property that makes it a voice benchmark rather than a routing benchmark. A text agent
with map access solves these tasks trivially. The difficulty is entirely in maintaining a shared
model of an unseen physical situation across a lossy channel.

Three conversation types, nine scenarios, three instances each.

| type | what it adds | scenarios | optimal blocks |
|---|---|---|---|
| **base** | nothing; a single destination on a clear map | `geo_a_base_i1..i3` | 5 |
| **rerouting** | four pavement closures, none announced | `geo_b_blocked_i1..i3` | 7 given the closures, 5 if clear |
| **long-horizon** | a second destination revealed only on arrival at the first | `geo_f_errand_i1..i3` | 11 across two legs |

---

## A.2 The map

### A.2.1 Why GeoJSON, and what that buys

The world is a single GeoJSON `FeatureCollection`, `map.geojson`, which is the same interchange
format a real mapping stack consumes. This is not decoration. It means the world is inspectable
with ordinary GIS tooling, that a segment closure is a property edit rather than a code change, and
that the routing the copilot queries runs over the same graph the walker physically traverses. A
divergence between "what the map says" and "what the street does" can only be authored deliberately,
never introduced by two implementations drifting apart.

| feature kind | count | what it is |
|---|---|---|
| `street_segment` | 105 | one block of pavement, with `from`, `to`, `length_m`, `closed` |
| `building` | 240 | block-filling footprints |
| `place` | 31 | named destinations, 9 flagged as landmarks |
| `entrance` | 5 | named doors on the two superblocks |
| `no_crossing` | 4 | corners where one side has no pedestrian crossing |
| `superblock` | 2 | Grand Central Terminal, United Nations |

**Figure A.1** `fig_anatomy` shows all of it drawn on the map itself.

### A.2.2 Grounding in Midtown East

The window is 5th Avenue to United Nations Plaza, E 42nd Street to E 49th Street: eight avenues by
eight streets, 64 corners. The named geography is real. Grand Central Terminal occupies the Park to
Lexington block between E 42nd and E 45th and is impassable, which is why several optimal routes are
forced into an L rather than a diagonal staircase. The United Nations occupies the eastern edge.
The nine landmarks (Chrysler Building, MetLife Building, Waldorf Astoria, Summit One Vanderbilt,
New York Public Library, Helmsley Building, Daily News Building, Trump World Tower, One Dag
Hammarskjold Plaza) are at their true corners and serve as orientation anchors the walker can
plausibly see.

Ordinary destinations (a pharmacy, a deli, a clinic) are authored rather than real, so that success
depends on the copilot reading the map rather than on the agent's memory of Manhattan. Anything a
model could know from pretraining is a landmark and never a goal.

### A.2.3 The geometry, and where the scoring space differs from the walking space

Manhattan blocks are not square, and the asymmetry is large. Walking **east to west along a
street**, avenue to avenue, is **274 m**. Walking **north to south along an avenue**, street to
street, is **81 m**. Every one of the 105 segments carries one of exactly those two values: 274 m on
all 49 avenue-to-avenue hops, 81 m on all 56 street-to-street hops. The window is 1,918 m wide and
567 m tall, a true aspect ratio of **3.38 to 1**.

Scoring counts **blocks**, not metres. The scoring space is therefore the square 8 x 8 lattice while
the walking space is the elongated true grid, and the scorer calls a 4-block crosstown leg
(1,096 m) and a 4-block uptown leg (324 m) the same length when on the ground one is 3.4 times the
other. A route that is optimal in blocks need not be shortest in metres.

We report blocks anyway, because the block is the instruction unit: a copilot says "walk two blocks
and turn left", never "walk 548 metres". The walker counts corners, not distance, so blocks are the
quantity both parties actually reason about. The metric view matters for interpreting effort, not
for scoring.

**Figure A.2** `fig_scale` puts the two views side by side. Route figures elsewhere in this appendix
use the lattice view, since that is the space the metrics live in.

---

## A.3 The interface

Fourteen tools, and the split that matters is **who may call them**. It is enforced by class:
`environment.py` wires `WalkCopilotTools` to the agent and `WalkUserTools` to the walker.

**The walker holds 9.** Movement: `turn_left`, `turn_right`, `turn_around`, `walk_to_next_corner`,
`move_forward`, `cross_the_road`. Perception: `read_signs`, `look_around`. Task control:
`ask_for_next_place`.

**The copilot holds 5.** `get_gps`, `find_place(query)`,
`directions(from_avenue, from_street, destination)`, `live_status()`, and
`note_user_location(near_cross_streets, facing)`.

**Figure A.3** `fig_tools` draws the surface, colour-coded by actor.

That asymmetry is the world. **`get_gps` is the only sense the copilot operates itself.** Everything
the walker can perceive, the copilot has to ask for and receive as speech. It cannot read a street
sign or look around; it can only request that the walker do so and then parse the answer off a lossy
channel.

Two consequences generate most of the difficulty, and both are deliberate.

**`get_gps` never returns heading.** It gives a position, never a facing. A copilot that knows the
walker is at Park and E 45th still does not know whether "turn left" sends them north or south.
Heading can only be recovered by asking, by inferring it from a `read_signs` or `look_around`
answer the walker volunteers, or by deducing it from the last executed movement. Section A.8 shows
this is the single largest behavioural difference between systems, and it is a direct consequence of
the one sense the copilot owns being the one that omits orientation.

**`live_status()` is the only advance warning of a closure.** A copilot that never calls it
discovers closures the way the walker does: by being sent into one. That converts the rerouting
condition from a planning problem into a recovery problem.

Two further structural facts. Seven of the 112 possible lattice edges do not exist, swallowed by the
two superblocks (two rungs through Grand Central, five along the United Nations frontage), which is
what makes them impassable rather than merely slow. And the four `no_crossing` markers name an
**arm** of a junction rather than a corner: the missing crosswalk sits on one approach, so a walker
can stand at the corner and still be unable to cross in one particular direction.

---

## A.4 The three cases

### A.4.1 base

Single destination, clear map, optimal 5 blocks. **Figure A.4** `fig_cond_base`. It tests the
minimum: establish where the walker is and which way they face, then issue five correct instructions
in sequence.

What the Grand Central superblock forces is a **crossing, not a shape**. There are four distinct
5-block routes, and every one of them uses E 42nd Street between Park and Lexington, because the
terminal admits no crossing further north. The figure draws the fewest-turns member of that family
(down one, east three, north one) and says so. Efficiency is therefore scored against a family of
four tied optima, not against a single reference polyline.

### A.4.2 rerouting

Four closed segments, invisible in advance. Optimal is 7 blocks given the closures against 5 if
nothing were shut, so a perfect run still pays a 2-block detour. **Figure A.5** `fig_cond_roadblock`
draws the clear-map route, the four barricades and the surviving detour together.

The condition is a belief-repair test. The copilot's plan is formed from a map that is correct about
geometry and wrong about passability. The repair sequence has four beats, and we quote a real one in
Section A.9: a stale instruction into a closed segment, the walker contradicting it, a
`live_status()` call that names the closure, and a corrected instruction.

**The four closures do not contribute equally, and only two of them cost blocks.** Dropping each in
turn: the Lexington closure between E 45th and E 44th sits on the clear route but alone costs
nothing, since the walker simply rounds the south of the terminal and still walks 5. The E 42nd
closure between Park and Lexington shuts that alternative too, and only with **both** does the walk
cost 7. The remaining two change no block count at all; what they do is cut the number of tied
7-block routes from four to **one**, so the optimum becomes unique and any deviation is penalised.
The difficulty is thus authored in two layers: two closures set the price, two more remove the
margin for error.

Which closures a given run actually meets depends on its path, so "four closures" describes the
world, not the difficulty of any single conversation.

### A.4.3 long-horizon

Two legs, 11 blocks total. The second destination does not exist in the task prompt the walker
receives at the start: it is revealed on arrival at the first. **Figure A.6** `fig_cond_errand`.

This tests whether a copilot that has just declared success can re-plan from a new position rather
than from its stale starting model. Leg 1 is 5 blocks and leg 2 is 6.

Neither leg has a unique optimum: leg 1 admits 3 tied 5-block routes and leg 2 admits **20** tied
6-block routes. The figure draws one member of each family and labels it "an optimal route", and
efficiency on this condition is scored against the block count rather than against a reference
polyline.

---

## A.5 Scoring

| metric | definition |
|---|---|
| **PRD** | the walker reached the destination, by replaying the tool stream through the world |
| **efficiency** | optimal blocks over blocks actually walked, defined only when the walker arrived |
| **pass@1** | PRD **and** efficiency at least 0.75 |
| **explore %** | share of walker moves made under uncertainty rather than on a confident instruction |
| **capped** | the conversation ended at the 6000-tick simulated ceiling, 1200 simulated seconds |

Efficiency is undefined on a run that did not arrive: a halted walk has covered fewer blocks than
optimal and would otherwise score above 1.0. On the rerouting condition efficiency uses the
hindsight denominator, because the online optimum only rises as the agent blunders into barricades,
so scoring against it would reward worse play.

Explore % is labelled per move from the tick stream: a move follows either a confident instruction
("turn left and walk one block") or an uncertain one ("try heading that way and tell me what you
see"). It is a ratio of summed counts over a cell, not a mean of per-episode rates, and its interval
resamples conversations rather than moves because a lost walker guesses repeatedly.

---

## A.6 Results

Per system, realistic channel, 45 episodes each.

| system | PRD | pass@1 | explore % | capped |
|---|---|---|---|---|
| Gemini-3.1-Flash-Live | **0.978** | 0.489 | **16.2** | **0.04** |
| Grok Voice | 0.867 | **0.533** | 22.7 | 0.33 |
| GPT-Realtime-2.1 | 0.400 | 0.200 | 43.4 | 0.58 |
| GPT-Realtime-2.1-mini | 0.311 | 0.089 | 40.3 | 0.82 |
| Nova 2 Sonic | 0.000 | 0.000 | 51.2 | 0.91 |

Per condition, pooled over systems.

| condition | PRD | pass@1 | capped | median duration |
|---|---|---|---|---|
| base | 0.640 | 0.387 | 0.36 | 631 s |
| rerouting | 0.387 | 0.080 | 0.57 | 1200 s |
| long-horizon | 0.507 | 0.320 | 0.45 | 978 s |

Rerouting is the hardest by both measures, and its median conversation runs the full clock.

**Figure A.7** `base_pos` / `base_neg`, **Figure A.8** `reroute_pos` / `reroute_neg` and
**Figure A.9** `errand_pos` / `errand_neg` show matched successful and failed final states for each
condition, each pair drawn from the same scenario instance so the contrast isolates agent behaviour.

---

## A.7 Two findings

### A.7.1 The world is clock-limited, not accuracy-limited

Of 225 conversations, 110 did not arrive. **104 of those 110, or 95%, ended at the simulated step
cap with the walker still en route.** Only 6 were genuine wrong-destination errors.

This reframes every zero in the results table. A nav 0.00 means "did not finish within 20 simulated
minutes", not "went to the wrong place". Systems do not get lost here; they run out of time. The
practical reading is that the binding constraint on a voice copilot is instruction throughput, not
spatial reasoning.

The corollary is that every 0.00 is a lower bound. A longer cap would move these numbers up, and the
ranking is a ranking of speed as much as of correctness.

### A.7.2 Exploration separates systems, not conditions

Explore share and pass@1 move against each other **across systems**. Between the five system means,
**Spearman rho = -0.80**; across the 15 (system, condition) cells, **rho = -0.66, p = 0.008**, with
least-squares fit `pass@1 = 68.2 - 1.19 x explore%`. **Figure A.10** `fig_explore_vs_pass`.

**The sign flips with the level of analysis, and that is the finding rather than a caveat.** Within
a single system, across its three conditions, the mean rho is **+0.25** (Gemini +1.00, GPT-2.1
+0.50, mini 0.00, Grok -0.50; Nova undefined, since pass@1 is 0 in all three of its cells). In four
of the five systems the `base` condition carries both the highest explore share and the highest
pass@1.

So exploration is a **between-system marker, not a within-system axis**. A copilot that is generally
poor at holding the walker's position issues more uncertain instructions and scores worse, which is
the negative relationship. But making a task harder does not push a given copilot into more
exploration: on the harder conditions it issues *fewer* guidance moves, not more uncertain ones,
because it stalls rather than gropes. Any claim of the form "more exploration causes worse outcomes"
is unsupported here.

Three further limits carried on the figure:

- rho squared is 0.43, so even between systems exploration explains under half the rank variance.
- The ordering is not monotone: Grok explores more than Gemini (23% against 16%) and still scores
  higher on pass@1 (0.53 against 0.49).
- **37 of 225 episodes produced no guidance move at all** (Nova 23, mini 12, Gemini 1, GPT-2.1 1)
  and enter neither numerator nor denominator. Nova's three cells rest on 9, 15 and 17 labelled
  moves, so its extreme explore share is also its least reliable number.

## A.8 Belief: position is easy, heading is hard

`note_user_location(near_cross_streets, facing)` lets a copilot record where it believes the walker
is and which way it believes they face. Every recorded belief is scored against ground truth, so the
copilot's internal model is directly measurable rather than inferred from behaviour. This is the
world's most diagnostic instrument.

**Not every system uses it, and that is itself a result.** Accuracies below are pooled over
snapshots. Per-episode means differ slightly and are on the figure.

| system | episodes using it | snapshots | position | heading |
|---|---|---|---|---|
| Grok Voice | 7 of 45 | 15 | 1.000 | **0.933** |
| Gemini-3.1-Flash-Live | 44 of 45 | 340 | 0.953 | 0.550 |
| GPT-Realtime-2.1 | 40 of 45 | 106 | 0.968 | 0.396 |
| GPT-Realtime-2.1-mini | 41 of 45 | 128 | 0.928 | 0.352 |
| Nova 2 Sonic | 35 of 45 | 60 | 0.954 | 0.217 |

**Position is solved and heading is not.** Every system places the walker on the right corner 0.93
to 1.00 of the time. The same systems get the facing right 0.22 to 0.93 of the time, a spread four
times wider. **Figure A.11** `fig_belief_position_vs_heading`.

This follows directly from the interface in Section A.3: `get_gps` returns position for free and
never returns heading. Position can be looked up; heading must be inferred. The systems separate on
exactly the quantity that cannot be queried.

### A.8.1 The heading gap is mostly omission, not error

Decomposing every snapshot by what the `facing` field actually held changes the reading. Between
**27% and 58% of snapshots leave `facing` blank or "unknown"**. Accuracy *conditional on a facing
being stated* is much higher and much tighter:

| system | facing stated and correct | stated and wrong | left blank | accuracy when stated |
|---|---|---|---|---|
| Grok Voice | 93% | 0% | 7% | 1.00 (14 of 14) |
| Gemini-3.1-Flash-Live | 55% | 18% | 27% | 0.75 (187 of 249) |
| GPT-Realtime-2.1 | 40% | 9% | 51% | 0.81 (42 of 52) |
| GPT-Realtime-2.1-mini | 35% | 7% | 58% | 0.83 (45 of 54) |
| Nova 2 Sonic | 22% | 25% | 53% | 0.46 (13 of 28) |

So the headline spread of 0.22 to 0.93 is **not** four systems being bad at reasoning about
orientation. Four of the five are right 0.75 to 1.00 of the time whenever they commit to a facing;
they simply decline to commit on a third to a half of their beliefs. The discriminator is whether a
copilot tracks heading at all, not whether it can.

Nova is the exception and fails differently: it is wrong as often as it is right when it does
commit, 0.46, which is near chance on a four-way choice and is consistent with its zero task
success.

### A.8.2 Recording a belief and arriving: a reversal worth showing

Whether tracking belief is *associated* with arriving cannot be read off a pooled comparison,
because the systems that rarely call the tool are also the ones that rarely arrive.

| system | used, n | PRD | not used, n | PRD |
|---|---|---|---|---|
| Grok Voice | 7 | 1.00 | 38 | 0.84 |
| Gemini-3.1-Flash-Live | 44 | 1.00 | 1 | 0.00 |
| GPT-Realtime-2.1 | 40 | 0.45 | 5 | 0.00 |
| GPT-Realtime-2.1-mini | 41 | 0.34 | 4 | 0.00 |
| Nova 2 Sonic | 35 | 0.00 | 10 | 0.00 |
| **pooled** | **167** | **0.497** | **58** | **0.552** |

**Within every system the recorded-belief side is at least as good; pooled across systems the order
reverses.** That is Simpson's paradox, driven almost entirely by Grok, which calls the tool in only
7 of 45 episodes yet arrives in 39 of 45. **Figure A.13** `fig_belief_vs_outcome` plots the pooled
pair as a separate sixth group precisely so the reversal is visible rather than buried.

We therefore make no causal claim. The defensible statement is narrower and still useful: the
systems that maintain an explicit model of the walker are the systems that arrive, and the one
system that arrives without maintaining one (Grok) is also the one whose few recorded beliefs are
perfectly accurate.

Two cautions the figures carry on their face. **Grok's 0.933 heading rests on 15 snapshots from 7
episodes**, so it is the least certain number in the table despite being the largest, and it is not
comparable to Gemini's 340. And heading accuracy differs by up to 0.07 depending on whether it is
pooled over snapshots or averaged over per-episode rates; we report pooled throughout, and both are
printed on the figure.

**Figure A.12** `fig_belief_trace` makes it concrete on one conversation: 24 recorded beliefs,
position right on 22 of 24, heading right on 8 of 24, with **11 of the 24 leaving `facing` blank**.
The two position errors are instructive because they are not the same mistake: one places the walker
a corner ahead of where they are, the other a corner behind. Both are corrected at the next check,
within 16 seconds. The copilot arrived, in 21 blocks.

## A.9 A worked example

The rerouting repair sequence, quoted verbatim with tick indices, from a conversation in which the
copilot recovered and the walker arrived. **Figure A.14** `fig_zoom_reroute` shows the same moment
on the map, zoomed to the barricade.

Setup: the walker starts at Madison Avenue and E 44th Street facing south, heading for the Grand
Central Lexington Avenue entrance. Four segments are shut, none announced. Ticks are 200 ms.

```
[  838 |  167.6s] A: Yes, keep walking straight east on East 42nd Street. You'll reach Lexington
                     Avenue at the next corner. I'll let you know what to do when you get there.
[  887 |  177.4s] WALKER walk_to_next_corner()
                -> You can't get through. The pavement along E 42nd Street is barricaded for
                   construction and there is no way past on foot. You are still at Park Avenue
                   & E 42nd Street.
[  904 |  180.8s] U: The pavement is barricaded for construction, and there's no way through.
                     I'm still at Park Avenue and E 42nd Street. What should I do?
[  949 |  189.8s] A: I'll check for any other nearby closures and then re-plan a route around
                     this one.                          [utterance continues past the tool call]
[  956 |  191.2s] TOOL   live_status()
                -> - [seg_n_2_7__n_3_7] E 42nd Street closed between Park Avenue & E 42nd Street
                   and Lexington Avenue & E 42nd Street: pavement shed, no way through
[  970 |  194.0s] A: The closure is exactly on the block ahead of you on East 42nd Street. Turn
                     left onto Park Avenue and head north for one block until you reach E 43rd
                     Street. I'll give you the next turn from there.
                                                   [same utterance, resumes after the tool result]
[ 1050 |  210.0s] WALKER walk_to_next_corner()
                -> You walk the block and reach the corner. You're at Park Avenue and E 43rd Street.
```

The four beats, in order:

1. **tick 838**, the copilot routes the walker east on E 42nd Street, which is `seg_n_2_7__n_3_7`
   and is shut
2. **tick 904**, the walker contradicts it: barricaded, no way through, still at Park and E 42nd
3. **tick 956**, `live_status()` returns exactly that segment
4. **tick 970**, the copilot names the closure and issues a corrected instruction, which the walker
   completes at tick 1050

This conversation meets **two** of the four barricades, at Park and E 42nd and at Lexington and
E 45th, calling `live_status()` four times across the walk. It arrives, but takes **19 blocks for a
7-block job**, efficiency 0.368. Recovery is not the same as competence: the copilot repairs its
belief each time and still pays a large detour, because it repairs reactively rather than calling
`live_status()` before committing to a route.

Full excerpts for every condition, in both a successful and a failed conversation, are in
`geo_excerpts.md`, each verified line by line against the stored record.

---

## A.10 Limitations

**The cap dominates.** With 95% of non-arrivals ending at the ceiling, this is as much a throughput
benchmark as a navigation one. Reported numbers are lower bounds and a different cap gives different
numbers.

**Nine scenarios is a small between-scenario sample.** pass@k and pass^k are between-scenario
quantities, so their intervals resample scenarios and are correspondingly wide. Single-cell
differences are rarely resolvable; claims should be made at the system level.

**Ordinary destinations are authored.** Landmarks are real, goals are not, which is deliberate but
does mean the world is not a faithful map of Midtown East for any purpose beyond this benchmark.

**`directions()` mis-resolves one errand destination.** Asked for the United Nations visitor
entrance it returns Church Center for the United Nations, two blocks south. This affects all three
long-horizon scenarios, and part of the failure on that condition is ours rather than the agent's.

**Scoring is in blocks, not metres.** A route optimal in blocks may not be shortest in metres, given
the 3.4 to 1 aspect ratio.

---

## Figure index

All appendix figures are in `scratchpad/ms/out/figures/appendix/`, each in two themes as
`<name>_light.png` and `<name>_dark.png`. The six final-state maps are in
`scratchpad/ms/out/figures/` and are light only, because their three-route layering was tuned on a
light ground.

| figure | file | px | what it shows |
|---|---|---|---|
| A.1 | `fig_anatomy` | 2360 x 1860 | every feature class drawn on the map, with counts |
| A.2 | `fig_scale` | 2240 x 1528 | lattice scoring space against true metric proportions, 3.38 to 1 |
| A.3 | `fig_tools` | 2360 x 1744 | the 14 tools, colour-coded by whether the walker or the copilot holds them |
| A.4 | `fig_cond_base` | 2000 x 2644 | base scenario, the forced E 42nd crossing, one of four tied optima |
| A.5 | `fig_cond_roadblock` | 2000 x 2644 | clear-map route, four barricades, the unique 7-block detour |
| A.6 | `fig_cond_errand` | 2000 x 2644 | two legs, second goal revealed on arrival |
| A.7 | `base_pos`, `base_neg` | 1800 x 1800 | matched success and failure, base |
| A.8 | `reroute_pos`, `reroute_neg` | 1800 x 1800 | matched success and failure, rerouting |
| A.9 | `errand_pos`, `errand_neg` | 1800 x 1800 | matched success and failure, long-horizon |
| A.10 | `fig_explore_vs_pass` | 2600 x 1760 | explore share against pass@1, 15 cells, with the fit |
| A.11 | `fig_belief_position_vs_heading` | 2680 x 1800 | position against heading, plus the blank-field decomposition |
| A.12 | `fig_belief_trace` | 2720 x 2160 | belief against truth across one conversation, 24 snapshots |
| A.13 | `fig_belief_vs_outcome` | 2680 x 1720 | arrival split by belief use, with the pooled reversal |
| A.14 | `fig_zoom_reroute` | 2000 x 2612 | the barricade moment, six numbered beats |

Figure A.14 is the only appendix figure that draws a walked path; A.1 to A.6 are task definitions
and A.10 to A.13 are charts.

---

## Reproducing every number and figure

| artefact | command |
|---|---|
| per-episode data, 630 rows | `scratchpad/ms/out/merge/nav_per_episode.csv` |
| geometry, anatomy and tool figures | `.venv/bin/python scratchpad/ms/geo_appendix_geom.py` |
| condition and zoom figures | `.venv/bin/python scratchpad/ms/geo_appendix_conds.py` |
| exploration and belief figures | `.venv/bin/python scratchpad/ms/geo_appendix_evidence.py` |
| final-state maps | `.venv/bin/python scratchpad/ms/geo_figures.py` |
| verbatim excerpts and their check | `scratchpad/ms/build_excerpts.py`, `verify_excerpts.py` |

The belief figures read a cache built by `scratchpad/ms/build_belief_cache.py`, which replays all
225 episodes once into `scratchpad/ms/out/belief_cache.json`.
