# Oroletto: interview prep

Claude Code transcript: interview prep for Oroletto, asked in both repos. Tool calls are listed by name; their output is left out.

# Backend: OthelloRL

## Turn 1

**You:**

> mhey . I have an interview and they might ask me about this project. give me most likely questiuons and technical ones

**Claude:**

Here are the questions you're most likely to get, with what to say for each, using this project's own facts.

## 1. The overview (almost certain)

**"Tell me about this project."**
Give a 60-second version:
- **Goal:** find out how far different RL methods can get at Othello, starting from just the rules.
- **Path:** classical methods on 6×6 → self-play methods → whether opponent variety helps (league, co-evolution) → scaling the winner to 8×8.
- **Result:** the AlphaZero agent, Oroletto, beats every hand-written bot. On 8×8 it plays between Edax levels 3 and 5, and it's deployed as a web game with four difficulty levels.

**"Why Othello?"**
- The game is small enough to train on one machine but too big to solve by brute force on 8×8.
- It has a clear win/lose signal.
- The position swings sharply, so greedy play (grabbing discs) loses. That makes it a good test of whether an agent learns long-term strategy.

**"Why start on 6×6?"**
- Iteration speed: experiments on 6×6 took hours, while one 8×8 seed took about 2.2 days.
- Use 6×6 to pick the method, then pay for 8×8 once.

## 2. RL fundamentals they'll probe

**"Explain the RL setup."**
- **State:** the board, encoded as planes for my discs, opponent discs and legal moves.
- **Action:** one of the squares, or pass.
- **Reward:** +1, 0 or −1 at the end of the game only. That's a sparse, delayed reward.
- **Key challenge:** credit assignment over about 30 moves per player.

**"Why is a sparse reward hard, and how did you deal with it?"**
- Value-based methods have to propagate the final result back through bootstrapping, which is slow and noisy.
- AlphaZero sidesteps this: MCTS gives a strong policy target every move, and the value net learns from the final outcome z.

**"On-policy vs off-policy?"**
- **DQN (off-policy):** a replay buffer, so it reuses old data.
- **PPO (on-policy):** needs fresh data, but its updates are more stable.
- Trade-off: sample efficiency vs stability.

**"What's Double DQN and why use it?"**
- Vanilla DQN takes the max over noisy Q estimates, which biases it upward.
- Double DQN picks the action with the online net and evaluates it with the target net.
- The project compared vanilla and Double DQN directly in the frozen-target experiment.

## 3. AlphaZero (the core, so expect depth here)

**"How does AlphaZero work?"**
- **Network:** a policy+value ResNet takes the board and outputs move probabilities p and a value v.
- **Search:** MCTS with PUCT. Each simulation picks the child that maximises Q + c·P·√N_parent / (1 + N_child), expands a leaf, evaluates it with the network (no rollout), and backs up v.
- **Move choice:** play the **most-visited** move, not the one with the highest Q.
- **Training targets:** the visit distribution π for the policy (cross-entropy) and the game result z for the value (MSE).

**"Why pick by visit count rather than max Q?"**
- Q is a mean of noisy backed-up values. A rarely visited move can have a lucky high Q.
- Visits reflect where the search kept going after averaging, so they're robust to a few noisy evaluations.
- The report has a worked example: a position where max-Q ≠ max-N at low budget, and the two agree as simulations increase.

**"Why does AlphaZero beat DQN and PPO here?"**
- Search acts as a policy-improvement operator: the network learns to imitate a stronger version of itself (π), every move.
- That's a dense, high-quality signal. DQN and PPO only get the sparse reward.

**"What's the role of temperature and Dirichlet noise?"**
- In self-play they add exploration, so the agent sees varied positions.
- In the app, temperature on the visit counts is the difficulty dial.

## 4. Experimental design (shows research maturity)

**"How did you evaluate?"**
- Fixed opening positions, played with both colours.
- Hundreds of games per pairing, reported with 95% confidence intervals.
- Multiple seeds.
- Rule-based bots as fixed yardsticks: random, a heuristic, and minimax at depths 2–4. On 8×8, Edax as an external yardstick.

**"Why openings and both colours?"**
- Deterministic agents would otherwise play the same game every time.
- First-move advantage would also bias the scores.

**"What was the frozen-target experiment?"**
- **Setup:** identical DQNs, each trained against one fixed opponent (random, heuristic or minimax-d3).
- **Finding:** once a learner reliably beats its trainer, games stop teaching it and progress stalls. Self-play keeps raising the bar. That's the motivation for self-play.

**"Did opponent variety help (league, co-evolution)?"**
- **League:** mixing in past opponents didn't beat pure self-play, and strength fell as the mixing fraction grew.
- **Co-evolution:** a population tied self-play at equal games per agent, and was worse at equal total compute. It showed no extra diversity and no strategy cycles.
- Negative results are still results. Be ready to say why: in a two-player zero-sum game, self-play already provides an adaptive curriculum.

## 5. Engineering and debugging (interviewers love these)

**"Hardest bug?"** Tell the **denormal floats** story:
- **Symptom:** training throughput dropped about 10× partway through a run.
- **Cause:** weight decay shrank weights into denormal (subnormal) float range, where the CPU falls back to very slow arithmetic.
- **Diagnosis:** profiling, and noticing the slowdown tracked training time, not data.
- **Fix:** flush denormals to zero and clamp tiny weights.
- **Lesson:** monitor throughput, not just loss.

**"How did you deploy it?"**
- Exported the network to ONNX and checked parity with PyTorch (max difference about 2.5e-6).
- Wrote a manifest with a SHA-256 hash and golden fixtures for parity tests in the app.

**"How do you make difficulty levels from one model?"**
- Same search budget for every level (200 simulations).
- Sample moves with probability ∝ N^(1/T): Easy T=8, Medium T=2, Hard T=1, Master T=0.
- Calibrated so each level beats the one below at least 70% of the time. The measured scores were 76%, 86% and 86%.
- Why not fewer simulations? Temperature degrades play more smoothly and keeps the moves human-plausible.

**"How did you scale from 6×6 to 8×8?"**
- Only the config changed: 6×64 ResNet (vs 4×64), 150 simulations (vs 64), 100k self-play games per seed.
- About 52 hours per seed on CPU.

## 6. Critical-thinking questions

**"What are the limitations?"**
- CPU compute: a small network and few seeds on 8×8.
- Edax ran without its opening book.
- Cross-paper comparisons to OLIVAW are indirect.
- Nothing was distilled for speed.

**"How would you make it stronger?"**
- A bigger network and more simulations on a GPU.
- Use explored-but-not-played positions as extra training data, as OLIVAW did.
- Symmetry augmentation (8 board symmetries).
- An endgame solver for the last ~12 empty squares.
- Have Oroletto play Edax/Egaroucid itself rather than only being measured against it.

**"How does it compare to top engines like Edax or Egaroucid?"**
- They use pattern-based evaluations (n-tuples) fitted on huge position databases, plus very fast alpha-beta and exact endgame solving.
- Oroletto learns everything from self-play, with no human data, but searches far fewer nodes per second.

**"If you did it again?"**
- Use a GPU earlier, monitor throughput from day one, and decide evaluation protocols before training.

## 7. Quick-fire technical questions
- **PUCT vs UCB1:** PUCT adds the policy prior P, so search focuses on moves the network likes.
- **Why a ResNet:** skip connections keep gradients flowing in deeper nets, and convolutions share weights across board locations.
- **Value target z vs bootstrapped:** z is unbiased but high-variance. Bootstrapping is the reverse.
- **Overfitting in self-play:** watch for strategy collapse. Evaluate against fixed external bots, not just previous versions.
- **Exploration vs exploitation:** ε-greedy (DQN), entropy bonus (PPO), Dirichlet noise and temperature (AlphaZero).
- **What's a confidence interval on a win rate:** the Wilson or normal approximation, √(p(1−p)/n).

**Tip:** have three concrete numbers memorised: beats minimax-d4 about 97% on 8×8, sits between Edax levels 3 and 5, and the denormal bug was about 10× slower. Concrete numbers make answers credible.

I can turn this into a one-page cheat sheet, or run a mock interview where I ask the questions and you answer.


---

## Turn 2

**You:**

> Basically explain how alpha zero works Explain MCTS, used in Alpha zero how does it side step the sparse reward ?
> Exxplain on policy vs off policy better. 
> explain MCTS and PUCT. 
> Where does the model store how many times it has made a particular move ?

**Claude:**

`[Bash: Find MCTS node storage in code]`

`[Bash: Locate MCTS node fields in AlphaZero code]`

`[Bash: Inspect MCTS module structure]`

`[Bash: Read Node, select, backup, policy code]`

## 1. AlphaZero in one picture

AlphaZero has two parts that improve each other:

- **A neural network** f(s) → (p, v). Given a board s, it outputs:
  - **p**: a prior over moves ("which moves look promising");
  - **v**: a value in [−1, +1] ("who is winning from here").
- **A search (MCTS)** that uses the network to think ahead a few hundred positions, then outputs a better move distribution **π** than p alone.

The training loop:

```
repeat:
  1. Self-play: for each move, run MCTS from the current board -> π
     play a move sampled from π, continue until the game ends -> result z (+1/0/-1)
  2. Store (s, π, z) for every position of the game
  3. Train the network:  p -> π  (cross-entropy),   v -> z  (MSE)
```

The idea that makes it work: **search is a policy-improvement operator.** MCTS with the network plays better than the network alone. The network is then trained to imitate the search. Next iteration, the search starts from a better network, so it's stronger still. The agent bootstraps itself upward.

## 2. MCTS, step by step (with PUCT)

The search builds a tree. Each node is a board position and each edge is a move. Every edge (s, a) stores four numbers:

| Field | Meaning |
|---|---|
| **N(s,a)** | how many simulations went through this move |
| **W(s,a)** | sum of values backed up through it |
| **Q(s,a)** = W/N | average value of this move so far |
| **P(s,a)** | the network's prior for this move |

One **simulation** has four phases:

1. **Select.** Start at the root and walk down the tree. At each node, pick the move that maximises the PUCT score:

   **U(s,a) = Q(s,a) + c_puct · P(s,a) · √N(s) / (1 + N(s,a))**

   - **Q** is the exploitation term: "this move has worked well so far".
   - The second term is exploration:
     - it is large when the network likes the move (P high);
     - it is large when the move has been tried rarely (N(s,a) small);
     - it shrinks as the move gets visited.
   - c_puct sets the balance between the two. Ours is 1.5.

2. **Expand.** Stop at a leaf, i.e. a position not yet in the tree. Call the network once to get p (the priors for its children) and v.

3. **Evaluate.** Use **v** as the leaf's value. There is **no random rollout to the end of the game**, which classic MCTS used. The value net replaces it.

4. **Backup.** Walk back up the path. On every edge, add 1 to N and add v to W. Flip the sign at each level, because a position good for me is bad for my opponent.

After, say, 150 simulations:

- **To play:** pick the root move with the **highest N**. Visit count is robust because Q can be inflated by a few lucky evaluations, while a high N means the search kept coming back to that move.
- **To train:** π = N(root, a) / Σ N, the normalised visit counts.

**PUCT vs plain UCT/UCB1:** UCB1 explores all moves evenly at first. PUCT weights exploration by the network's prior P, so a few hundred simulations go to the 2–3 sensible moves instead of all ~10.

## 3. How does this sidestep the sparse reward?

The problem: in Othello the only reward is +1/0/−1 at the end, about 60 moves later. A method like DQN has to push that single number backward one step at a time through bootstrapping (Q(s) ← r + γ max Q(s′)). That is slow, noisy, and it's hard to tell which of the 30 moves deserved credit.

AlphaZero gets around it in three ways:

1. **A dense policy target on every move.** Every position gets a full target distribution π from the search, which says "these are the good moves here". Most of the learning happens through this supervised-style signal, not through the reward. It's like having a teacher annotate every move.
2. **The value net turns the sparse reward into a dense estimate.** v is trained directly on the final outcome z: plain regression with no bootstrapping chain. MCTS then uses v at every leaf, so the search gets an immediate "who is winning" estimate without waiting for the game to end.
3. **Lookahead does the credit assignment.** The search explicitly simulates what follows a move, so "this move leads to positions my value net thinks are lost" is discovered within a single move's search, not over thousands of games.

In short: **the reward only has to teach the value net. The search turns the value net into a rich per-move target, and that is what trains the policy.**

## 4. On-policy vs off-policy, more precisely

Two policies matter:

- **Behaviour policy:** the one that collected the data (made the moves).
- **Target policy:** the one you're trying to learn or evaluate.

**On-policy (PPO, SARSA, REINFORCE).** The two are the same. The update formula is only mathematically valid for data generated by the current policy.

- After one update the policy changes, so old data is stale and must be thrown away.
- Pros: stable and theoretically clean.
- Cons: sample-hungry.
- PPO stretches this a little. It reuses a batch for a few epochs, and its clipping keeps the new policy close to the one that collected the data.

**Off-policy (DQN, Q-learning).** The two can differ. Q-learning's target r + γ maxₐ Q(s′, a) asks "what is the value if I act *greedily* from here?", whoever actually made the move.

- So you can store every transition in a **replay buffer** and reuse it many times, even data from old policies or other agents.
- Pros: sample-efficient.
- Cons: can be unstable. The "deadly triad" is function approximation + bootstrapping + off-policy data.

**Analogy:** on-policy is learning only from your own current games. Off-policy is also studying recordings of old games, including other people's.

**Where AlphaZero sits.** Its replay buffer holds the last few iterations of self-play, so the data is slightly off-policy. But its targets (π from search, z from the game) aren't derived through a max-based bootstrap, so the stale data does little harm. It behaves close to on-policy in practice.

**In this project:**
- **Off-policy:** DQN, which is why the league experiment could mix in games against past opponents through the buffer.
- **On-policy:** PPO.

## 5. Where are the visit counts stored?

**Not in the network.** They live in the **temporary search tree**, which exists only while choosing one move.

In our code (`othello/rl/alphazero/mcts.py`):

- **Storage.** Each `Node` (line 19) holds parallel arrays, one entry per legal move: `node.P`, `node.N`, `node.W`, plus `sumN` (the total visits at that node). They're created in `_expand` (lines 70–72).
- **Updates.** `_backup` (line 132) does `node.N[i] += 1`, `node.W[i] += v` along the path.
- **Reading.** `_select` (lines 76–85) reads N, W and P to compute the PUCT score. `policy` (line 139) turns `root.N` into π. `choose` (line 147) picks the move, using the most-visited when T = 0.
- **Lifetime.** A fresh root `Node` is built for every move (in `selfplay.py`), and the tree is thrown away after the move. We don't reuse the tree between moves.

So the counts are a scratchpad. What persists is their **distilled lesson**: π is saved with the position in the replay buffer, and training pushes the network's p toward π. Over time **the network's weights absorb "which moves the search tends to visit"**, so later searches start with better priors. In other words:

> N is short-term memory, inside one search. The network weights are long-term memory, learned from many searches' N.

This distinction is a good interview point: tabular methods store per-state counts permanently, but AlphaZero can't, because there are far too many positions. It generalises through the network instead.


---

## Turn 3

**You:**

> which is updated every move and which is updated less policy/target ?

**Claude:**

`[Bash: Check DQN target-net and AZ update frequencies]`

It depends whether you mean DQN's two networks or the two policies from the on/off-policy definition. The names overlap, so interviewers often mix them up.

## DQN: online network vs target network

DQN keeps two copies of the same Q-network:

| | **Online network** (often called the "policy net") | **Target network** |
|---|---|---|
| Updated | **Often**, by gradient descent | **Rarely**, by copying the online weights |
| In our code | one training step every **4 moves** (`train_every=4`) | synced every **2,000 training steps** (`target_sync=2_000`), i.e. about every 8,000 moves |
| Job | chooses moves (ε-greedy) and is the network being trained | computes the TD target: r + γ · max Q_target(s′, a′) |

Code: `othello/rl/dqn.py:66` (settings), `:105` (train every 4 steps), `:127` (sync).

**Why update the target net rarely?** The loss is (Q_online(s,a) − target)². If the target came from the same network you're updating, every gradient step would also move the target. That's like chasing your own shadow, and it tends to oscillate or diverge. Freezing the target for a few thousand steps makes each stretch of training a stable regression problem. Then you copy the weights over and repeat.

**Double DQN** uses both networks in the target:
- the **online** net picks the best next action: a* = argmax Q_online(s′, ·);
- the **target** net scores it: Q_target(s′, a*).

Splitting selection from evaluation removes the overestimation bias of taking a max over noisy estimates.

## The on/off-policy terms: behaviour policy vs target policy

These two aren't updated on different schedules.

- **Behaviour policy:** the one making moves. In DQN it's ε-greedy on the *online* net, so it changes whenever the online net changes (every 4 moves).
- **Target policy:** the one being learned. In DQN it's the greedy policy (the max in the target).

This pair is what makes DQN **off-policy**: it acts ε-greedily but learns about greedy play. That's unrelated to the "target network", which is only a stability trick.

## AlphaZero, for comparison

AlphaZero has no target network.

- **Every move:** a fresh MCTS tree is built, its counts N and W are filled in, and then it's discarded.
- **Rarely:** the network is trained once per iteration, after **256 self-play games** (`games_per_iter: 256`). All games in an iteration use the same frozen weights.

**One-line answer for the interview:** "In DQN, the online network is trained every few steps and picks the moves. The target network is a frozen copy synced every few thousand steps, so the TD target doesn't move while we regress toward it."


---

## Turn 4

**You:**

> Does Alpha zero predict p to indicate which move has the best likelihood to result in a win ?

**Claude:**

No, not quite, and the difference is a good thing to get right in an interview.

**p is "which move would my search pick"**, not "how likely is this move to win". It's trained to imitate the MCTS visit distribution π. A high p(a) means "a deeper search tends to focus on move a". That correlates with good moves, but it isn't a win probability:

- **p sums to 1 across moves; win chances don't.** If two moves both win 90% of the time, p might split 50/50 between them. Each gets 0.5, nowhere near 0.9.
- **p is relative.** In a lost position the best move might still get p = 0.95, because it's the least bad, even though it rarely wins.
- **p carries some exploration.** π comes from visit counts, which include visits made for exploration (and Dirichlet noise in self-play). So p is a "where to look" prior, not a verdict.

**The win likelihood is v, the value head:**
- v(s) ∈ [−1, +1] is the *expected game outcome* from position s for the player to move.
- It's trained on the actual result z.
- Ignoring draws, (v + 1)/2 ≈ the probability of winning from s.

**Per move, the closest thing to "chance this move wins" is Q(s, a)** inside the search. It's the average of the v values backed up through move a, i.e. the expected outcome after playing a. Q is computed during search; the network doesn't output it directly.

| Quantity | Answers | Where it comes from |
|---|---|---|
| **p(s, a)** | "Which moves are worth searching / would search choose?" | policy head, trained on π |
| **v(s)** | "How good is this position for me?" | value head, trained on z |
| **Q(s, a)** | "How good is move a, according to the search?" | MCTS: W / N |
| **N(s, a)** | "How confident is the search in a?" | MCTS visit count; used to pick the move |

**One-liner:** "The policy p predicts where search will spend its effort, a ranking prior. The value v predicts the expected outcome. Move-level win estimates are the search's Q values, and the final move is chosen by visit count."


---

# UI: Othello Player

## Turn 5

**You:**

> How are different levels managed using temperatures? How is the level of the User gauged for the mybuddy part ? how are hints handled ?

**Claude:**

`[Bash: grep -rniE "buddy" --include=*.md --include=*.rs --include=*.ts --include=*.svelte --include=*.json ]`

`[Bash: sed -n 40,70p models/oroletto-8x8.json; grep -n -iE "hint" api/src/session.rs ui/src/lib/engine/web-]`

`[Bash: sed -n 245,268p api/src/session.rs; grep -n -iE "hint" SPEC.md | head]`

Your "mybuddy" part isn't in the project, so I've guessed what you mean at the end. Here are the other two answers.

## Levels and temperature

**Design (SPEC §3.3 and `models/oroletto-8x8.json`):**
- Every level uses the same network and the same search of 200 simulations. The only difference is a move temperature T.
- After the search, each legal move has a visit count N. The computer picks a move with probability proportional to N^(1/T).

| Level | T | What it does |
|---|---|---|
| 1 Easy | 8.0 | Visit counts are flattened, so it plays almost at random. |
| 2 Medium | 2.0 | It prefers good moves but often picks the second or third. |
| 3 Hard | 1.0 | It picks in proportion to the visit counts. |
| 4 Master | 0.0 | It always plays the most-visited move. |

- The values come from the manifest at runtime, and the manifest still says `levels_calibrated: true`. The code has not read it yet. My summary has it as `false`, so check which is right.
- **Right now Oroletto is not wired in.** That is milestone M4. On the desktop, levels 2–4 are greedy placeholders. The browser build uses classic bots instead: random, greedy, alpha-beta at depth 3, and alpha-beta for about 0.9 s.

## Hints

- The hint uses the **level-4 engine**, which is the strongest and plays the most-visited move, so a hint is always the engine's best move.
- It only works in a vs-computer game on your turn. Otherwise it returns `HintUnavailable`.
- It runs the search synchronously in `api/src/session.rs`, so the app calls it off the UI thread.
- The UI highlights the square with a ring. Orolleto also says something when you ask for a hint.
- **The game doesn't record hints.** Nothing logs whether you used one, and the review doesn't count it.

## "mybuddy" and gauging your level

**Nothing in the project gauges your level.** The levels are fixed: you pick one and it never adapts. There's no rating, no win-rate tracking, and no "mybuddy" feature. The nearest thing is the post-game review. It scores each move you play against a ranking of the computer's moves, and it shows a hit rate. That measures how close your moves were to the computer's choices, but it isn't stored or used to change anything.

If you meant a "buddy" mode that adapts to you, I could build it. One way:
1. Take your review hit rate over recent games and keep a running estimate of your level.
2. Map that estimate to a temperature, so the opponent stays a bit stronger than you.
3. Log hints too, so leaning on them counts against your estimate.

Is that what you meant? And how should the estimate move between games, if you want this?


---

## Turn 6

**You:**

> if alpha zero trains p why does the model consider visit counts still ?

**Claude:**

The network's policy output `p` is a prediction made in one forward pass, with no lookahead. The visit counts come from the search, which uses `p` as a starting guess and then checks it with lookahead. The search result is better than the raw `p`.

**How the search uses `p`.** MCTS (PUCT, `c_puct` 1.5) starts each move with `p` as a prior. It then runs 200 simulations. Each one walks down the tree, prefers moves with a high prior or a good value so far, and updates the values from the value head. The visit count N of a move ends up as "how often the search kept coming back to it after seeing what happens next".

**Why the network is trained on visit counts.** In AlphaZero the policy head is trained to imitate the search's visit distribution (cross-entropy against N/ΣN), and the value head is trained on the game outcome. The loop is:

1. The search improves on `p`.
2. `p` is trained to match the improved search.
3. The next search starts from a better `p`.

So the network is trained to approximate the search, not to replace it. After training, `p` alone is still a bit weaker than `p` plus search. The visit counts are the better policy.

**Why this project needs visit counts.** The levels are built on them, as described in my last answer. Move choice is proportional to N^(1/T), and T=0 is the argmax of N.

- Raw `p` would offer only one weakening knob, a softmax temperature on the logits. The counts give you the search's own ranking, so a "weak" level is a softened version of the strong play, not a different and noisier player.
- The visit counts also feed the review. When Oroletto is wired in, they replace my current alpha-beta ranking. The top-k=3 and top-p=0.2 checks run over the visit shares.

You could skip the search and play from `p` alone. That would be faster, but it would play a weaker game at every level, including Master. It would also leave the top level with nothing to separate it from the levels below, because Master's top rank comes from the search.


---

## Turn 7

**You:**

> how are the visit counts kept track of and all what is the use of p then ? explain the enire idea of alpha zero ?

**Claude:**

## The idea in one paragraph

AlphaZero learns Othello with no human games. A neural network guesses good moves and who is winning. A tree search checks those guesses by looking ahead. The search produces better moves than the raw guesses, and the network is trained to imitate those better moves and the final game results. Repeat, and each side improves the other.

## 1. The network (two outputs from one body)

Input is 3 planes of 8×8: your discs, the opponent's discs, and legal moves, always from the side to move.

- **Policy head `p`:** 65 numbers, one per square plus pass, which become a probability for each move. It answers "which moves look promising here?".
- **Value head `v`:** one number from -1 to +1. It answers "how good is this position for the side to move?".

## 2. The search tree and the visit counts

For each position the search builds a tree. Every edge (position, move) stores three things:

- **N**: how many simulations went through this edge (the visit count).
- **W**: the sum of the values returned below it. **Q = W/N** is the average.
- **P**: the prior, which is `p` from the network, stored once when the node is first created.

One simulation does this:

1. **Select.** Start at the root and repeatedly pick the child move maximising `Q + c_puct · P · √(ΣN) / (1 + N)`. This is PUCT. The first term prefers moves that have worked well so far. The second prefers moves with a high prior that have not been tried much. As N grows, the second term shrinks, so the search settles on what the values show.
2. **Expand.** On reaching a position not yet in the tree, run the network once. Store `p` as the priors of its children, and take `v`.
3. **Back up.** Walk back up the path. Add `v` to each edge's W, adding 1 to its N. The sign is flipped each ply, because what is good for one player is bad for the other. Every move, including a pass, flips the side to move, which is the "value negated each ply" line in the manifest. If the game is over, use the real result instead of `v`.

After 200 simulations the root's children have counts N, and the counts are the search's decision. They are stored in the tree nodes (in a map from position to its edges), and there's nothing else to track.

## 3. So what is `p` for?

`p` is not the final answer. It steers where the 200 simulations are spent. Othello has about 10 legal moves, and each has about 10 replies, and so on. Without `p` the search would spend its budget evenly and waste most of it on bad lines. With `p` it goes deep on the plausible moves and ignores the rest. `v` does the matching job at the leaves: instead of playing each line to the end with random moves, one network call estimates the result.

A better `p` and `v` make the same 200 simulations stronger. And a very good `p` can be nearly right on its own, which is why the search only has to correct it.

## 4. Training: self-play

1. Play many games where the model plays both sides. At every move, run the search and record three things: the position, the visit distribution π = N/ΣN, and later the final result z (+1 win, -1 loss, 0 draw, from that mover's side).
2. Train the network so that `p` matches π (cross-entropy) and `v` matches z (squared error).
3. Use the new network for the next round of self-play. Repeat.

Two things keep it learning:
- **Exploration.** During training the root priors get Dirichlet noise (the manifest's `dirichlet_alpha` 0.5), and early moves are *sampled* from the counts instead of taken as the maximum. This makes the games varied.
- **The improvement loop.** The search is stronger than `p` alone, so π is a better target than the current `p`. Training pulls `p` towards it, and the next search is better still.

The manifest says this model trained on about 100,000 self-play games, with a 6-block, 64-channel network of about 458,000 parameters.

## 5. Playing, and where levels come in

When playing a real game there is no noise and no training. You run the 200-simulation search and pick a move from the counts. That is why the visit counts still matter after training: they are the output of the search, and `p` is only its starting guess. The temperature T then chooses how greedily to follow the counts (T=0 is the most-visited move, T=8 is nearly random), which gives the four levels.


---

## Turn 8

**You:**

> what are the global structures being handled in Alpha zero ? What is used for final predictions of which move to take ?

**Claude:**

## The global structures

AlphaZero keeps four things across the whole run, and one structure that lives only for a single move.

**Across the whole run:**
1. **The network weights** (policy and value heads, one shared body). This is the model that gets learned, and the only thing shipped in the app (`oroletto-8x8.onnx`).
2. **The replay buffer.** It holds recent self-play records of (position, π, z): the position, the search's visit distribution, and the game result. Training samples random batches from it, and old records are dropped as new games arrive.
3. **The self-play workers.** They are not a data structure, but they are global state in practice: many games run at once, each calling the current network.
4. **The optimiser state and training counters** (learning rate, step count). The original AlphaGo Zero also had a "best model" gate where a new network had to beat the old one. AlphaZero dropped the gate and always uses the latest network.

**Per move, and thrown away afterwards:**
5. **The search tree.** It is a map from position to its edges, and each edge stores N, W (so Q = W/N) and P. It is built from scratch for each move. Some implementations keep the subtree under the move actually played as the start of the next search, but nothing carries over beyond that. At play time, only the network weights and this tree are needed.

## What chooses the final move

**The root's visit counts N**, and nothing else. After the 200 simulations, you look at the children of the root:

- Compute π(move) = N(move) / ΣN.
- **Training self-play:** sample a move from π early in the game (to vary the games), then play the most-visited move later.
- **Real play (this app):** sample from N^(1/T). T=0 means the argmax of N, which is the Master level.

What is *not* used for the final choice:
- The raw policy `p` is only a prior inside the search.
- The Q values are not used directly. A move with a high Q but few visits is still not chosen, because the visits already encode how the search weighed Q against the prior. This is also why the counts are more stable than Q: a move with one lucky simulation has a high Q but a low N.
- The value head `v` is not used to pick moves. It only scores leaf positions inside the search.


---

## Turn 9

**You:**

> how are the 200 simulations run ? what decides which direction each simualtion takes? is N then used to calculate Pi which is used to update P ?

**Claude:**

Simulations run one after another, not in parallel (implementations that batch them to use the GPU still select each path one at a time). The last part of your question has a catch, covered under "Is N used to update P?".

## How one simulation runs

Each simulation starts at the root, which is the current position, and follows a path of the tree's own best-looking moves down to a leaf. A leaf is a position the tree hasn't expanded yet.

**At each node on the way down**, pick the move that maximises

`score(a) = Q(a) + c_puct · P(a) · √(ΣN) / (1 + N(a))`

with `c_puct` = 1.5 here, and ΣN the total visits over this node's children. The two terms:

- **Q(a)** is the average value seen through this move so far. It pulls the simulation towards moves that have been working.
- **The second term** is an exploration bonus. It is large when the prior P(a) is high and N(a) is small, and it shrinks as N(a) grows.

**Example.** Take the root's 3 moves A, B and C, with priors 0.6, 0.3 and 0.1, and nothing visited yet, so every Q is 0. The first simulation picks A, the highest prior. If A looks bad once its value comes back, say Q(A) becomes -0.4, then A's score drops, and the next simulations go to B and then C. If A looks good, its Q stays high and it keeps winning while the bonus on B and C shrinks slowly.

That is what decides the direction of each simulation: priors early on, values later, with the bonus term deciding how fast the balance shifts.

**When the path reaches a leaf:**
1. If the game is over, the value is the real result.
2. Otherwise, run the network on that position once. Store its policy as the priors P of the new children, and take its `v` as the value.
3. **Back up** along the path. Each edge gets N += 1 and W += value, with the sign flipped at every ply, and Q = W/N is recomputed.

After 200 of these, the root's children have their final counts N.

## Is N used to update P?

Not the way you described, and these two things are different:

- **During a search, P is never updated.** It is whatever the network output when the node was created. Only N, W and Q change as simulations run.
- **π = N/ΣN is a training target, not an update rule.** After the search it is recorded together with the position. Later, in the training step, the network's weights are adjusted by gradient descent so that its policy output for that position moves towards π. The same step trains `v` towards the game result.

So the full chain is: the search turns P and v into the counts N, and training turns the recorded π into new network weights. The next search then gets new P values from the better network. P changes only through the weights, between rounds of self-play.

In this app no training happens. The network weights are fixed. The search runs every move with P from the network, builds the counts N, and then the temperature picks the move from them.


---

## Turn 10

**You:**

> what is puct ?

**Claude:**

PUCT stands for **Predictor + Upper Confidence bounds applied to Trees**. It is the rule that decides which move a simulation follows at each node of the search tree.

**Where it comes from.** It builds on UCB1, a rule for the multi-armed bandit problem: you have several slot machines with unknown payouts, and you must balance pulling the one that has paid best so far (exploit) against trying ones you know little about (explore). UCT applies UCB1 to every node of a game tree. Plain UCT has no opinion about which moves are plausible, so every untried move starts equal. PUCT adds a **predictor**, the network's policy P, which tilts the exploration towards moves the network likes.

**The rule used here:**

`score(a) = Q(a) + c_puct · P(a) · √(ΣN) / (1 + N(a))`

| Part | Meaning |
|---|---|
| `Q(a)` | Average value of the move so far (W/N). The exploit term. |
| `P(a)` | The network's prior for the move. |
| `√(ΣN)` | Grows slowly as the node gets more total visits, so exploration never fully dies. |
| `1 + N(a)` | Shrinks the bonus for a move as it is visited more. The `1+` avoids dividing by zero for an unvisited move. |
| `c_puct` | A constant (1.5 here) for how much to favour exploring over exploiting. Larger values spread the simulations more widely. |

**How it behaves.**
- Early, when N is small everywhere, the bonus dominates, so the search follows the priors.
- Later, as N grows, the bonus shrinks and Q dominates, so the search follows what the values say.
- A move with a low prior still gets visited eventually, because `√(ΣN)` keeps growing while its own N stays small. A strong enough Q elsewhere can keep it waiting for a long time.

The simulation picks the highest-scoring move at each node and moves on to the child, which is how the 200 simulations get spread over the tree.


---

## Turn 11

**You:**

> during deployment in a packaged model what do p and v do ?

**Claude:**

In the packaged model, `p` and `v` do the same jobs as during training. The only things missing are the training step and the root noise.

**What ships.** `oroletto-8x8.onnx` is a frozen set of weights. Its input is the 3 planes (your discs, the opponent's discs, legal moves). Its outputs are `policy_logits` (65 numbers) and `value` (1 number), as listed in the manifest. Nothing in the app updates it.

**What the app does with each output:**

- **`p` (policy logits):** the app masks out illegal moves, applies a softmax over the legal ones, and stores the result as the priors P on the children of the node being expanded. It is used inside the PUCT rule to steer the simulations. It is never read off to pick the move.
- **`v` (value):** used as the score of a leaf position. It goes into the back-up and updates N, W and Q along the path, with the sign flipped each ply. At a finished game the real result is used and the network is not called.

**One move in the app:**
1. Run the network on the current position to create the root, getting `p` and `v`.
2. Run 200 simulations. Each one ends in a leaf, calls the network once, and backs up `v`. That makes roughly 200 network calls per move, which is why the SPEC budgets 2 seconds per move.
3. Read the root's visit counts N, apply the level's temperature T, and play the chosen move.
4. Throw the tree away.

**What changes from training:**
- No Dirichlet noise at the root (the manifest says "no root noise").
- No recording of π or z, and no replay buffer.
- No gradient steps. `p` and `v` come out of the same fixed weights every time, so the same position always gets the same priors and value. Any variation between games comes only from the temperature sampling.

**Hints and the review.** A hint uses the same search at T=0. The review will use the visit shares from the same search, once Oroletto is wired in.

The app does not do any of this yet. Oroletto and the search are milestone M4, and today the desktop levels are greedy placeholders.


---

## Turn 12

**You:**

> why did we not include any of this in the app ?

**Claude:**

`[Bash: ls -la models/ tests/golden 2>&1 | head -20; sed -n 205,222p SPEC.md; git log --oneline | tail -15; ]`

The model and the plan were ready, but the work was never done. Nobody decided to leave it out.

**What exists.**
- `models/oroletto-8x8.onnx` (1.8 MB) is in the repo.
- Its manifest and a parity test file (`tests/golden/parity_8x8.json`) are there too.
- SPEC §9 lists "M4: ONNX inference + MCTS + temperature" as the next step after the plain UI.

**What I built instead.** The milestones in the git history go like this:
- M0–M3 (engine, API, Tauri shell, plain UI) are done.
- Next came the M5-style work: the casino look, chip animations, the web build, the post-game review, and Orolleto and his dialogue pool.
- M4 never started. `Cargo.toml` has no ONNX runtime dependency, so nothing in the Rust code can load the model.

**Why that order happened.** Every request in this session was about the look, the web build, the review, or the dealer, so I worked on those. I should have raised the gap sooner. The review feature in particular says "the computer's best moves" while it's really an alpha-beta heuristic. The app also says Easy to Master are all Oroletto, but on the desktop levels 2–4 are all the same greedy bot, and in the browser they are classic bots.

**What M4 needs:**
1. Add an ONNX runtime to the Rust side (the `ort` crate, or the pure-Rust `tract`).
2. Write the MCTS with PUCT: 200 simulations, `c_puct` 1.5, no root noise, value negated each ply.
3. Add the temperature sampling over the visit counts, with the four levels read from the manifest.
4. Run the parity tests against `parity_8x8.json`, to check the Rust network inputs and outputs match the PyTorch ones.
5. Connect the hint (level 4) and the review (visit shares) to it.

**The browser build is harder.** It has no network, so it would need an in-browser runtime such as `onnxruntime-web`, plus a TypeScript MCTS. The SPEC currently says Oroletto stays desktop-only.

Do you want me to start on M4 for the desktop app? I'd begin with the Rust MCTS and the parity test, since that's where a mistake would be hardest to spot later. The manifest says `levels_calibrated: true`, and I'd also want to confirm that is correct before trusting the temperatures.


---

## Turn 13

**You:**

> explain how the model we have deployed works

**Claude:**

`[Bash: sed -n 66,120p models/oroletto-8x8.json; python3 -c "
import onnx
m=onnx.load('models/oroletto-8x8.o]`

A caveat first: the app doesn't run this model yet (see my last answer). So this describes the model file and how it will run, not what happens when you play now.

## What the file is

`oroletto-8x8.onnx` is a trained neural network, 1.8 MB. It has about 458,000 parameters and a residual-network design: 6 blocks of 64 channels. It was trained in your OthelloRL repo (run `alphazero_8x8/seed0`, commit `727a50d`) on about 100,000 self-play games. I haven't opened the graph itself, since the `onnx` Python package isn't installed here. The structure comes from the manifest, so I can't confirm the layer details beyond what it states.

## The forward pass

**Input:** a tensor of 3 planes of 8×8, from the side to move's point of view:
1. its own discs
2. the opponent's discs
3. its legal moves

The same network plays both colours, because the board is always shown as "me versus them".

**Body:** the planes go through a convolution into 64 channels, then the 6 residual blocks. Each block is two convolutions with a skip connection that adds the block's input back to its output. Convolutions suit Othello because patterns like edges, corners and diagonals mean the same thing wherever they sit on the board.

**Two heads split off from the shared body:**
- **Policy head:** outputs `policy_logits`, 65 numbers (64 squares and pass). Pass is index 64, and squares are numbered row×8+col, with a1 = 0.
- **Value head:** outputs one number, `value`, from -1 to +1. It estimates how the position looks for the side to move.

**Accuracy of the export:** the manifest says the ONNX output differs from the PyTorch original by at most 2.5e-6 over 200 test positions. So the export is faithful.

## How strong it is by itself

The manifest's `final_scores` give some idea. The "policy_only" rows play straight from the policy head with no search.

| Opponent | Search | Policy only |
|---|---|---|
| random | 1.00 | 0.98 |
| heuristic | 1.00 | 0.96 |
| minimax depth 4 | 0.97 | 0.64 |

The score is a win fraction. Policy only beats the weak bots but loses ground against depth-4 minimax at 0.64, while the full search reaches 0.97. That gap is why the app needs the MCTS: the raw network is good, but the search takes it from decent to dominant. These figures are your OthelloRL results, and I haven't reproduced them.

## How the app will use it

For each computer move:
1. Encode the board into the 3 planes.
2. Run 200 MCTS simulations, calling the network once per new leaf. The policy output gives the PUCT priors and the value output scores the leaf.
3. Pick the move from the root's visit counts at the level's temperature (8.0, 2.0, 1.0 or 0.0).

## Things to check before wiring it in

- **`levels_calibrated`** says `true` in the manifest on disk, but my earlier notes said `false`. I'd confirm the values with whoever last edited it before relying on the temperatures.
- The **parity file** (`tests/golden/parity_8x8.json`) is how I would check the Rust side feeds in the same planes and gets the same outputs.


---

## Turn 14

**You:**

> given one starting position how are th 200 simulation smade to differ ?

**Claude:**

They differ because the tree changes after every simulation. In the app there is no randomness inside the search, since the root noise is switched off. The search is a deterministic function of the network's output and the tree built so far.

## What changes between simulations

Each simulation finishes by backing up a value, which changes N, W and Q along its path. The PUCT score for the same node is then different next time:

`score(a) = Q(a) + c_puct · P(a) · √(ΣN) / (1 + N(a))`

Walking through the first few simulations from one position:

1. **Simulation 1.** The root is expanded and has priors P but no visits, so every Q is 0. The score is just the exploration bonus, which is highest for the highest prior. It goes to that move, say A, and the network is called on the resulting position. The value comes back, say it's bad for the opponent, which is good for you. A gets N = 1 and a positive Q.
2. **Simulation 2.** Now A has N = 1, so its bonus has shrunk (`1 + N` is 2), but its Q is positive. Whether A or B is higher depends on the numbers. If A's Q is large enough, A wins again, and this time the path goes one level deeper, into A's best reply. If A's value was only mildly good, B's untouched bonus wins and the simulation goes to B.
3. **Later simulations.** As N grows on a move, its bonus fades. Moves that have been tried and disappointed lose to untried moves, and moves that keep returning good values keep getting picked, and are explored deeper.

So the simulations differ because earlier ones have used up part of the bonus and have moved the Q values. Two simulations from the same root only follow the same path when the tree state still ranks things the same way.

## Where the variety comes from

- **The priors P** decide which moves are tried first.
- **The values v** from the network at each new leaf decide which moves get promoted or dropped.
- **The shrinking bonus** forces the search to visit less-favoured moves occasionally, so it does not lock onto one line too early.

## Where there is randomness

- **Training self-play:** Dirichlet noise (alpha 0.5) is added to the root priors, so different games start with different emphasis.
- **Choosing the move:** the sampling from N^(1/T) at T above 0 is random, and that is where your four levels differ.
- **The search itself in the app:** none. Run it twice on the same position with the same network and you get the same visit counts, the same π, and at T=0 the same move every time.

One caveat: if you ever batch simulations in parallel to speed up inference, implementations usually add a "virtual loss" to stop several threads from picking the same path. That introduces order-dependence, but not true randomness.


---

## Turn 15

**You:**

> is MCTS built during test as well or is it just p there ?

**Claude:**

MCTS is built at test time too, fresh for every move. In the standard AlphaZero setup, the packaged model is only the network. The search is code that runs around it, so it is not stored in the file.

## What happens at play time

- **Full AlphaZero play (the plan for this app):** for every computer move, the app builds a new tree and runs 200 simulations. The network is called about 200 times per move. The decision then comes from the root's visit counts.
- **Policy only:** you could skip the search, take `p`, mask the illegal moves and play the top one (or sample from it). It is one network call per move, so it is very fast.

## Why not just use `p`

Your own manifest shows the difference. Against minimax at depth 4, the policy alone scored 0.644 and the full search scored 0.966. `p` is a good guess with no lookahead, so it misses tactics a few moves deep, like a move that gives up a corner. The search checks those cases.

## What decides which one to use

It's a design choice, not a property of the model. Both use the same file. The SPEC chose the search because:
- the levels are built from the visit counts and a temperature;
- Master needs the strongest play;
- the time budget of about 2 seconds per move allows 200 simulations.

A policy-only mode could be a cheap extra: a "fast" opponent or a very low level. Its strength would come from `p` alone, though, and the temperature would act on the logits, not on the visit counts. That would make the ladder less well defined than the one in the SPEC.


---

## Turn 16

**You:**

> then why do we need to package a RL at all, we are not updating the model right ? then how does having a good P network help at test time ?

**Claude:**

You package the result of the RL, not the RL. The training process (self-play, replay buffer, gradient updates) stays in your OthelloRL repo. What's in the app is a frozen network. And you're right that nothing updates at test time: every bit of learning has already happened, and the network is the stored result.

## Why a good `p` and `v` still help when frozen

The search has a small budget, and the network decides how well it is spent.

**1. `p` makes 200 simulations reach deep.**
Othello has roughly 10 legal moves per position. A uniform search over 10 moves with 200 simulations gives about 20 visits per move, which reaches depth 2 or 3 and no further. With a good `p`, most simulations go to the 2 or 3 plausible moves, and the tree gets much deeper along the lines that matter. The same 200 simulations then see further, because the network has already thrown away most bad moves.

**2. `v` replaces rollouts.**
Classic MCTS scores a leaf by playing random moves to the end of the game, which is slow and noisy, because random play is a poor guide to who is winning. Here, one network call returns a learned estimate of the result. That single call is the reason 200 simulations are enough.

**3. The search corrects the network where it is weak.**
`p` and `v` are approximations. They are good on average and wrong on some tactical positions. The search looks a few moves ahead and uses the real game rules, so it catches some of those mistakes. The network, in turn, supplies what the search lacks: an evaluation that pure lookahead can't compute.

So each part covers the other's weakness, and neither is enough alone. Your results show it: policy only was 0.644 against minimax depth 4, and the search with the same weights was 0.966.

## What each alternative would give you

| Approach | What you'd get |
|---|---|
| Frozen network, no search | Fast, weaker (the policy-only rows). |
| Search with no network (random rollouts, uniform priors) | Much weaker at 200 simulations. It would need thousands of rollouts. |
| Frozen network and search (the plan) | Strongest at the budget of about 2 s per move. |

## Why the RL was still needed

The trained network is where all the Othello knowledge lives: which positions are good, which moves look promising. Producing that knowledge took the 100,000 self-play games. The app only borrows it.

You can also keep learning later by running more training in OthelloRL and swapping in a new `.onnx` plus manifest. The app would not need code changes, since the manifest carries the settings.


---

## Turn 17

**You:**

> how are the p component and v component used ?

**Claude:**

Here is one network call, followed through with example numbers.

## What comes out of a call

The search reaches a position it hasn't seen and calls the network once. It gets back:
- `policy_logits`: 65 raw numbers
- `value`: one number, say +0.3 (the side to move is slightly ahead)

## How `p` is used

1. **Mask the illegal moves.** Suppose only 3 moves are legal: C4, D3 and F5. Throw away the other 62 logits.
2. **Softmax the legal ones** into probabilities, e.g. C4 = 0.55, D3 = 0.35, F5 = 0.10.
3. **Store them** as the priors P on the new node's child edges. They are written once and never changed during this search.

Later, whenever a simulation passes through this node, the PUCT rule reads them in the exploration bonus `c_puct · P(a) · √(ΣN) / (1 + N(a))`. So `p` decides which move gets tried first and how long a move with few visits keeps getting a chance. C4 gets the first visit and F5 gets one only when the others stop looking better.

## How `v` is used

1. The simulation ends at this new node, so `value` +0.3 is the simulation's result.
2. **Back up.** Go back up the path to the root. At each edge, add 1 to N and add the value to W. The value's sign flips at each step, because +0.3 for the side to move here is -0.3 for the player who moved into this position. So the parent edge gets -0.3, the one above it +0.3, and so on.
3. Recompute Q = W/N on each edge.

So `v` doesn't choose anything directly. It becomes the Q term in later PUCT decisions, which makes moves that lead to good-looking positions more attractive.

## In one line

`p` is read when a node is created and shapes where later simulations go. `v` is read when a simulation ends and shapes how good each move looks afterwards. At the end of 200 simulations, only the root's visit counts matter for the move choice.

Neither is used for the final move directly. The only place `p` alone could pick a move is the policy-only mode, with no search.


---

## Turn 18

**You:**

> what is W ?

**Claude:**

W is the **total value** collected on an edge (a move from a node). It's the running sum of every value backed up through that edge during the search.

- Each simulation that passes through the edge adds its result to W. The result is either the network's `v` at the leaf or the real game result.
- The sign is flipped at each ply on the way up, so W is always counted from the point of view of the player making that move.
- **Q = W / N** is the average value per visit, and the one used in the PUCT score.

**Example.** The move A has been visited 4 times. The values backed up through it were +0.3, +0.1, -0.2 and +0.4. Then N = 4, W = +0.6 and Q = 0.15.

Why store W and not just Q? A sum is easy to update: add the new value, then divide by N. It also lets the average be computed exactly without storing the earlier values.

W is not used on its own. Neither W nor Q picks the final move; only N does.


---
