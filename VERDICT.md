# THE VERDICT — why we lose at matched budget, and what we build

**Chief architect's synthesis of the five-hypothesis strike. 2026-08-28.**
Evidence of record: `strike/BRIEFS_SUMMARY.md`, re-verified against `main` @ `9ddcfa3` and against
sheltur (read-only) tonight. Corrections to the briefs are marked **[VERIFIED]** / **[CORRECTED]**.

---

## (a) The diagnosis, in one paragraph

Our AI is not out-thought. It is misinformed. Given the same clock and the same number of imagined
futures as Foul Play, it loses four games in five — because the battle it imagines is not the battle
it is in. Two defects do it, and both are in the *inputs* to the search, not the search itself.
**First, we invent an opponent who does not exist.** When we guess what the enemy is carrying, we
pick the most common item, then *separately* the most common ability, then *separately* the four most
common moves, and staple them onto one Pokémon. The result is a composite monster no player has ever
brought to a ladder — it holds an Assault Vest and still uses Calm Mind. Foul Play instead draws one
*real, complete* set that a human actually plays. We measured our version: four times in ten, the
opponent we searched against did not even own a move the real opponent went on to use, and at the
matched setting this happens on **100%** of decisions. Worse, taking the most-likely-of-everything
builds an opponent strictly scarier and more versatile than any genuine one, which quietly pushes us
into over-respecting threats and over-switching — the same behaviour as our 999-turn switch loop and
our human-ladder losses. **Second, the board we hand the search is missing pieces of the real board:**
how long the enemy has been asleep, whether they are locked into a Choice item, incoming Wish and
Future Sight, Substitutes, Leech Seed, remaining PP. All of it is public information sitting in the
battle log. All of it is hard-coded to zero. By the fourth imagined turn, only one line in five still
describes the real game. So a search using a position-judge that is *far* better than Foul Play's
(0.748 vs 0.544) spends its 100ms brilliantly solving the wrong problem. **The fix is not a new brain.
It is to stop lying to the brain about what it is fighting.**

---

## (b) Ranked mechanisms — resolving H1 vs H2

**Tension 1 resolved.** H1's compute multiplier is real and it is *irrelevant to the matched gap*.
FP is pinned to `--search-parallelism 1` in our harness; its 2-4× world fan-out cannot fire. It is a
**ladder-parity** finding, not a matched-sim finding — it tells us ladder-FP is stronger than our
benchmark-FP, so our 18% is measured against a *weakened* opponent. The matched gap must therefore be
explained entirely by what we do differently at K=1 — and **[VERIFIED tonight on the box]** the rig
does pin `ECLIPSE_ENGINE_DETERMINIZATIONS=1`, `ECLIPSE_ENGINE_BUDGET_MS=100`. Critically, **most of
H3's ledger is inert at K=1**: the budget split (`EngineMctsSearch.kt:121`, `budgetMs / k`), the
iteration-cap split (`:128-129`), the #492 K≥2 fold and adaptive-K all no-op at k=1 — and the "root
candidate restriction" is inert too, because the beam knobs are explicitly `IGNORED_UNDER_MCTS`
(`EngineMctsLane.kt:148-156`) and every candidate enters via `RootCandidates.joinable`. That
mechanically demotes H3 and promotes H2/H4, which are 100%-prevalence at exactly the matched setting.

> **Caveat worth its own line: we benchmark at K=1 but we *ship* K=2** (`DEFAULT_DETERMINIZATIONS = 2`,
> `EngineSearchSettings.kt:446`). At K=2 the budget split and the untokened synthetic-residual
> incoherence are both live. Benchmark results do not transfer to the live server for free, and the
> coherent-world fix must cover the K≥2 path or the servers keep the disease we just cured.

Shares are expressed as pp of the **~32pp deficit to parity** (18% → 50%). They are sub-additive: M1
and M2 share a root (a real set carries the real EV/nature/item that the fidelity audit names as its
own #1 error lever), and M3 is partly *caused* by M1.

| # | Mechanism | Prevalence (measured) | pp-share | Confidence | Status |
|---|---|---|---|---|---|
| **M1** | **Uncollapsed / incoherent opponent world.** `Determinizer.kt:223` short-circuits `if (k == 1) return mixtureOnly(...)`; `TheirSideMapper` draws ability (`:241`), item (`:274`), moves (`:324`) from three independent posteriors. **No legality filter exists anywhere in the mod** — grep for `set_makes_logical_sense`/legality returns nothing. | 100% of K=1 decisions carry `MIXTURE_WORLD_NOT_COLLAPSED`; 40.7–42.6% of contested decisions searched a world missing a later-revealed move; ≥0.4–0.5% rules-violating worlds | **12–16** | MED-HIGH | **[VERIFIED]** in code. Magnitude inferred — no counterfactual ever run |
| **M2** | **Transition infidelity.** `EngineState.kt:30,32` default `wish`/`futureSight` to zero and `OwnSideMapper.kt:111-117` never passes them; `VolatileMapper.kt:56-66` hardcodes `confusion/lockedMove/slowStart/yawn = 0`; `EngineState.kt:26,28` drop `volatileStatuses`/`substituteHp`; `lastUsedMove` (poke-engine's **choice lock**) left `""`; opponent PP flat `DEFAULT_PP=16`. | 100% of 39,803 searched decisions; ply1→2 error +54%; only 21.7% of lines real by ply 4 | **6–10** | MED-HIGH | **[VERIFIED]**. `lockedMove=0` is a legality violation. Named autopsy losses exist (Specs Enamorus, Future Sight) |
| **M3** | **Switch-economy attractor / degenerate cyclic play.** 999-turn cap-out exhibit: 822/1016 decisions an alternating switch loop. PR #203 family **open**; findings #204-#213 built but config-default-OFF. | 1 catastrophic exhibit + a human-autopsy loss at −28.71 search value | **3–5** | MEDIUM | Partly downstream of M1 — do not double-count |
| **M4** | **Leaf blind to item/ability.** **[VERIFIED]** zero of the 214 leaf features name an item or an ability (`LEARNED_LEAF_FEATURE_MAP.md`). But item/ability *do* cross the wire (`PokeEngineFormat.kt:137,142`) and poke-engine has rules for them — so coherence pays through the **simulator**, and the blind leaf **caps** the payoff. | structural | **0–3 now; 3–6 after M1** | LOW-MED | A *conditionally activated* lever. This is why H1's judge-swap read bit-identical |
| **M5** | **Procedure deviations *actually live* at K=1** — shrinks on inspection to: the hard-coded root mask (`EngineMctsHygiene.kt:98-101`), the hard-coded tiebreak (`:138,174-177`, `FLAT_BAND=0.10`), and unconditional AUTHORITY_MARGIN laundering (`EngineMctsDecision.kt:145` — applied with no guard). Root restriction is **inert under MCTS**. | mask+tiebreak+prior bundle A/B'd **NULL** at 1,198 games | **0–2** | MEDIUM | Laundering never isolated; **none of the three has a knob** — Arm C needs code |
| **M6** | **FP's compute multiplier** (2-4× worlds on uncertain turns, each with full budget). | not active in our harness | **0 matched / material on ladder** | HIGH | Resolves H1 vs H2. Ladder-only; forbidden by the 500-player constraint |
| **M7** | **Judge quality.** Ours AUC 0.7481 vs FP 0.5436; isolated swap bit-identical 67/600. | — | **0** | HIGH | **Closed as a primary lever.** Reopens only via M4 |

Corroborating datum for the M6/H3 split: our own sweep found **"D2 (2 worlds) +4.33pp graduating and
D4 (4 worlds) +1.33pp not, at 25ms — two well-fed worlds beat four starved"** (`EngineSearchSettings.kt:106-108`).
Starved worlds are worse than fewer fed ones — i.e. FP's full-budget-per-world *is* a real edge at
K≥2, and our ÷K split is a genuine defect **outside** the matched condition.

**[CORRECTED] Four brief claims did not survive verification:**
1. "coherent-world and state-fidelity are **being built tonight**" — **both worktrees are byte-identical
   to `main`, zero commits, clean.** Nothing has been built. The fix wave has not started. This is the
   single biggest change to the plan below.
2. AGENTS.md says finding #28 (opponent-move observation) needs its own PR — the audit records it
   **FIXED in round 3**. Minor doc debt.
3. "Two boxes." There is **one** sims box (sheltur, 24 cores, RAM-bound) plus this 12-core workstation
   acting as the farm. Plan capacity as ~1.4 boxes, not 2.
4. The no-belief arm was filed as "blocked / never run" — it is in fact **one environment variable
   away**, and has been for as long as the harness has existed. That is the most expensive thing in
   these briefs: an assumption at the foundation of the campaign, testable in two hours, untested.

---

## (c) The build plan

### Instrument reality (this governs everything)

- 600 pairs ⇒ CI half-width **±3.5pp** (gate audit: 94W vs 106W, discordant 53/65, CI [−5.54,+1.54]).
- Frame noise on **frozen** binaries ≈ **3.6pp** (`PAIRING.md:31`). Trajectory pairing lifted attribution
  15.0% → **67.2%** on the first real confirmatory (70% on the A-vs-A probe). Leg 3 — both searchers'
  own Monte Carlo — is still unseeded.
- ⇒ **MDE ≈ 5pp at 600 pairs, ≈3.5pp at 1,200.** Our estimated fix effects (6–16pp) clear 600; the
  *marginal* contrasts (fidelity alone, 6–10pp) sit on the boundary and need 1,200.
- Throughput: **1,200 battles = 600 pairs ≈ 2h** on 8 lanes (series16-box: 7,098s, fairness 0.148 FP
  cpu-s/turn, in band). **Compute is not the bottleneck — build capacity and decision hygiene are.**
- **[CORRECTED] We do not have two equal boxes.** Sheltur is 24 cores but **RAM-bound** (11GB
  available; ~1 extra 8-lane arm alongside the running generation job). The "farm" is this 12-core
  workstation, which ran ~1,100 of cycle 4 — and which has already once been reported as running
  when it had zero activity. It gets a named owner and a heartbeat before it is counted.
- Cycle-4 generation: 261/2,335 at 8.42 games/min, **ETA ~01:50**. Arms get the box after that.

### Tension 2 resolved — the experiment chain: a LADDER, not a factorial

**One jar, knob-selected arms, shared seeds, trajectory-paired.** Building separate jars per arm
reintroduces a build confound and breaks jar-pair continuity (a hazard this campaign has already been
bitten by). Arms are configuration, always.

| Arm | Config | Contrast it buys |
|---|---|---|
| **A0** | today's `main` (MIXTURE, fidelity off) | shared baseline |
| **A1** | fidelity ON (both sides), mixture unchanged | **A1−A0** = M2 alone |
| **A2** | fidelity ON + `DETERMINIZE_MODE=COLLAPSED` + legality filter | **A2−A0** = the wave (best-powered); **A2−A1** = M1 marginal |

Three arms over 600 shared seeds = 1,800 battles ≈ **3h**. Read A2−A0 first: it is the largest
expected effect and therefore the only contrast reliably powered at 600. If A2−A0 ≥ +5pp, extend the
*marginals* to 1,200 seeds; do not spend on marginals before the wave clears.

**Why not the 2×2 factorial:** the missing cell (coherence without fidelity) is a build nobody would
ship, and an interaction term needs ~4× the n of a main effect — against a 3.6pp noise floor we cannot
estimate an interaction we could act on. We buy three contrasts from two treatment arms instead of
four arms for a term we would ignore.

Then the two experiments worth more per game than any fix measurement, because they test *assumptions*:

- **Arm B — NO-BELIEF (600 seeds, ~2h). ⭐ CONFIG-ONLY — RUNNABLE ON TONIGHT'S JAR, NO BUILD.**
  **[VERIFIED]** `ECLIPSE_BELIEF_NO_CONSUME=1` (`HeadlessDecider.kt:105`, applied `:439-440`) pins
  `TurnState.beliefs` to `NoBeliefs`, whose `determinizations()` returns empty
  (`OpponentBeliefs.kt:183`) and collapses the tree to the neutral uniform-over-legal-set opponent
  (`OpponentActionModel.kt:148`). It has **never been run**, blocked by a since-retired
  results-over-ablation directive. "Belief is our advantage" has been asserted for the entire campaign
  and never measured. If no-belief ≈ belief, our belief investment is decorative; if no-belief >
  belief, it is *negative* and M1's fix is merely getting us back to zero. **This is the single
  highest information-per-game experiment we own and it costs zero engineering.** Do not conflate with
  `ECLIPSE_BELIEF_PRIORS_ONLY` (seats priors, skips evidence) — that is a useful *third* rung, run it
  as B′ if the box has room.
- **Arm C — VANILLA PROCEDURE (600 seeds, ~2h). NEEDS CODE** — mask, tiebreak, and AUTHORITY_MARGIN
  are all hard-coded with no knob (`EngineMctsHygiene.kt:98-101,138`; `EngineMctsDecision.kt:145`).
  Upstream leaf + no prior/mask/tiebreak + raw MCTS argmax. This **prices the cheapest possible
  "rework"**: if vanilla-on-our-engine beats us by >5pp, the rework is a *deletion project*, which is
  the best news available.
- **Arm L — LADDER-FP CALIBRATION (300 seeds, ~1h, once).** FP at its *default* parallelism. Sizes M6
  honestly so we never again declare progress against a weakened opponent.

Every arm preregistered before launch (house pattern: `docs/k_sweep_prereg_draft.md`), own-head arming
stamps, budget parity checked in the 0.147–0.177 FP-cpu-s/turn band, reported as paired discordance +
CI — never raw win rate.

### Tension 3 resolved — incremental AND one structural bet, in that order of dependency

The fix wave is not the conservative choice; it is the **prerequisite** to every structural bet. You
cannot weaponise a belief system that is wrong 41% of the time. You cannot distil a better evaluator
whose inputs are fictional. You cannot fairly test ISMCTS against a determinizer that never samples.
So: incremental *first by dependency*, not by timidity — and one structural bet starts building in
parallel, because it is on the critical path and does not wait on the wave's result.

#### THE STRUCTURAL BET: "The Exploiter" — belief as a WEAPON (H5 #1)

**Why this one, and why it is not a carbon copy of FP.** FP plays a generic, memoryless, approximate
equilibrium: a 16-term hand heuristic, usage-stat worlds, no cross-battle memory, no learning loop, and
a provable non-convergence in simultaneous-move subgames. Copying its search gets us, at best, to FP.
The thing FP **structurally cannot do** is know *this specific opponent* and punish them. We already
own every piece of that: a per-opponent belief engine, cross-battle opponent memory (armed, #309), a
per-decision corpus, a nightly retraining loop, and a value head with 38% more discriminative power
than FP's. H2 found our belief system is currently a *liability*. That is precisely why it is the
right place to build: it is the only component where we are both behind and structurally ahead.
It is also the component that beats **humans**, who are far more exploitable than an equilibrium — and
humans, not FP, are the stated objective.

Staged, with kill criteria:

- **Stage 1 — make the belief true.** = the fix wave (M1+M2). **KILL:** if A2−A1 ≤ 0 at 1,200 pairs,
  coherent determinization is not our lever → escalate to ISMCTS + regret matching as a search rework.
- **Stage 2 — prove the belief is an asset.** = Arm B re-run *after* Stage 1 lands. **KILL:** if
  no-belief is within noise of belief-on post-fix, freeze all belief investment and move the slot to
  search and eval features. This gate is non-negotiable; it is the assumption the campaign is built on.
- **Stage 3 — exploit.** Two cheap components, both reusing existing machinery:
  (i) **response modelling** — replace the generic opponent policy at opponent nodes with our learned
  policy prior *conditioned on* the belief and the cross-battle memory (`mctsPriorPolicy` already
  emits per-candidate priors; #309 already stores per-opponent history);
  (ii) **calibrated restricted-Nash mixing** — one exploitation parameter λ blending the safe search
  policy with the best response to the modelled opponent, with λ set **per opponent by our observed
  prediction accuracy against them** (predicting well ⇒ exploit; predicting badly ⇒ fall back). The
  one-shot-calibration risk H5 flags is exactly what per-opponent accuracy gating answers.
  **KILL:** no ≥3pp vs the Stage-1 build at 1,200 pairs *and* no gain on the human/ladder set within
  two weeks ⇒ shelve.

#### THE SMALL STRUCTURAL BET: root-level regret matching (H5 #2, cheap version)

DUCT provably fails to converge on simultaneous-move RPS subgames, and Pokémon is full of them. Our
own pathological exhibit — 822/1016 decisions an alternating switch loop, capping out at 999 turns —
is the exact signature of a deterministic policy trapped in a cyclic subgame. We already sample
stochastically (softmax in a 6.0-pt window) but we mix by **score**, which has no convergence property;
regret matching mixes by **regret**, which does. Cheap first version: compute a regret-matching mixed
strategy from the root's own joint visit/value statistics and sample from that instead of the
score-softmax. **No engine change, no Rust — a root post-processing step.** And note our iteration
budget is ~61k median, i.e. the *large*-sample regime where regret matching is favoured, not the small
regime where H5 warns DUCT wins. **KILL:** if it neither removes the cap-out class nor gains ≥3pp,
keep only if it kills cap-outs.

#### Explicitly rejected / deferred

| Option | Ruling | One-line reason |
|---|---|---|
| RL / transformer / self-play-from-scratch pivot | **REJECTED** | FP beat every RL and LLM entrant 50-14 at NeurIPS 2025; VGC-Bench agents ~100% exploitable at accessible scale; GAE breaks under imperfect-info self-play — and a net policy violates the 500-player strength-per-ms constraint |
| More judge/eval tuning as a standalone line | **REJECTED** | Isolated swap was bit-identical 67/600 at AUC 0.748 vs 0.544 — the eval is not the binding constraint |
| Carbon-copying FP's compute multiplier as a strength project | **REJECTED as strength; kept as a ladder-only mode** | It is the opposite of our product constraint; but we must *measure* it (Arm L) or we are benchmarking against a weakened FP |
| Pluribus-style abstracted blueprint | **DEFERRED indefinitely** | No instrument validates an abstraction; unbounded design risk |
| True ISMCTS | **DEFERRED, not rejected — it is the escalation** | It is the principled version of the coherent-world fix; Ihara's result is at *fixed iterations* and our budget is wall-clock. Bad first bet, good second one. Triggered by Stage-1's kill criterion |
| NNUE-distilled eval | **DEFERRED behind its precursor** | Worthless until the world is coherent and the wire carries item/ability. The precursor (extend the 214-feature leaf) *does* get a slot |
| 2×2 factorial of the two fixes | **REJECTED** | Missing cell is unshippable; interaction needs 4× the n against a 3.6pp noise floor |
| Rewriting the search engine now | **REJECTED for now** | The evidence indicts the search's *inputs*, not the search. Arm C and Stage-1's kill criterion are the two triggers that would change this |

#### The next two weeks, as a slate

| Slot | Work | Gate it must clear |
|---|---|---|
| 1 | Fix wave lands as default-on (M1+M2) | A2−A0 ≥ +5pp at 600 pairs; `lockedMove` correctness ships regardless |
| 2 | **Exploiter Stage 3** — response modelling + calibrated λ | ≥3pp vs Stage-1 build at 1,200 pairs, or gain vs humans |
| 3 | Leaf feature extension (item/ability/volatiles → the 214-feature map) | must clear the flywheel's honest AUC gate *and* a paired arm — AUC alone is not a strength claim |
| 4 | Root regret matching (cheap version) | removes the cap-out class, or ≥3pp |
| 5 | K≥2 repair for the shipped config (`ECLIPSE_WORLD_BUDGET=FULL` + residual-cluster coherence) | the servers run K=2; the benchmark does not |
| — | *Held in reserve, triggered not scheduled:* ISMCTS (on Stage-1 kill), wrapper deletion (on Arm C win) | |

### Tension 4 — the flywheel's role: retargeted, not retired

Two cycles refused; cycle 3 missed by 0.00014 with human strata newly positive; the gate audit then
showed the refusal was *correct* (−0.0034 AUC ↔ −2.0pp in games, null, CI [−5.54,+1.54]). Read
honestly: **more self-play data against the same distribution does not convert to wins; opponent
diversity was the ingredient that broke the dose-response curse.** So the flywheel stops being a
"train a better judge" programme and takes three concrete jobs:

1. **Corpus factory for the arms.** Every experiment arm's decision logs are training data — the
   measurement programme and the training loop become the same programme, for free.
2. **Feature-extension vehicle (the NNUE lesson, applied where it converts).** The M2 fix *creates new
   observable columns* — choice lock, sleep turns, Wish/Future Sight, Substitute, volatile durations,
   PP — that neither our leaf nor FP's heuristic reads. Retrain the value head *with* them. This is a
   capability FP cannot match, and it is the activation of M4.
3. **Opponent-model trainer for Stage 3.** Archetype- and opponent-conditioned priors, retrained
   nightly from the corpus. This is the Exploiter's fuel and it exists nowhere else.

Discipline: never report a flywheel promotion as a strength gain. Sign agrees, magnitude does not.
Resource ruling: **cycle 4's read must not consume tomorrow's box time — the arms have priority.**

### Tension 5 — would "reworking the entire AI" be justified? Honestly: no, but a third of it, yes.

The phrase has three concrete meanings and they get three different answers.

1. **Rework the paradigm** (RL/transformer/learned policy replacing search). **NO — and the evidence is
   strong enough to say so flatly.** Search + heuristic won the field; learned agents are exploitable
   at our budget; and it breaks the product constraint. This is the one direction where "think outside
   the box" would be a mistake, and I will say so rather than flatter the ambition.
2. **Rework the wrapper** (delete our 13 procedural deviations, run FP's procedure on our engine).
   **CHEAP AND UNPRICED — Arm C prices it in two hours.** If it wins, the rework is a deletion project,
   which is the best outcome on the table. We have never run it paired. That is an embarrassing gap and
   it closes this week.
3. **Rework the information layer** (stop determinizing from independent marginals; move to coherent
   sampling, complete state, and an opponent model that is exploited rather than averaged over).
   **YES — this is where the evidence actually points, and it is a genuine rework of roughly a third of
   the system**: `Determinizer`, `TheirSideMapper`, `OwnSideMapper`, `VolatileMapper`, `EngineState`,
   the belief→wire contract, and eventually the leaf's feature set. The fix wave is its first two
   commits; the Exploiter is its endgame; ISMCTS is its escalation.

**What would change this verdict:** Arm C beating us by >5pp moves the indictment to the wrapper. A2−A0
coming back null *after both fixes land clean* exonerates the information layer entirely and moves the
indictment to DUCT itself — at which point ISMCTS + regret matching becomes a full engine rebuild and
earns it. Both triggers are answered inside 72 hours. We are three days from knowing, not three weeks.

---

## (d) FIRST BUILD ORDER — tomorrow morning, 2026-08-29

House rules binding on every lane: fresh worktree, **one lane = one worktree = one writer**, disjoint
file sets, adversarial verification pass over the diff before commit, Conventional Commits, **no
Co-Authored-By trailer**, never touch `main`, subagents get an explicit model (`opus` for these three
build lanes). Standing risk: the `claude-review.yml` starvation bug (~12 starvations yesterday) — with
three lanes landing in one day, fix or budget for it, and **read the PR's actual review comments for
the head SHA before any starvation waiver**.

**Lanes are file ownership. Knobs are arms. They deliberately do not line up — the opponent-side
fidelity work lives in Lane 2's file but ships under Lane 1's knob.** Say this in both briefs.

**Lane 0 — THE FREE EXPERIMENT, and it goes first (no code, no PR, no build).** The moment cycle-4
generation clears the box (~01:50), launch **Arm B** on the *existing* jar:
`rig/run_parallel.py --pair-trajectories --knob ECLIPSE_BELIEF_NO_CONSUME=1`, 600 seeds, 8 lanes,
against the same seeds as a belief-on control, ~2h. Prereg the gate before launch. Nothing else in
this plan costs so little or can change so much: it decides whether the belief layer — the component
the whole structural bet is built on — is an asset, a wash, or a liability. If it reads *liability*,
Lane 2 changes from "make the belief coherent" to "make the belief optional", and the two-week bet
changes with it. **Nobody starts Lane 2 before Arm B is launched.**

**Lane 1 — `claude/state-fidelity`** (worktree exists, empty). *Owns:* `EngineState.kt`,
`OwnSideMapper.kt`, `VolatileMapper.kt`, `PokeEngineFormat.kt`, own-side `AdapterFidelity` tokens.
Ship behind `ECLIPSE_ADAPT_FIDELITY=on|off`, **default off for the measurement window only**.
Content: own-side `wish`/`futureSight` from `MoveHistoryTracker.wishResolveTurn`; volatile durations
`confusion`/`lockedMove`/`slowStart`/`yawn` from real field durations; `volatileStatuses` +
`substituteHp` from `BattleStateReader`/`BattleMemoryMoveEffects`; toxic counter from
`ResidualDamageCalculator`. **Non-goal: the opponent side** (Lane 2 owns that file). Note in the PR
that `lockedMove=0` is a legality violation, so this knob's default flips to ON the moment the ladder
reads out, regardless of the pp result.

**Lane 2 — `claude/coherent-world`** (worktree exists, empty). *Owns:* `Determinizer.kt`,
`TheirSideMapper.kt`, `AdapterFidelity.kt`. Three commits:
(a) `ECLIPSE_DETERMINIZE_MODE=MIXTURE|COLLAPSED`; under COLLAPSED remove the `k == 1` short-circuit at
`Determinizer.kt:223` and collapse onto one real set-cluster — **and cover the K≥2 synthetic-residual
cluster too**, which today falls through to the same three marginals and emits *no* token (the k-sweep
prereg calls the ≈100%/≈0% figure a lower bound; it is right).
(b) A legality filter rejecting any sampled world whose item/ability/move combination carries zero
posterior mass (the AV + Calm Mind class), with its own gap token.
(c) Opponent-side fidelity, gated by **Lane 1's** `ECLIPSE_ADAPT_FIDELITY`: `lastUsedMove` from
`OpponentBeliefs.choiceLockProbability` (sample per world under COLLAPSED — more honest and free),
opponent volatiles, opponent PP where the protocol reveals it.

**Lane 3 — `claude/arm-knobs`** (new worktree). *Owns:* `EngineMctsHygiene.kt`, `EngineMctsDecision.kt`,
`EngineMctsSearch.kt`, harness bridge lane files. Adds only the switches that **verifiably do not
exist**, all defaulting to current behaviour, all stamped into the decision record's manifest so an arm
can prove it was armed: `ECLIPSE_SEARCH_VANILLA=1` (Arm C — bypass the hard-coded mask at
`EngineMctsHygiene.kt:98-101`, the hard-coded tiebreak at `:138`, and the unconditional AUTHORITY_MARGIN
laundering at `EngineMctsDecision.kt:145`; combine with the existing `ECLIPSE_ENGINE_MCTS_LEAF=UPSTREAM`
and `ECLIPSE_ENGINE_MCTS_PRIOR=NONE`), and `ECLIPSE_WORLD_BUDGET=SPLIT|FULL` at `EngineMctsSearch.kt:121`
and `:128-129` (isolates the ÷K defect; inert at the benchmark's K=1, live on the shipped K=2).
**No belief knob is needed — `ECLIPSE_BELIEF_NO_CONSUME` already exists.**

**Lane 4 — measurement (no mod code).** Write the ladder prereg (arms, seeds, gate rule, parity band,
stop rules) and a `RELAUNCH_LADDER.md` runbook so re-runs use cheap fresh agents rather than replaying
a warm agent's transcript. **Verify the 3090 farm is synced to the new jar, with a named owner and a
heartbeat visible on the dash** — it has been reported running while idle once already; that does not
happen twice.

**Sequencing.** Lane 0 launches first and needs nobody. Lanes 1-3 build in parallel (disjoint by
construction). Merge all three, then build **ONE** jar from merged `main` — never per-arm jars; arms
are configuration, always. Box discipline unchanged: one heavy phase at a time, RAM is the binding
constraint, monitor MemAvailable **and SwapFree** (the swap floor flagged as missing is still missing),
nothing goes near the sacred ports.

**72-hour arc.**
*Tonight (post-01:50):* **Arm B** on the free jar. Zero engineering, two hours, answers the campaign's
founding assumption.
*Day 1 (08-29):* read Arm B first thing — it may re-scope Lane 2 before Lane 2 has written a line.
Then three build lanes + the ladder prereg; cycle-4 gate read (must not take box time from the arms);
farm synced, owned, and heartbeating. Evening: one jar, launch the **A-ladder** (A0/A1/A2, 600 shared
seeds, ~3h) on sheltur; **Arm L** (ladder-FP calibration, 300 seeds) on the farm if the farm is
verified — otherwise it queues behind the ladder.
*Day 2 (08-30):* read **A2−A0**. ≥ +5pp ⇒ extend the marginals to 1,200 seeds and open the leaf
feature-extension lane (item/ability/volatile columns — M4's activation). Null ⇒ launch **Arm C**
immediately, open the root regret-matching lane, and treat the information-layer hypothesis as wounded.
*Day 3 (08-31):* Arm C read; Exploiter Stage-3 design lands (response modelling on the existing
`mctsPriorPolicy` machinery); the two-week bet is formally committed **with its kill criteria written
down before a line of it is built** — that ordering is the discipline that has been missing.

---

## The one-line version, for the owner

*We do not need a new brain. We need to stop lying to the brain about what it is fighting — and then
turn the thing we lie with, the opponent model, into the weapon Foul Play structurally cannot build.
The cheapest and most important test of that whole idea costs one environment variable and two hours,
and it runs tonight.*
