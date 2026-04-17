---
title: "We Stopped Too Early: Why I'm Building a Classical Chess Engine from Scratch"
published: 2026-03-28
updated: 2026-03-28
description: 'Writing a classical chess engine from ground up'
tags: [Chess,AI,NNUE,Classical Chess]
image: "../../assets/images/post-covers/classical-chess-engine-from-scratch.png"
category: 'Chess'
draft: false 
---

> Cover image source: [civitai.com](https://civitai.com/images/5773325)


I've been writing chess engines for few years now. Not playing chess. Writing engines. There's a difference, and it matters for what I'm about to say.

I'm an average chess player at best. I know enough to not blunder my queen in the opening, and that's roughly where my playing ability stops. But the engines, the code, the search trees, the eval functions, the way you can make a CPU chase down the best move in a position with nothing but logic and math, that stuff has had me hooked since I first touched it. I've written engines. Bad ones first, then less bad ones. I've read the papers, dug through every chess engines ever existed , benchmarked things that didn't need benchmarking like i have OCD. (yeah yeah, i know i need help). I've spent weekends staring at positions where my engine made a move I thought was terrible and then slowly realized the engine was right and I was wrong.

So when I say the chess engine world moved on too fast from classical search, I'm not saying it from the outside.

## How Classical Engines Actually Work

Before I get into what I think is missing, I want to explain how classical engines actually work. Not at a surface level. The real thing. Because I've noticed that most people who talk about "classical vs neural" don't have a clear mental model of what classical eval actually does, and it matters for the argument I'm making.

A classical chess engine has two separable components: search and evaluation. These are different problems solved in different ways, and conflating them causes a lot of confusion.

**Search** is the process of exploring the game tree. Given a position, the engine generates all legal moves, plays them out, then generates all responses, plays those out, and keeps going until it hits a depth limit or a terminal state. The naive version of this is called minimax: at your turn you pick the move that maximizes your score, at the opponent's turn they pick the move that minimizes your score. The score propagates back up the tree.

If the game tree had branching factor *b* and you searched to depth *d*, the number of nodes you'd need to visit is:

$$N = b^d$$

For chess, *b* is roughly 35 on average. At depth 10, that's 35^10, which is around 2.76 \* 10^15 nodes. On modern hardware doing maybe 10 million nodes per second, that's about 87 years of computation for a single move. Completely unusable.

The first major insight that made chess engines practical is alpha-beta pruning. The idea is that you don't need to explore the entire tree. If you're searching a subtree and you find a move that's already worse than something you've already found, you can stop searching that subtree entirely. The opponent would never let you play into it anyway.

The algorithm maintains two values: alpha (the best score the maximizing player is guaranteed) and beta (the best score the minimizing player is guaranteed). When beta falls below alpha, you prune.

```python
function alphabeta(position, depth, alpha, beta, maximizing):
    if depth == 0:
        return evaluate(position)
    
    if maximizing:
        value = -infinity
        for move in legal_moves(position):
            value = max(value, alphabeta(play(position, move), depth-1, alpha, beta, false))
            alpha = max(alpha, value)
            if alpha >= beta:
                break  // beta cutoff
        return value
    else:
        value = +infinity
        for move in legal_moves(position):
            value = min(value, alphabeta(play(position, move), depth-1, alpha, beta, true))
            beta = min(beta, value)
            if beta <= alpha:
                break  // alpha cutoff
        return value
```

In the best case, with perfect move ordering, alpha-beta reduces the effective branching factor from *b* to roughly *sqrt(b)*. For chess that means around 6 instead of 35, which brings depth 10 down to 6^10 = about 60 million nodes. Manageable.

But move ordering matters enormously. If you search the best moves first, you get more pruning. Real engines spend a lot of effort on this: killer moves, history heuristics, static exchange evaluation to order captures. The idea is to search the most promising moves first so alpha-beta cuts off the garbage quickly.

On top of this, engines add a bunch of other things. Null move pruning: if you skip your turn entirely and the position is still good for you, it's probably not worth searching deeply. Late move reductions: moves that are unlikely to be good get searched to shallower depth. Aspiration windows: search with a narrow window around the expected score, and widen only if the score falls outside it. Iterative deepening: search depth 1 first, then depth 2, then depth 3, using results from shallower searches to order moves for deeper ones. Transposition tables: cache positions you've already evaluated so you don't re-search them when the same position is reached by a different move order.

All of this adds up to engines that can realistically search 20-30 moves deep in a few seconds.

**Evaluation** is what happens at the leaves. When the engine hits its depth limit on a node, it needs a number. That number is the static evaluation of the position, a single score representing how good the position is for White (positive) or Black (negative).

Classical eval is a weighted sum of hand-crafted features. Here's a simplified version of what Stockfish's classical eval was doing before NNUE:

```plaintext
eval(position) = 
    material_score(position)
  + pawn_structure_score(position)
  + piece_mobility_score(position)
  + king_safety_score(position)
  + passed_pawn_score(position)
  + bishop_pair_bonus(position)
  + rook_open_file_bonus(position)
  + ... (dozens more terms)
```

Each of these terms is computed by looking at the board state directly. Material is just counting pieces with known values (pawn = 100 centipawns, knight = 320, bishop = 330, rook = 500, queen = 900). Pawn structure looks at doubled pawns, isolated pawns, backward pawns, pawn chains. Mobility counts how many squares each piece can reach. King safety looks at pawn shields, open files near the king, attacking pieces.

The weights on these terms were tuned, originally by hand, and later by automated methods like Texel tuning, where you adjust weights to minimize prediction error on a large set of positions with known outcomes.

This whole system is interpretable. You can look at it and say "the engine thinks this position is +1.3 because White has a passed pawn worth +0.5, better king safety worth +0.4, and a bishop pair worth +0.15." Every piece of the score is traceable.

The problem is that chess positions have interactions that this additive model can't capture well. A passed pawn is worth different amounts depending on whether there are rooks on the board, what color the opposing bishops are, how far advanced it is, whether the opposing king can catch it. These interactions are real and strong, but modeling them explicitly requires exponentially more features as you try to capture more of them. The classical eval was always an approximation, and the approximation had a ceiling.

## What NNUE Actually Did

NNUE stands for Efficiently Updatable Neural Network. The name is more descriptive than it sounds.

The architecture was designed specifically for use inside a chess engine search loop. The key constraint is speed: the eval function gets called millions of times per second, so it needs to be fast. A full neural network forward pass for every node would be prohibitively slow. NNUE solves this with a specific architecture and an incremental update trick.

The input to the network is a high-dimensional binary feature vector. The most common scheme is called HalfKA or HalfKP: for each side, and for each piece type and square combination, you have a binary feature for whether that piece is on that square, relative to the king's position. This gives you something like 40,000 input features, but almost all of them are zero for any given position.

The network has a few fully connected layers:

```plaintext
Input layer:   ~40,000 features (sparse)
Hidden layer 1: 256 neurons (ReLU)
Hidden layer 2: 32 neurons (clipped ReLU)
Hidden layer 3: 32 neurons (clipped ReLU)
Output:         1 value (eval in centipawns)
```

The first hidden layer is the expensive one because the input is large. But here's the trick: when you make a move on the board, only a few features change. A piece moved from square A to square B turns off a few features and turns on a few others. Instead of recomputing the entire first layer from scratch, you incrementally update it. Add the weights for the new features, subtract the weights for the old ones. This makes the first-layer update O(k) where k is the number of changed features, not O(n) where n is 40,000.

The subsequent layers are small enough (256 to 32 to 32 to 1) that they can be computed quickly. And with SIMD instructions doing vectorized integer arithmetic, the whole forward pass takes a few dozen nanoseconds.

The result is an eval function that's both faster and much stronger than the old classical eval. The network learns, from millions of self-play games, exactly how pieces interact with each other, what king safety really means, when a passed pawn is dangerous and when it isn't. All the interactions that the classical weighted sum approximated badly, the network captures automatically.

When Stockfish switched to NNUE in 2020, it gained something like 80-100 Elo almost immediately. The jump was that clean. And it's continued improving every year since.

I'm not skeptical of any of this. NNUE is real and it works.

What I keep thinking about is the question it didn't answer.

## The Thing That Kept Bothering Me

For years I've had this thought I couldn't shake. It'd come up reading about new NNUE architectures, benchmarking my own engines, or just sitting with a position thinking about what makes it good or bad.

We never really answered the question of what evaluation *is*.

Every classical engine I've ever read, written, or studied answers the same question: *how good is this position?* It scores material. It looks at pawn structure. It adds something for king safety, checks mobility. Then it sums everything up into a number and says "this position is +1.3 for White." NNUE answers the same question, just with a more expressive function. The question itself never changed.

But I keep thinking about how grandmasters actually think. Not how they describe it in chess books, which is often post-hoc rationalization, but what they're actually responding to when they look at a position.

A strong player doesn't look at a position and think "White has a passed pawn worth 0.5 and better king safety worth 0.4." They look at the position and think: "White has a plan. Black is shuffling." Or: "There are six candidate moves for White, but only one of them actually threatens anything. Black has to find that one move or the game is over."

That's a statement about the *structure of the future*, not the *state of the present*. And it's the thing I've never seen formalized in a chess engine.

## The Classical Evaluation Ceiling

I've spent a long time thinking about where specifically classical eval breaks down. Not in vague terms. In specific position types I've tested against my own engines repeatedly.

**Zugzwang** is the clearest case. A position is in zugzwang when the side to move would prefer to pass. In pure zugzwang, any move you make makes your position worse. Classical eval has essentially no way to detect this. The features it computes are all about the current board state. Whether moving is an advantage or disadvantage depends entirely on the structure of the continuation tree, which classical eval doesn't look at.

Engines handle zugzwang in practice by searching deeply enough that the tree reveals it. But that's the search doing the work, not the eval. The eval itself is blind to it.

**Fortress positions** are another one. A fortress is a drawn position where the losing side can hold a draw by keeping pieces in a specific configuration, even though they're materially behind. Classical eval typically scores fortresses as losing, because the feature-based view sees material deficit and doesn't see why the position is actually drawn. Engines again handle this through deep search and endgame tablebases, not through eval.

**Quiet maneuvering positions** are harder to articulate but just as real. Sometimes a position is good for one side not because of any feature you can point to, but because they have a long-term plan and the opponent has nothing meaningful to do. The side with a plan is improving their position slowly while the opponent shuffles. Feature-based eval often scores these positions as roughly equal because nothing dramatic is happening, and then the engine is surprised when the slowly improving side converts.

All of these cases have something in common. The feature-based eval is missing information about what *can* be done from the position, not just what *is* in the position. It's evaluating the board state when it should be evaluating the board state's relationship to the future.

## A Different Question

What if instead of asking "how good is this position," the engine asked "how much does this position constrain the future?"

When a grandmaster looks at a position and says "White is clearly better," they're often responding to exactly this. White has a clear plan. Black's options are bad or forced. The position is good not because of any single feature, but because the future is lopsided.

Information theory has a way to measure this: entropy.

Shannon entropy of a probability distribution *P* over outcomes is:

$$H(P) = -sum_i [ p_i * log2(p_i) ]$$

High entropy means the distribution is spread out. Many roughly equal options. Low entropy means the distribution is concentrated. A few options dominate.

Applied to chess: imagine you're at a position and you have a probability distribution over your legal moves, weighted by how good each move is. If all your moves are roughly equally good (or bad), that's high entropy. If one move is clearly forced and everything else loses, that's low entropy.

Here's the thing: low entropy on one side combined with high entropy on the other is what a grandmaster means when they say "White has a clear plan and Black is lost." White's move tree is concentrated. Black's move tree is chaotic or all-losing.

So let me define this more carefully.

For a position *P* and a side *s*, let *M(P, s)* be the set of legal moves for side *s*. For each move *m*, let *v(m)* be a shallow evaluation of the resulting position (using any eval function). Convert these values to a probability distribution using softmax:

$$q(m) = exp(v(m) / T) / sum_{m' in M} exp(v(m') / T)$$

Here *T* is a temperature parameter controlling how sharply the distribution peaks around the best moves. Low temperature means the distribution concentrates on the top moves. High temperature means it spreads more evenly.

Now define the positional entropy for side *s* at position *P*:

$$H(P, s) = -sum_{m in M(P, s)} [ q(m) * log2(q(m)) ]$$

And define the entropy advantage for the moving side (White, say) over Black:

$$A(P) = H(P, Black) - H(P, White)$$

A positive *A(P)* means Black has higher entropy than White. Black has many options, most of them roughly equal in value, while White's options are more concentrated around a few clearly good moves. This is what "White has a clear plan" looks like mathematically.

A negative *A(P)* means White is the one lost in complexity, which is bad.

And crucially, *A(P)* near zero for *both* sides with high individual entropy values means the position is genuinely complicated and roughly equal. *A(P)* near zero with low individual entropy values means both sides have limited options, which is a different kind of equal: quiet, forced, maybe drawish.

These four cases correspond to real chess categories that a single material-and-features score completely flattens:

```plaintext
H(White) low,  H(Black) high   --> White has initiative, Black is struggling
H(White) high, H(Black) low    --> Black has initiative, White is in trouble  
H(White) high, H(Black) high   --> Complicated, roughly equal
H(White) low,  H(Black) low    --> Simplified, forced, probably drawish or endgame
```

No classical eval I've ever seen gives you that four-way distinction. The standard score gives you a single number on a line, and all four of those cases can produce the same score with different meanings.

## Handling Depth

One obvious problem: the entropy I defined above is computed over the *immediate* legal moves. That's only one ply deep. Chess is a deep game and shallow entropy might not mean much.

The solution is to recurse. Define the continuation entropy at depth *d*:

$$H_d(P, s) = (1 - lambda) * H_0(P, s) + lambda * E_{m ~ q} [ H_{d-1}(next(P, m), opponent(s)) ]$$

Where:

*   *H\_0(P, s)* is the immediate entropy defined above
    
*   *lambda* is a discount factor in (0, 1) controlling how much future entropy matters
    
*   The expectation is taken under the move distribution *q*
    
*   *next(P, m)* is the position after playing move *m*
    
*   *opponent(s)* flips the side
    

This is a recursive definition. At depth 0, you just use the immediate move entropy. At depth *d*, you take the immediate entropy and blend it with the expected future entropy of the positions you'll reach.

The discount factor *lambda* is important. It controls how much weight you give to future positions versus the current one. High *lambda* means the engine is forward-looking. Low *lambda* means it's more focused on immediate structure.

There's a connection here to temporal difference learning in reinforcement learning: the discounted sum of future rewards. Except here the "reward" is entropy, not a win/loss signal.

In practice, computing this exactly is expensive. You can't afford to recurse deeply on every node during search. The implementation approach I'm planning uses approximations: sample a subset of continuations under the distribution *q* rather than exhaustively branching, cache computed entropy values in an extended transposition table, and use the shallow version at leaf nodes while using the deeper version at root and early nodes.

## The Zugzwang Case, Formally

Here's how the formalism handles zugzwang. And I think it actually gets it right.

In zugzwang, any move the side to move makes is worse than passing. That means if White is in zugzwang, all of White's moves lead to positions that are better for Black. Formally, *v(m)* is low (bad for White) for every *m* in *M(P, White)*.

In this case, the softmax distribution *q* over White's moves is high entropy but with all values low. All options are roughly equally bad. The entropy is high not because White has many good choices but because White has many equally terrible ones.

But the key is what this does to the entropy advantage *A(P)*. Black, in the same position (before White moves), has options that avoid zugzwang entirely: they can play any move that maintains the zugzwang structure. Their distribution *q* is concentrated on those moves. *H(P, Black)* is low.

So *A(P) = H(P, Black) - H(P, White)* is large and negative: Black has low entropy (clear plan: maintain zugzwang) while White has high entropy (many moves, all bad). This correctly identifies that White is in trouble, which is exactly what zugzwang means.

A classical eval looking at the same position might see material equality and no obvious features pointing to a winner. The entropy eval sees the structure of the future and gets the right answer.

## The Fortress Case, Formally

Fortress is slightly different. Say Black is a piece down (classical eval says White is winning) but Black has arranged pieces in a fortress. Any White attempt to break through fails. Black's responses to all White incursions are forced and available.

From Black's perspective: high entropy on White's side (White has many attempts) but low entropy on Black's side (Black always has the clear defensive response). From White's perspective: high entropy (many plans to try) but none of them lead to progress.

The recursive entropy captures this because at depth 1, every White move leads to a position where Black has a high-quality, clear response (low entropy for Black). White's apparent entropy is high but "fake": the moves explore lots of territory that all leads to the same dead end.

Over multiple recursive steps, the engine would learn that White's high-entropy position doesn't translate to a high-entropy advantage because Black's responses are always available and clear.

This is exactly what tablebases tell us about fortress positions, arrived at from a completely different direction.

## Why CPUs, Why Classical, Why Now

I'm an engineer. I like CPUs. That's not a philosophical position, it's just what I'm interested in.

There's a version of this field that went: CPUs got fast, then we made classical engines, then we added neural nets, now GPUs do most of the interesting work. And that arc makes sense commercially and competitively. But I find myself not very interested in the GPU side of things. Not because it's wrong or unimportant, it clearly works, but because I want to see how far pure logic on a CPU can go when you change the underlying assumptions.

NNUE is extraordinary but it's also a black box. The net learns something, it improves position scores, we can verify it's better but we can't say *why* in any satisfying human-readable way. You can probe the activations, you can do ablation studies, but you can't look at a position and trace through why the eval is +1.3. The number comes from a matrix multiply chain that encodes everything and explains nothing.

I want to build something where the reasoning is inspectable. Where you can look at a position, ask "why does the engine think White is better here," and get an answer that points to the structure of the future: "White's moves concentrate around three forced continuations, Black has twelve roughly equal options that are all slightly bad, and the recursive entropy projects that this asymmetry grows over the next four plies."

That's a chess explanation. Not a feature sum, not a net output. An actual description of what the position does to the future.

I'm not interested in winning the GPU arms race. I'm interested in what's possible when you change the question.

## What the Research Claim Actually Is

This is a research project, not just an engine project. The goal is a paper, and the paper needs a precise falsifiable claim.

The claim is: **positional advantage in chess is better modeled as future option asymmetry, quantified by differential continuation entropy, than by any linear or nonlinear combination of static board features.**

That's testable. I'm going to construct specific position classes where this should hold:

*   Zugzwang positions from endgame tablebases (ground truth: one side loses with the move)
    
*   Known fortress positions (ground truth: drawn despite material imbalance)
    
*   Quiet strategic positions from grandmaster games where one side maneuvers for 15-20 moves without material change before converting
    
*   Positions just before tactical combinations, where the entropy asymmetry should spike before the classical score changes
    

For each class, I'll compare the entropy eval against classical feature eval (not NNUE, that's a different comparison) and measure how well each predicts the ground truth outcome.

The hypothesis is that entropy eval will outperform classical feature eval specifically on the position types where future option structure matters most, while performing comparably elsewhere.

The target is the ICGA Journal as primary and arXiv as a preprint as soon as the theoretical framework is tight enough to stand on its own.

## Where I'm Starting

First two weeks: math only. No code.

The formal definition needs to be exact before I write a single function. What exactly is the probability distribution over moves? Softmax with what temperature? Is temperature fixed or position-dependent? What counts as a "move" for entropy purposes: all legal moves, or only moves that survive a shallow quiescence search? How does the recursive depth interact with the search depth in the main alpha-beta loop? Does the transposition table store entropy values separately from eval scores, or can they be combined?

These aren't implementation questions. They're definitional. And if the math is shaky, no amount of code will fix it.

After that: engine. Probably built on top of an existing move generation library so I'm not spending six months reinventing bitboards. The novel part is the eval and the entropy computation. That's where the time goes.

The main implementation risk is computational cost. Entropy computation at each node is more expensive than a classical eval: you need to generate all legal moves, evaluate each one shallowly, compute the softmax, and compute the entropy. On a node you'd spend maybe 10x as long as a classical eval. That kills your nodes per second and limits effective search depth.

The way I'm thinking about handling this: use entropy eval only at nodes above a certain depth threshold, fall back to a fast classical eval at leaf nodes, and use the entropy estimate as a signal for search extension rather than leaf evaluation. A position with extreme entropy asymmetry gets searched deeper automatically. This is analogous to how check extensions work in standard engines: the search structure uses a cheap signal to decide where to look harder.

* * *

I don't know if this works yet. I think it does, but I've been wrong about engine ideas before. What I do know is that nobody has tried it, and after ten years in this space, that's enough reason to try.

The classical engine era ended not because it was exhausted, but because something better came along. I want to find out how much was left.