## 2. Inputs

An analysis begins with two inputs:

### 2.1 Physics Prompt

A brief natural-language description of the physics goal. Examples:

> Search for the Higgs boson produced in association with a Z boson, where the
> Higgs decays to a pair of b quarks, using ALEPH data at sqrt(s) = 200–209 GeV.

> Measure the inclusive W+W- production cross-section at LEP2 energies using
> fully hadronic final states.

The prompt need not specify methodology. It states the physics target and any
constraints (dataset, final state, energy range).

### 2.2 Experiment Context

The agent proceeds using its training knowledge and any documentation
provided in the analysis directory or CLAUDE.md. The agent should use
whatever sources are at hand to understand:

- Detector specifications and relevant physics context
- Standard object definitions and reconstruction techniques
- Known backgrounds and their characteristics
- Prior analyses in the same or related channels (for context, not copying)

The agent must document the basis for any experiment-specific claims it
uses (detector parameters, object selections, performance numbers).

**Verify against data.** Information from training knowledge may be
incomplete or wrong. The agent must verify claims against the data itself
where possible. The data is the ground truth. Discrepancies between
expected and observed behavior must be documented and resolved.

---
