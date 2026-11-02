# Poker Bot · MFAMS Project I

A poker bot for the **Citadel Securities poker bot competition**. The plan is a heads-up no-limit Texas Hold'em bot that learns its strategy through self-play (counterfactual regret minimization, CFR). We validate it first on two small games, Kuhn and Leduc poker, then scale it up to abstracted Hold'em.

- **Timeline:** 5 weeks (up to 7)
- **Team:** 5–6 people, split into a Math track and a CS track
- **Languages:** Python, with C++ for the training hot loop

> **Status:** Week 1. We're pinning down the rules and building the infrastructure. See [Roadmap](#roadmap) for what's done.

---

## Contents

- [How the team is split](#how-the-team-is-split)
- [Pin down the rules](#pin-down-the-rules)
- [Roadmap](#roadmap)
- [Milestones](#milestones)
- [Repository layout](#repository-layout)
- [Getting started](#getting-started)
- [Contributing](#contributing)
- [Research and references](#research-and-references)

---

## How the team is split

### Math people: research
- Read up on **CFR, Monte Carlo CFR (MCCFR)** and the other ideas this project relies on: regret matching, exploitability, abstraction and equity.
- Write down a list of **questions, assertions and ideas** for the project. We'll discuss them together next week.
- Once we have a clearer plan, tell the CS people what we need so they can support it.
- If a paper helps you, add it to the [`Research/`](Research/) folder and add one line about it to [`docs/references.md`](docs/references.md). This helps both teammates and AI tools understand the project.

### CS people: infrastructure
- Build out the infrastructure. Plan which features need to be supported and analyze each one as much as possible.
- Get **Texas Hold'em** and **Leduc poker** working. This means the rules only, not the strategy.
- Get **Kuhn poker** working, with support for simple strategy inputs and basic calculations such as expected value and best response.

### Handoff
The **game interface** is where the two tracks meet. It's one shared API that lets the same CFR code run on Kuhn, Leduc and abstracted Hold'em. Agree on it early.

---

## Pin down the rules

Nothing else is final until these are confirmed. Fill in each answer and its source as we find out.

| Question | Answer | Source |
|---|---|---|
| Variant: heads-up or 6/8-handed; cash or tournament | TBD | |
| Blinds and stack sizes | TBD | |
| Per-move time limit | TBD | |
| Compute, memory and file-size limits | TBD | |
| Allowed languages | TBD | |
| Does state persist between games? | TBD | |
| How scoring works | TBD | |
| Server or API protocol | TBD | |
| Starter code provided? | TBD | |

---

## Roadmap

Tick items off as they're merged. Put your name next to anything you pick up.

### Core infrastructure
- [ ] Game state representation and legal action generation (side pots, min-raise rules, all-ins)
- [ ] A client that talks to the competition's engine or server
- [ ] Hand-history logging and replay for debugging
- [ ] A single entry point (state in, action out) with a safe fallback (fold or check) on error or timeout

### Hand evaluation and equity
- [ ] A fast 5- and 7-card hand evaluator using lookup tables
- [ ] Preflop equity tables
- [ ] Monte Carlo equity against random hands and against ranges
- [ ] Hand-strength bucketing, which clusters hands by equity distribution

### Strategy
- [ ] A heuristic baseline bot: preflop ranges by position, pot odds, bet sizing, and short-stack push/fold
- [ ] Game abstraction: card buckets, a small set of bet sizes, and a rule for mapping opponents' off-tree bets to the nearest one
- [ ] A CFR/MCCFR solver, validated first on Kuhn and Leduc poker with exact exploitability, then scaled to abstracted Hold'em
- [ ] Compact storage and fast lookup of the trained strategy
- [ ] Real-time subgame solving or search, if the time limit allows
- [ ] Tournament adjustments (ICM, stack-depth awareness), if the format is tournament play

### Opponent modeling
- [ ] In-session stats such as VPIP, PFR, aggression and fold-to-raise
- [ ] Detecting and exploiting simple leaks without becoming exploitable ourselves
- [ ] An opponent zoo for testing: calling station, maniac, nit, random

### Testing and evaluation
- [ ] A local simulator and tournament harness
- [ ] A self-play league between bot versions
- [ ] Enough hands to be meaningful, duplicate matches, variance reduction (AIVAT), and confidence intervals
- [ ] Benchmarks against our baseline bot and any publicly available bots

### Robustness and performance
- [ ] Latency profiling against the time limit
- [ ] Exception handling and illegal-action prevention
- [ ] Memory and file-size checks, fast deterministic startup, and no unnecessary I/O
- [ ] Long-run stress tests and edge cases (all-ins, split pots, weird bet sizes)

### Delivery
- [ ] Version control, CI, and reproducible training runs
- [ ] Packaging to the submission requirements, with testing on the exact artifact we submit
- [ ] A feature freeze date (**TBD**) and a known-good bot that is always deployable
- [ ] Documentation

---

## Milestones

| When | Milestone | How we check it |
|---|---|---|
| End of week 2 | Kuhn poker solved | CFR strategy matches the known equilibrium |
| End of week 3 | Leduc poker training | Exploitability keeps falling toward zero |
| End of week 5 | Hold'em bot submitted | Beats the baseline bots, with error bars (bb/100 ± 95% CI) |

---

## Repository layout

Proposed layout. Update it as the code takes shape.

```
.
├── README.md
├── Research/            # papers and notes (Math track)
├── docs/
│   └── references.md    # reading list: one line on why each source matters
├── engine/              # game rules: Kuhn, Leduc, Hold'em; shared game interface
├── eval/                # hand evaluator, equity, bucketing
├── solvers/             # CFR, MCCFR, exploitability
├── bots/                # baseline bot, opponent zoo, final bot entry point
├── arena/               # simulator, duplicate matches, stats harness
├── client/              # competition server or API adapter
└── tests/
```

---

## Getting started

> Fill this in once the first code lands.

```bash
git clone <repo-url>
cd <repo>
# setup steps TBD (Python version, dependencies, C++ build)
```

---

## Contributing

- Work on a **branch** and open a **pull request**; get at least one review before merging to `main`.
- Keep `main` deployable: there should always be a bot on it that can be submitted.
- Add tests with new game logic. The hand evaluator and legal-action generation must be exactly right.
- Write down decisions (rules, API, abstraction choices) in this README or under `docs/`.

---

## Research and references

- [`docs/references.md`](docs/references.md): annotated reading list
- [`Research/`](Research/): papers and notes

Starting points: Neller & Lanctot (2013), *An Introduction to Counterfactual Regret Minimization* · Zinkevich et al. (2007), *Regret Minimization in Games with Incomplete Information* · Brown & Sandholm (2018), *Libratus* · Tools: OpenSpiel, RLCard, PokerKit
