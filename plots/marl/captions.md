# marl plots

source: MARL/maddpgpt1.

## current
maddpg_trained_randstart.png | cooperative navigation, MADDPG, 25000 episodes, randomised starts | use
coopnav_rollouts_randstart.png | twelve rollouts of the episode-25000 policy, worst and typical | use
maddpg_predator_prey_sparse_team_randstart.png | predator-prey, sparse team reward, shaped, seed 1 | use
maddpg_predator_prey_sparse_team_randstart_noshaping.png | same, barrier costs removed, seed 1 | use
maddpg_predator_prey_sparse_team_randstart_seed2.png | shaped, seed 2 | maybe
maddpg_predator_prey_sparse_team_randstart_noshaping_seed2.png | unshaped, seed 2 | maybe
shaping_replication.png | cross-play contact steps, shaped against unshaped, two seeds (27 Oct) | use
best_response.png | fine-tuning each side against a frozen opponent (27 Oct) | use
maddpg_vs_gps_julia.png | MADDPG against guided policy search on the Li et al. LQ game, Julia, 5 seeds (27 Oct) | use
maddpg_vs_gps_lq.png | earlier version of the same comparison (29 Sep) | skip, superseded
lq_guidance_stability.png | guidance weight against stability and bias in the LQ game (22 Oct) | use
appendixA_gradient_play_grid.png | gradient play grid over step size and guidance weight (27 Oct) | maybe
formation_flying.png | three-satellite formation, exact stochastic LQ solver (6 Sep) | use
mappo_predator_prey_sparse_team_randstart.png | MAPPO against the October MADDPG run, predator-prey (11 Nov) | use
mappo_trained_randstart.png | MAPPO cooperative navigation (11 Nov): training diverged to NaN, log 48 ends with NaN policy std, rollout panel empty | skip until rerun
mappo_predator_prey_sparse_team_randstart_noshaping.png | MAPPO predator-prey without shaping, against the October MADDPG run (11 Nov, final eval predators 7.68, prey -8.08) | use

## old
maddpg_trained.png, maddpg_trained_1500.png, maddpg_predator_prey.png, maddpg_predator_prey_sparse_team.png, random_policy.png, random_predator_prey.png | fixed-start or baseline runs | skip unless wanted for history
*_smoke.png, *_buf25000*.png, *_upd1*.png, *critic_magnitude*.png | diagnostics and ablations | skip

