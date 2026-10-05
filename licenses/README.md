# Third-party licensing

This repository's own code (the scripts, training configs and documentation) is
MIT licensed — see the `LICENSE` file at the repository root.

The environment those scripts run inside, however, builds on the
[Unity ML-Agents Toolkit](https://github.com/Unity-Technologies/ml-agents),
specifically the **Walker** humanoid and **SoccerTwos** example environments.
Those assets — the humanoid prefab, the soccer field, and the base agent
scaffolding — are licensed under the **Apache License 2.0**. Our observation
spaces, reward functions, training architecture and curriculum were rebuilt on
top of them.

Carried here as the Apache-2.0 terms require:

- `ML-Agents-LICENSE.md` — the Toolkit's licence, holding Unity Technologies'
  copyright notice
- `THIRD-PARTY-NOTICES.md` — the Toolkit's third-party notices, covering bundled
  dependencies (System.Buffers, and others) under their own terms

The full licence text is available at
<http://www.apache.org/licenses/LICENSE-2.0>.
