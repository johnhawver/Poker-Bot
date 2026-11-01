# Reading list

## Libratus

- **[Libratus: the world's best poker player](https://thegradient.pub/libratus-poker/)**
  Jiren Zhu, *The Gradient*, 2018-06-29

  Why it matters for us: the same pipeline we're building, at full scale.
  - **Abstraction**: group similar hands and bet sizes so the game fits in memory → our abstraction layer
  - **Blueprint**: solve the abstracted game offline with CFR → our strategy file
  - **Subgame solving**: re-solve the current spot in more detail during play (stretch goal, not in our 5-week plan)
  - **Self-improvement**: fill in the off-tree bet sizes opponents actually use → idea for after we submit

  Paper: Brown & Sandholm, "Superhuman AI for heads-up no-limit poker: Libratus beats top professionals," *Science* 2018.
