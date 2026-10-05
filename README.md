# Humanoid 3v3 Soccer — Walker Soccer Twos

Teaching ragdoll humanoid agents to **walk**, then to **play 3v3 soccer**, in Unity ML-Agents.

A two-stage curriculum. Stage 1 trains stable bipedal locomotion with PPO and no ball at all.
Those weights then transfer into a multi-agent soccer environment trained with **MA-POCA** and
self-play. The agents learn to chase, kick, hold defensive shape and specialise into roles —
without any scripted behaviour telling them to.

Built as a team project for a course at McGill University (2025–26).

![Stage 1 locomotion](CustomWalkingSoccerTwos/Videos/Stage%201/Stage1_GoodClip_V1.gif)

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

| | Stage 1 — locomotion | Stage 2 — soccer |
|---|---|---|
| Algorithm | PPO (single-agent) | MA-POCA (multi-agent) |
| Steps | ~18M | continued from Stage 1 |
| Wall clock | ~12 hours | — |
| Learning rate | 3e-4, constant | — |
| Batch / buffer | 8192 / 81920 | — |
| Environment | 10–20 parallel arenas, moving targets | 3v3 with self-play |

Roughly 50+ hours of training across the two stages, including the failed runs. ELO was tracked
throughout to measure progress rather than relying on reward alone. Emergent behaviour observed
included role specialisation (one agent committing forward while another held back) and
defensive clearing.

**Honest status:** performance is imperfect. The agents walk and play recognisably, but the
policy is not a strong footballer and the ELO gain from Stage 2 is smaller than the gain from
Stage 1. The value of the project is the pipeline and the documentation of what did and did not
work.

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

**Not included, deliberately:** 51 of the 53 training checkpoints (they are intermediate saves;
one final model per stage is kept so the policy can be run without retraining), and the original
submitted report, which carries other people's names.

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
