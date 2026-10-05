# Humanoid 3v3 Soccer — Walker Soccer Twos

Teaching ragdoll humanoid agents to **walk**, then to **play 3v3 soccer**, in Unity ML-Agents.

A two-stage curriculum. Stage 1 trains stable bipedal locomotion with PPO and no ball at all.
Those weights then transfer into a multi-agent soccer environment trained with **MA-POCA** and
self-play. The agents learn to chase, kick, hold defensive shape and specialise into roles —
without any scripted behaviour telling them to.

Built as a team project for a course at McGill University (Nov 2025 – Dec 2025).

![Stage 1 locomotion](CustomWalkingSoccerTwos/Videos/Stage%201/Stage1_GoodClip_V1.gif)

▶ **[Watch the final 3v3 policy play (video)](https://drive.google.com/file/d/1bQIqSRJGe6BNBm9RDNiDwuzUt5F4Cvfo/view?usp=sharing)**

## The two design decisions that make it work

**1. Zero-padded observations, so the weights transfer cleanly.** The observation space is 269
floats in both stages. In Stage 1 the 26 soccer channels are present but filled with zeros. The
network's input dimension therefore never changes, and the Stage 1 locomotion weights load
straight into Stage 2 with no adapter, no projection layer and no retraining of the torso. That
single choice turns a fragile hand-off into a drop-in.

**2. Reward shaping plus curriculum self-play, to kill exploit loops.** Early runs found
degenerate solutions — agents that scored by exploiting a physics quirk rather than by playing
football. The reward structure was rebuilt around staged incentives and self-play opponents, so
the policy had to keep improving against a moving target instead of camping a loophole.

## The observation space (269 floats)

| Group | Dims | Contents |
|---|---|---|
| Locomotion core | 243 | 16 body-part positions ×3, linear velocities ×3, angular velocities ×3, 13 local rotations ×4, 13 joint-strength ratios, velocity-goal distance, average velocity, velocity goal, two 4-float rotation deltas, target position |
| Soccer context | 26 | Ball position ×3 and velocity ×3, team ID, role (striker / goalie / generic), own goal ×3, opponent goal ×3, two teammate positions ×3, two opponent positions ×3 |

Role and goal positions are what let distinct positional policies emerge without a behaviour tree.
Capping teammates and opponents at two each keeps spatial awareness without inflating the
dimension count.

## The action space (40 continuous)

| Group | Dims | Notes |
|---|---|---|
| Joint rotations | 26 | chest 3, spine 3, head 2, thighs 4, shins 2, feet 6, arms 4, forearms 2 |
| Joint strengths | 13 | Dynamic stiffness control across the major appendages |
| Kick intensity | 1 | Only enabled once ball contact exceeds a threshold |

Locomotion is torque-driven through `ConfigurableJoint` and a `JointDriveController`, with an
orientation cube supplying a stable reference frame to remove global drift.

## Training

Both stages were configured for long runs. These are the actual values from the training
configs in `Training Configs/`, not approximations.

| | Stage 1 — locomotion | Stage 2 — soccer |
|---|---|---|
| Algorithm | PPO (single-agent) | **MA-POCA** (multi-agent) |
| Configured steps | 30,000,000 | 20,000,000 |
| Learning rate | 3e-4, constant | 1e-4, constant |
| Batch / buffer | 8,192 / 81,920 | 8,192 / 81,920 |
| Discount (γ) | 0.99 | 0.995 |
| Entropy (β) | 0.005 | 0.005 |
| Clip (ε) / GAE (λ) / epochs | 0.2 / 0.95 / 3 | 0.2 / 0.95 / 3 |
| Network | 512 units, 3 layers, normalised observations | identical — which is what makes the transfer work |
| Environment | 10–20 parallel arenas, moving targets, no ball | 2v2 escalating to 3v3, dynamic team-size detection, self-play |

Two differences between the stages carry the design:

- **The learning rate drops 3× in Stage 2** (3e-4 → 1e-4). Stage 2 is fine-tuning transferred
  weights, not learning from scratch, and a full-size step would wreck the gait the agent had
  already learned.
- **γ rises from 0.99 to 0.995.** At 0.995 a reward roughly 150–200 steps away still carries more
  than half its weight, which is what lets a goal influence the pass that set it up.

Wall clock was about **12 hours for the first 18M locomotion steps**, and roughly **50+ hours
total** across both stages once the failed runs are counted.

### Self-play

Competitive training is schedule-driven rather than hand-started:

| Setting | Value |
|---|---|
| Snapshot interval | every 50,000 steps |
| Snapshots retained | 5 |
| Team swap | every 200,000 steps, graduated over 10,000 |
| Opponent mix | 50% latest model, 50% snapshot pool |
| Initial ELO | 1200 |

The opponent mix matters. Playing only the newest model lets it drift into a strategy that beats
itself; playing only old snapshots lets it stagnate against outdated opponents.

### Curriculum

Stage 2 does not begin as soccer. Four environment parameters ramp in lockstep:

| Parameter | Progression | Why |
|---|---|---|
| `ball_touch` | 0.35 → 0.5 → 1.0 | Chase the ball first, enable kicking second, full soccer last |
| `ball_spawn_radius` | 1.5 → 2.0 → 1.5 | Broad spatial search early, tighter spawns once scoring matters |
| `locomotion_scale` | 0.5 → 0.4 → 0.15 | Fade the inherited locomotion reward so soccer signals dominate |
| `self_play_weight` | 0.0 → 1.0 at 50% progress | Cooperative first, competitive once the basics exist |

Reward was concentrated on outcomes: **+50 × time_bonus** for scoring, **−10 × self_play_weight**
for conceding, and a **−0.04 × toward-speed** penalty on own-goal shots to stop accidental
backward clears.

### What went wrong, and what fixed it

The training log records nine phases, and the informative ones are the failures:

- **Locomotion reward dominance.** Early Stage 2 mean reward sat at 60–100 against an expected
  5–20, with **no goals in 530,000 steps** and ELO sliding 1052 → 676. The inherited locomotion
  reward was worth +80–100 per episode versus +0–2 from soccer, so the agents simply kept
  walking. Fixed by fading `locomotion_scale`.
- **Wall-sticking and corner piling.** Agents learned to trap the ball against geometry and farm
  reward without playing football. Fixed with touch cooldowns and stuck-ball detection.
- **A full training collapse**, recovered from by softening the penalty schedule.

**Honest status:** performance is imperfect. The agents walk, chase, kick and hold shape
recognisably, and role specialisation emerged without being scripted — but this is not a strong
football policy, and Stage 2's ELO gain is smaller than Stage 1's. The value is in the pipeline
and the record of what did and did not work.

## What is in this repository

The custom environment our team wrote. Everything here is the contents of one Unity asset folder,
so the layout is preserved exactly as Unity expects — dropping it in does not break the `.meta`
GUID references in the scenes and prefabs.

```
CustomWalkingSoccerTwos/
├── Scripts/              WalkerSoccerAgent.cs (~56 KB), env controller, ball controller, settings
├── Training Configs/     Stage 1 (PPO) and Stage 2 (MA-POCA) YAML configs — the real training recipes
├── Scenes/               Stage 1 and Stage 2 training + testing scenes
│   └── TFModels/         One trained checkpoint per stage (see note below)
├── Prefabs/              Ragdoll humanoid, platforms, soccer field variants
├── Materials/            Team agent materials
├── Videos/               Stage 1 behaviour clip
└── Docs/                 ~90 KB of written design record — see below
```

The `Docs/` folder is the part worth reading. It is a genuine engineering log rather than a
summary written afterwards:

- **`FINAL_PROJECT_OVERVIEW.md`** — full architecture, with the observation and action space
  broken down channel by channel and the reasoning behind each choice
- **`PROJECT_HISTORY.md`** — the training log, phase by phase, including what failed
- **`PPO_AND_POCA.md`** — how the two algorithms differ and why the credit-assignment problem
  forced the switch
- **`PRESENTATION_SUMMARY.md`**, **`CHANGELOG.md`** — results and iteration history

**Not included, deliberately:** 51 of the 53 training checkpoints — those are intermediate
saves. One final model per stage is kept so the policy can be run without retraining: Stage 1 at
step **30,000,173** and Stage 2 at step **20,000,114**. Also omitted is the originally submitted
report, which carries other people's names and student IDs.

## Running it

This is a component, not a standalone Unity project. To use it:

1. Create a Unity project with the **ML-Agents** package installed, matching the toolkit version
   the fork is based on.
2. Copy `CustomWalkingSoccerTwos/` into `Project/Assets/ML-Agents/` inside that project.
3. Open a scene from `Scenes/`, and either
   - press Play with the included model set in the Behaviour Parameters to watch the trained
     policy, or
   - train from scratch with `mlagents-learn "Training Configs/WalkerSoccerStage1_Locomotion.yaml" --run-id=stage1`,
     then continue into Stage 2 with the Stage 2 config.

## Licensing and attribution

Two different things are going on here, and they are worth separating:

- **Our code** — the scripts, configs and documentation in this repository — is MIT licensed. See
  [LICENSE](LICENSE).
- **The environment it runs inside** builds on Unity's [ML-Agents Toolkit](https://github.com/Unity-Technologies/ml-agents),
  specifically the **Walker** humanoid and **SoccerTwos** examples, which are Apache-2.0 licensed.
  The humanoid prefab, the soccer field assets and the base agent scaffolding originate there, and
  our observation spaces, reward functions and training architecture were rebuilt on top. Upstream's
  licence and third-party notices are carried in [`licenses/`](licenses/) as the Apache-2.0 terms
  require.

## Team

Built as a team project; the work was shared across the group.
