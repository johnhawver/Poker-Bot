# Building the Python poker engine

Nov 1, 2026 · @Bobby

Stage 1 of the Poker Bot plan: heads-up no-limit Hold'em rules, a fast hand evaluator that never makes a mistake, and an arena that reports bb/100 with 95% error bars. The design below is built so the CFR work on Kuhn, Leduc and abstracted Hold'em later plugs into the same interfaces.

## The short version

1. Build in this order: cards → evaluator → rules engine → bots → arena → stats. Each piece is testable on its own before the next starts.
2. Write your own evaluator and engine (they are the deliverable), but use **PokerKit** as a test oracle: feed both the same random hands and action sequences and require identical results.
3. Keep two layers apart: an exact **rules engine** (any legal bet size, used by the arena and the competition adapter) and **abstract games** (a few discrete actions, used by CFR). Both implement one shared `Game`/`State` interface modeled on OpenSpiel's.
4. Make the arena duplicate from day one: same seeded deck, seats swapped, and compute the error bar over hand *pairs*.

## Architecture

| Layer | Owns | Used by |
| --- | --- | --- |
| `cards` | Card encoding (int 0–51 = rank\*4 + suit), deck, parsing "As Kh" | everything |
| `evaluator` | 5/6/7-card hand strength as one integer (higher wins) | engine, equity calcs, bucketing later |
| `engine` | Exact HUNL rules: blinds, streets, min-raise, all-in, showdown, payouts | arena, competition adapter |
| `games` | Shared `Game`/`State` protocol; Kuhn, Leduc, abstract Hold'em | CFR solvers, exploitability |
| `bots` | `Bot` interface + baselines (random, always-call, always-pot, equity threshold) | arena |
| `arena` | Runs matches, duplicate pairing, seeding, multiprocessing | stats |
| `stats` | bb/100, 95% CI, per-street breakdowns | reports, write-up |

### Proposed repo layout

```
pokerbot/
  pokerbot/
    cards.py          # encoding, deck, string parsing
    evaluator.py      # hand ranking, uses tables built by tools/build_tables.py
    engine.py         # HUNL State: legal_actions, apply, payoffs
    games/
      base.py         # Game / State protocol (shared with CFR)
      kuhn.py
      leduc.py
      holdem_abstract.py
    bots/
      base.py         # Bot.act(observation) -> Action
      baselines.py
    arena/
      match.py        # play one hand / one match
      duplicate.py    # seeded decks, seat swapping
      stats.py        # bb/100 and confidence intervals
  tools/build_tables.py
  tests/
    test_evaluator.py # exhaustive 5-card + 7-card checks
    test_engine.py    # scripted hand histories + property tests
    test_vs_pokerkit.py
    test_arena.py
```

Tooling: Python 3.12, `pytest`, `hypothesis` (property tests), `numpy`, `ruff`, GitHub Actions running the fast tests on every push.

## Step 1: Cards and the hand evaluator

**Representation.** Store a card as an int 0–51 and a hand as a 52-bit mask. Ints make tables and C++ porting easy later; masks make "is this card used" checks O(1).

**Recommended evaluator design (fast in pure Python, easy to port to C++):**

- Keep four 13-bit suit masks and a rank-count array for the 7 cards.
- Flush: any suit mask with popcount ≥ 5. Look up that mask in a precomputed 8192-entry table for best flush / straight flush.
- Straight: OR the suit masks into one rank mask, look it up in an 8192-entry straight table (remember the wheel A-2-3-4-5).
- Otherwise rank by counts (quads, full house, trips, two pair, pair, high card) and pack category + kickers into one integer.
- Avoid "best 5 of 7 by trying all 21 combinations": it is 21× slower and you will feel it in the arena.

Alternative if you want raw speed later: a perfect-hash evaluator (see `phevaluator`) or a 7-card lookup table. Start simple and correct; swap for speed once tests lock behavior.

**Tests that prove it never makes a mistake.** Enumerate every hand and compare category counts with the known totals:

| Category | 5-card hands | 7-card hands |
| --- | --: | --: |
| Straight flush (incl. royal) | 40 | 41,584 |
| Four of a kind | 624 | 224,848 |
| Full house | 3,744 | 3,473,184 |
| Flush | 5,108 | 4,047,644 |
| Straight | 10,200 | 6,180,020 |
| Three of a kind | 54,912 | 6,461,620 |
| Two pair | 123,552 | 31,433,400 |
| One pair | 1,098,240 | 58,627,800 |
| High card | 1,302,540 | 23,294,460 |
| **Total** | **2,598,960** | **133,784,560** |

Also check there are exactly **7,462** distinct 5-card strength values, and that ordering matches PokerKit or `treys` on several million random 7-card head-to-head pairs. The full 5-card run fits in CI; run the 133M 7-card enumeration as a one-off (multiprocessing or numpy) and record the result.

## Step 2: The rules engine

Model a hand as a state machine with these methods, mirroring OpenSpiel so the CFR code later sees the same shape:

```python
class State(Protocol):
    def current_player(self) -> int: ...        # 0, 1, CHANCE, or TERMINAL
    def legal_actions(self) -> list[Action]: ... # fold, check/call, bet/raise(amount)
    def apply(self, action: Action) -> "State": ...
    def is_terminal(self) -> bool: ...
    def returns(self) -> tuple[float, float]: ... # chips won/lost per player
    def chance_outcomes(self) -> list[tuple[Action, float]]: ...
    def information_state_key(self, player: int) -> str: ...  # what this player can see
    def observation(self, player: int) -> Observation: ...     # what a Bot receives
```

Heads-up rules people commonly get wrong, so write a test for each:

- The button posts the small blind and **acts first preflop, last on every later street**.
- Minimum raise: the raise increment must be at least the previous bet or raise increment on that street (minimum opening bet is 1 bb).
- An all-in that is smaller than a full raise does **not** reopen betting for a player who has already acted.
- Return the uncalled part of a bet; split pots on ties (decide an odd-chip rule).
- If both players are all in, deal the remaining board with no more betting.
- Reset stacks every hand (ACPC used 200 bb; match whatever the Citadel rules say once you have them).

**Testing the engine.**

- *Scripted hand histories:* 20–30 hand-written hands with exact expected pots and payoffs, including every edge case above.
- *Property tests (hypothesis):* random legal action sequences must conserve chips, never give a player a turn out of order, and always terminate.
- *Differential test vs PokerKit:* play the same deck and actions through both engines and assert identical legal actions and payoffs. This catches more bugs than anything else.

Speed: keep `apply` cheap (small dataclass with slots, or apply/undo on a mutable state). Plan for roughly 10k–50k hands per second per core with simple bots; that is plenty for the arena, and the C++ hot loop comes later for MCCFR only.

## Step 3: Bots and the arena

**Bot API.** `Bot.act(observation) -> Action`, where the observation contains only what that seat can see (own hole cards, board, betting history, stacks). Never hand bots the `State` itself, which prevents accidental cheating. Add `new_hand()` and `hand_over(result)` hooks for bots that track opponents.

**Baselines to beat:** random legal action, always-call, always-raise-pot, and a simple equity-threshold bot (Monte Carlo equity vs a random hand, raise above X, call above pot odds).

**Duplicate matches.** For each seed: deal hand with A on the button, then replay the exact same cards with seats swapped. Score each pair as A's total over both hands.

**Win rate and error bar.**

```
for each duplicate pair i: x_i = A's chips won over both hands / big blind
bb/100  = 100 * mean(x_i) / 2
95% CI  = bb/100 ± 1.96 * 100 * stdev(x_i) / (2 * sqrt(n_pairs))
```

Use pairs, not individual hands, as the independent samples. Run seeds across processes with `multiprocessing` and merge. Sanity checks: a bot against itself must give a CI containing 0, and always-call should lose heavily to the equity bot. If you later want tighter bars with fewer hands, read the AIVAT paper below.

## Build order and done checks

| Day | Build | Done when |
| --- | --- | --- |
| 1–2 | `cards`, `evaluator`, table builder | Exhaustive 5-card counts and 7,462 classes pass; random 7-card matches PokerKit |
| 3–4 | `engine` | Scripted hands, property tests and PokerKit differential tests all pass |
| 5 | `bots`, `arena`, `stats` | Self-play CI contains 0; equity bot beats always-call with a CI clear of 0 |
| In parallel | `games/base.py` + Kuhn on the same protocol | Research track can start CFR without waiting for Hold'em |

Agree on `games/base.py` on day 1: it is the handoff between the software track and the research track.

## Resources

### Libraries (reference, oracle, or dependency)

| Resource | Use it for |
| --- | --- |
| [PokerKit](https://github.com/uoftcprg/pokerkit) (`pip install pokerkit`) | Test oracle for rules and hand ranking; very complete NLHE state machine. Its paper (Kim, 2023) is a good read on engine design. |
| [OpenSpiel](https://github.com/google-deepmind/open_spiel) | Model for the `Game`/`State` API; reference Kuhn, Leduc and `universal_poker`; Python CFR, MCCFR and exploitability to check your results against. |
| [RLCard](https://github.com/datamllab/rlcard) | Simple, readable Leduc and Hold'em environments plus a CFR example. |
| [treys](https://github.com/ihendley/treys) | Readable pure-Python Cactus Kev evaluator; second oracle for hand ranking. |
| [PokerHandEvaluator / phevaluator](https://github.com/HenryRLee/PokerHandEvaluator) | Fast perfect-hash evaluator in C/C++ with Python bindings; good model when you move to C++. |
| [OMPEval](https://github.com/zekyll/OMPEval) | Very fast C++ evaluator and equity calculator; benchmark target for stage 5. |
| [MIT Pokerbots on GitHub](https://github.com/mitpokerbots) | Their Python engine and bot skeletons show what a competition engine and bot API look like. |

### Hand evaluator write-ups

- Cactus Kev's Poker Hand Evaluator (Kevin Suffecool): the classic prime-product and lookup design behind `treys`.
- "Poker hand evaluator roundup" (Coding the Wheel): compares Cactus Kev, Two Plus Two 7-card table, and others.
- PokerHandEvaluator's `Documentation/` folder: explains the perfect-hash algorithm step by step.

### CFR and game theory papers

| Paper | Why |
| --- | --- |
| Neller & Lanctot (2013), *An Introduction to Counterfactual Regret Minimization* | Start here. Kuhn CFR with full code; everyone should read sections 1–4. |
| Zinkevich et al. (2007), *Regret Minimization in Games with Incomplete Information* | The original CFR paper. |
| Lanctot et al. (2009), *Monte Carlo Sampling for Regret Minimization in Extensive Games* | MCCFR (external and outcome sampling) for Leduc and Hold'em. |
| Tammelin (2014), *Solving Large Imperfect Information Games Using CFR+*; Brown & Sandholm (2019), *Discounted Regret Minimization* | CFR+, linear and discounted CFR: much faster convergence for little code. |
| Bowling et al. (2015), *Heads-up Limit Hold'em Poker Is Solved* (Science) | How Cepheus scaled CFR+. |
| Brown & Sandholm (2018), *Superhuman AI for heads-up no-limit poker: Libratus* (Science) | The blueprint + abstraction approach you are copying in stage 5. |
| Moravčík et al. (2017), *DeepStack* (Science) | Alternative approach; good background. |
| Johanson et al. (2013), *Evaluating State-Space Abstractions in Extensive-Form Games*; Ganzfried & Sandholm (2014), *Potential-Aware Imperfect-Recall Abstraction with Earth Mover's Distance* | Card bucketing for stage 5. |
| Ganzfried & Sandholm (2013), *Action Translation in Extensive-Form Games with Large Action Spaces* | Mapping real bet sizes onto your few abstract sizes. |
| Burch et al. (2018), *AIVAT* | Variance reduction for tighter bb/100 error bars. |

### Courses and rules references

- CMU 15-888 *Computational Game Solving* (Farina & Sandholm): lecture notes on regret minimization, CFR variants and abstraction.
- Annual Computer Poker Competition (ACPC) rules: the standard HUNL setup (stack sizes, duplicate scoring) researchers benchmark on.
- Kuhn equilibrium for your stage 3 check: first player's value is −1/18; first player bets a Jack with probability α ∈ \[0, 1/3\], a King with 3α, and calls with a Queen with α + 1/3.

## Open questions

- **Citadel competition rules:** stack depth, blinds, time per action, language/API and submission format. Mirror their spec in the engine and bot API as soon as we have it.
- Whether to keep the arena pure Python or call the C++ evaluator through pybind11 once stage 5 starts (recommended: keep Python until profiling says otherwise).
