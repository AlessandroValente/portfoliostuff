# MADDPG.jl — Handoff for a fresh session

Paste this at the start of a new Cowork session. I will also attach an
earlier version of the code so you can compare against the current state.

---

## 1. What this is

I'm a university student building **MADDPG.jl**, a Multi-Agent Deep
Deterministic Policy Gradient solver for **differential games**, in Julia. It
sits in the **JuliaDifferentialGames** ecosystem alongside my advisor
Bennet's package **DifferentialGamesBase.jl**
(https://github.com/JuliaDifferentialGames/DifferentialGamesBase.jl — public;
verify constructor signatures against the source before calling into it).

**Environment:** Julia 1.12, Windows, PowerShell, VS Code.
**Package:** `C:\Users\vales\Documents\MARL\maddpgpt1`
Run things from the package root with `julia --project=.`

I started with no prior Julia, RL or game theory experience, so please explain
reasoning and give keystroke-level instructions for anything I need to run. I
will push back when something looks wrong — please engage with that rather
than agreeing.

---

## 2. What exists now

```
src/
  MADDPG.jl                 module, exports
  environment.jl            GameProblem -> CommonRLInterface wrapper.
                            Constraint modes (:barrier default, :penalty,
                            :none), barrier_cap, and a state_projection hook
                            applied after each integration step
  cooperative_navigation.jl 3 agents, 3 landmarks, shared coverage cost
  predator_prey.jl          3 predators + 1 faster prey (accel 4.0 vs 3.0),
                            hard arena wall, polygon obstacles, and three
                            reward modes: :distance, :sparse, :sparse_team
  obstacles.jl              convex polygons as half-planes; signed clearance;
                            blocking projection; barrier constraints;
                            min_width (tunnelling guard)
  networks.jl               Flux actor/critic builders (Float64)
  replay_buffer.jl          shared circular buffer
  agent.jl                  Agent + update_agents!, gradient clipping
  solver.jl                 MADDPGSolver + train!
scripts/
  train_cooperative_navigation.jl
  train_predator_prey.jl
  plot_random_policy.jl / plot_random_predator_prey.jl   500-episode baselines
  analyse_run.jl            regression/t-test on a learning curve CSV
  reproduce_1500.jl         regenerates an old figure with pinned old params
sgnep/                      stochastic LQ game solver (see section 4)
test/runtests.jl            full suite; passes
```

`train!` returns `(agents, history, agent_history, evals)` where `evals` holds
greedy-evaluation rewards, critic |Q| and the exploration noise level. Both
training scripts save CSV histories so plots can be remade without retraining.

---

## 3. Verified results

- **Cooperative navigation**, 25k episodes: plateaus ≈ **−45** with **all
  three landmarks covered**.
- **Predator–prey**: with dense distance shaping the predators **never caught
  the prey**. With the paper's sparse tag reward shared across predators
  (`:sparse_team`) they reach **~27% of each episode in contact** (mean
  episode reward ≈ 6.7). Mechanism: the prey is 33% faster, so a straight-line
  chase cannot work, and per-predator distance rewards produce exactly that.
- **Obstacle barrier at full strength destroyed the objective** — the prey's
  mean per-step reward fell from +0.64 to +0.003. Obstacle barriers are now
  off by default (the obstacles are impassable anyway, so the barrier was pure
  shaping). The arena barrier is kept for continuity with earlier runs.

---

## 4. The stochastic LQ solver (`sgnep/`)

`DifferentialGamesBase` declares `AbstractStochasticGame` but has **no
concrete `StochasticGameProblem`**. I implemented one, plus a solver:

- `SGNEP.jl` — `StochasticGameProblem` (wraps a nominal `GameProblem` + a
  `NoiseModel`: additive, state-multiplicative, control-multiplicative)
- `SFNELQ.jl` — coupled Riccati recursion for the risk-neutral feedback Nash
  equilibrium
- `cwh.jl` — Clohessy–Wiltshire–Hill dynamics + a three-satellite formation
  flying game
- `validate_formation.jl` — four checks, all passing: Monte Carlo vs analytic
  (**z = 0.01 / 0.19 / 0.42** over 2,000 runs), best response reproduces
  equilibrium gains, single player reduces to textbook LQR, price of anarchy
  1.0623×
- Both files are written without imports so they can drop straight into
  DifferentialGamesBase as `src/problems/SGNEP.jl` and `src/solvers/SFNELQ.jl`

**Key result worth knowing:** under purely **additive** Gaussian noise the
feedback gains are *identical* to the deterministic ones (certainty
equivalence) — noise only raises the cost by `Σ ½tr(P·W)`. Different gains
require **multiplicative** uncertainty.

---

## 5. The open problem

**Both environments plateau at ~5,000–6,000 episodes out of 25,000.**

Verified statistically, not by eye:
`julia --project=. scripts/analyse_run.jl predprey_eval_history_sparse_team.csv 6000`
gives slope **t = +0.11**, total change **+0.078** over 19,000 episodes. Flat.

**Already tested and ruled out — please don't repeat these:**

1. **Critic divergence.** Mean |Q| ends at 2.459; the value implied by actual
   returns (0.287/step, γ=0.90, discount sum 9.28) is ≈2.66. The critic is
   valuing the policy correctly.
2. **Exploration collapse.** The noise schedule was changed
   (`noise_decay` 0.9995→0.9999, floor 0.01→0.02) so exploration stays alive
   across the whole run — 0.12 at episode 5,000 instead of 0.016. **The
   plateau did not move.**

**Remaining candidates:** a genuine adversarial equilibrium (in a two-sided
game a flat curve may be the correct answer), or something structural —
γ = 0.90 gives ~10 steps of credit horizon, possibly too short to reward a
manoeuvre that produces contact 15 steps later. A clean test would be to
freeze the prey's policy and train the predators alone: substantial
improvement rules out equilibrium.

---

## 6. Hyperparameters that were changed, and the honest caveat

γ 0.95→0.90, `lr_actor` 1e-3→5e-4, `update_every` 1→2, `noise_decay`
0.9995→0.9999, `noise_floor` 0.01→0.02, `grad_clip` 1.0 added,
episodes→25,000. Also added: greedy evaluation episodes, best-policy
checkpointing (off for predator–prey, where a high summed score just means one
side is winning), per-agent histories.

**These were changed to fix noisy/early-plateauing curves, and their effect is
confounded with the simultaneous jump from 1,500 to 25,000 episodes. The noise
change was subsequently shown not to affect the plateau.** I'd value a
skeptical review of whether these helped at all — please compare against the
older code I'm attaching and tell me honestly if any should be reverted.

---

## 7. How I'd like you to work

The previous session's main failure mode, which I'd like to avoid repeating:
**hypotheses were implemented as fixes before being tested as hypotheses.**
Three explanations for the plateau were proposed, two were built, and none was
the cause — each costing hours of training time.

So please:

- **Verify before claiming.** Numerical cross-checks in Python/NumPy caught
  real bugs before the Julia was written; that worked well. Do the same.
- **Don't read trends off plots.** Use `scripts/analyse_run.jl`. I twice had
  to correct claims that a flat curve was "still improving".
- **Say "I don't know"** when that's the truth, rather than producing another
  plausible fix.
- Prefer small, testable increments; run the test suite after touching `src/`
  (Julia caches modules, so restart the REPL after editing `src/`).

---

## 8. What I'd like next

1. A skeptical comparison of the current code against the older version I'm
   attaching — did the changes help, and should anything be reverted?
2. Your view on the plateau: genuine equilibrium, or worth chasing?
3. Longer term: benchmark MADDPG against the SLQ solver. A stochastic LQ game
   has a known closed-form Nash equilibrium, so it is the one setting where I
   can measure how far the learned policy is from optimal, rather than just
   "better than random".
