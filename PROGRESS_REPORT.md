# MADDPG.jl — Progress Report

Alessandro Valente · September 2026

---

## 1. Status against requested items

| # | Request | Status |
|---|---|---|
| 1 | Train for 10,000 episodes at least | **Done** — now 25,000 as standard, for both environments |
| 2 | Use blocking barrier for leaving the arena | **Done** — hard wall by state projection; leaving is impossible by construction |
| 3 | Implement stochastic LQ solver; look at (S)GNEP | **Done** — `StochasticGameProblem` + `SFNELQ` solver |
| 4 | Add polygon obstacles to predator–prey | **Done** — convex polygons, blocking + optional barrier |
| 5 | Make a CWH game with Gaussian uncertainty | **Done** — three-satellite formation flying |
| 6 | Validate the SLQ solver | **Done** — four independent checks, all passing |
| 7 | Look at training progress | **Done** — see findings 5–7 below |

---

## 2. Code changes

### New files

| File | Purpose |
|---|---|
| `src/obstacles.jl` | Convex polygon obstacles: half-plane representation, signed clearance, blocking projection, barrier constraints, `min_width` |
| `src/predator_prey.jl` | Predator–prey environment (3 predators + 1 faster prey), three reward modes |
| `sgnep/SGNEP.jl` | `StochasticGameProblem` — the missing concrete `AbstractStochasticGame` in DifferentialGamesBase |
| `sgnep/SFNELQ.jl` | Stochastic feedback Nash LQ solver (coupled Riccati recursion) |
| `sgnep/cwh.jl` | Clohessy–Wiltshire–Hill dynamics + three-satellite formation game |
| `sgnep/validate_formation.jl` | Four-part validation of the solver |
| `sgnep/test_sgnep.jl` | Solver test suite |
| `scripts/analyse_run.jl` | Statistical trend analysis of learning curves (regression + t-test) |

### Modified

- **`src/environment.jl`** — added `state_projection` hook (applied after each integration step, before rewards) and `barrier_cap`.
- **`src/solver.jl`** — greedy evaluation episodes, per-agent reward histories, gradient clipping, best-policy checkpointing, critic-magnitude and exploration-noise logging.
- **`src/agent.jl`** — gradient clipping via `OptimiserChain(ClipGrad, Adam)`; greedy action selection now consumes no RNG draws, so evaluations do not perturb the training stream.
- **Scripts** — 500-episode statistical baselines, CSV histories saved for re-plotting without retraining, fixed plot margins, separate critic-diagnostic figures.

### Hyperparameters changed

γ 0.95→0.90, `lr_actor` 1e-3→5e-4, `update_every` 1→2, `noise_decay` 0.9995→0.9999, `noise_floor` 0.01→0.02, `grad_clip` 1.0 added, episodes →25,000.

> **Caveat, stated openly:** these were changed to address noisy/early-plateauing curves. Their effect is confounded with the simultaneous increase from 1,500 to 25,000 episodes, and the noise-schedule change was subsequently shown *not* to affect the plateau (finding 6). They are documented rather than claimed as improvements.

---

## 3. Findings

### 1. The stochastic LQ solver is validated on a real orbital problem
Three-satellite along-track formation flying on CWH dynamics with Gaussian process noise. Four checks, all passing:

- **Monte Carlo vs analytic:** analytic expected costs 40996.9 / 44429.1 / 44073.3 against 2,000-run simulation, **z = 0.01, 0.19, 0.42**. These come from disjoint code paths — a backward Riccati recursion and a forward noisy simulation — so agreement is meaningful evidence.
- **Best response** reproduces the equilibrium gains (max difference ~1e-12).
- **Single-player case** reduces exactly to textbook LQR.
- **Price of anarchy 1.0623×** — the satellites optimising separately burn ~6% more than a single controller would. Δv at equilibrium: 1.28 / 2.61 / 2.13 m/s, with the cheapest-fuel satellite doing the most work, as the cost asymmetry intends.

### 2. Certainty equivalence — worth knowing before designing experiments
Gaussian **process** noise leaves the feedback gains **bit-for-bit unchanged**; it raises the achieved cost by exactly `Σ ½tr(P·W)` and does nothing else. A more robust controller cannot be obtained by adding process noise — that requires **multiplicative** uncertainty (e.g. thruster gain error), which the solver also supports via its `D`/`sigma_control` channel. This is a property of the mathematics, not of the implementation.

### 3. Removing human guidance *improved* predator–prey
The headline learning result. With dense distance shaping (`:distance`), predators **never caught the prey**. With the paper's sparse tag objective and shared team credit (`:sparse_team`), they achieve **~27% of every episode in contact**.

Mechanism: the prey is 33% faster (accel 4.0 vs 3.0), so a straight-line chase can never succeed. Rewarding each predator for its *own* distance produces exactly that straight-line chase — the shaping was actively teaching the wrong strategy. Shared credit is the only structure under which hanging back to cut off an escape route pays.

### 4. Reward-shaping terms can silently dominate the objective
At full strength the obstacle barrier drove the prey's mean per-step reward from **+0.64 to +0.003** — the obstacle cost had consumed essentially the entire evasion signal, turning the game into obstacle avoidance. Obstacle barriers are now off by default, since the obstacles are impassable anyway and the barrier was pure shaping.

### 5. Both environments converge by ~5,000–6,000 episodes
Cooperative navigation plateaus around **−45** with **all three landmarks covered** (previously two). Predator–prey `:sparse_team` plateaus at **~6.7**.

### 6. The plateau is real, and verified statistically rather than by eye
Regression on the predator curve from episode 6,000: slope **+4.1e-6 per episode, t = +0.11**, implying a total change of **+0.078** across 19,000 episodes. Flat.

Two hypotheses for the plateau were tested and **rejected**:
- *Critic divergence* — refuted by the |Q| diagnostic (finding 7).
- *Exploration collapse* — the noise schedule was changed so exploration stays alive to 25,000 episodes (0.12 at episode 5,000 instead of 0.016). The plateau **did not move**.

The remaining candidates are a genuine adversarial equilibrium, or a structural cap such as γ = 0.90's ~10-step credit horizon being too short for a manoeuvre that produces contact 15 steps later. A clean test would be to freeze the prey's policy and train the predators alone: substantial improvement would rule out equilibrium.

### 7. The critic is healthy
Mean |Q| ends at **2.459**. Predicted from the actual returns: 7.18 per 25-step episode = 0.287/step, with γ = 0.90 giving a discount sum of 9.28, so ≈ **2.66**. The critic is valuing the policy essentially correctly; its slow rise was convergence, not divergence (which would show 10–100× returns).

### 8. Predators constrain the prey without catching it more
Predators flat (t = 0.11) while the prey degrades slightly (t = −2.34, −1.46 total). Since contact pays the predators, flat predators means contact frequency is *not* rising — the prey is losing ground to arena-barrier cost instead, i.e. being squeezed toward the walls.

### 9. A tunnelling defect, found and fixed
Blocking is a position test, not a swept test. Measuring per-step travel against obstacle width showed the prey moves **0.578 per step** while one obstacle was only **0.52** wide — it could pass straight through, and the impassability test would have passed anyway because the endpoint lands outside. Obstacles were resized (narrowest inflated width now 0.64) and `predator_prey` now **checks this at construction and warns**.

---

## 4. Open questions

1. **Where should `SFNELQ` live?** `SGNEP.jl` belongs in DifferentialGamesBase, but the base package currently contains no concrete solvers — FNELQ, FALCON and iLQGames are all external. Should the solver be its own package?
2. **Relationship to EAGLE/OSPREY.** The hierarchy comment marks `StochasticGameProblem` as "covariance steering". That constrains the terminal state *covariance*, which is a different problem from the risk-neutral expected-cost Nash equilibrium implemented here. If the type is meant to carry covariance-steering constraints it needs an extra field, and possibly a different name.
3. **Risk sensitivity.** Only the risk-neutral case is implemented; an exponential-of-quadratic (LEQG) variant would reuse this recursion.
4. **Is the predator–prey plateau an equilibrium?** See finding 6 — the frozen-opponent test would settle it.
5. **Parameters for the 25,000-episode runs you ran.** Your cooperative-navigation plots use a landmark ring at radius ≈0.9 (coincident with the agents' start ring) rather than 0.45, and a ±2 arena for predator–prey. Absolute rewards are therefore not comparable with mine; could you confirm the settings so both can be reported consistently?

---

## 5. Deliverables

**Figures:** `formation_flying.png`, `maddpg_trained.png`, `maddpg_predator_prey_sparse_team.png`, `predprey_critic_magnitude_sparse_team.png`, `random_policy.png`, `random_predator_prey.png`

**Data:** per-agent training and greedy-evaluation histories, random-policy baselines with standard deviations, critic magnitude and exploration level — all CSV, so figures can be re-made without retraining.

**Tests:** environment wrapper, constraint modes, obstacle geometry, impassability, tunnelling margin, reward-mode ordering, replay buffer, gradient updates, plus the SLQ suite (LQR reduction, moment propagation, certainty equivalence, Nash stationarity).
