---
title: Move generation is 1% of the code and 20% of the bugs
published: 2026-08-01
updated: 2026-09-17
description: 'How pseudo legal generation, special case rules, and perft testing turn the smallest module in a chess engine into its most bug-dense foundation'
tags: [Chess,Chess Engine]
image: "../../assets/images/post-covers/move-generation-chess.png"
category: 'Chess'
draft: false 
---

> Cover image source: [civitai.com](https://civitai.com/images/5773325)

Let's imagine a scenario, you got a brand new engine and you write a move generater on a afternoon.Maybe a few hundred ines.Knight offsets in a table, bishop rays that slide until they hit something, pawns that push forward. It looks finished. Then you run perft and depth 3 returns 8,913 or some number instead of 8,902 and you spend the next two days hunting eleven phantom moves that should never have existed.

This is the normal arc. Move generation is the first real module in a chess engine that feels deceptively small. In a modern engine it is a few percent of the codebase and well under ten percent of search time CPU. Yet it is the highest bug density code most engine developers will ever write.Big reason for that is the are almost entirely simple *except* for a half-dozen special cases that interact. Pins meet en passant. Castling meets check. Promotion meets capture. Each rule is a paragraph. Their intersection is where engines silently generate illegal moves for months.

## Every move your engine must generate

Before thinking about legality, enumerate what exists. Six piece types, but wildly unequal complexity.

**King.** One square in any direction, plus castling. The king is the only piece whose move legality depends on squares it does not occupy: the destination and (for castling) the squares it passes through must not be under attack. Everything else about the king is trivial. Those two attack tests are not.

**Queen, rook, bishop.** Sliding pieces. Same loop: step in each direction, emit empty squares, emit the first enemy piece as a capture, stop. The only difference is direction sets. Queen is rook plus bishop. These pieces contribute almost no bugs except through pins, a pinned rook that slides off the pin line is illegal, but the sliding loop itself is correct. If you already have a **Bitboard** and slider attack generation from **Magic Bitboards**, reusing the attack getter for move enumeration keeps the two consistent, which matters more than it sounds.

**Knight.** Eight offsets, bounds checked. No sliding, no special cases. The knight is the only *honest* piece in the engine. If your knight generation is wrong, your board representation is wrong.

**Pawn.** Half the bug surface in one piece. A pawn has five distinct move categories, each with its own precondition:

1. **Single push** - one square forward if empty. White pawns toward rank 8, black toward rank 1. On promotion rank, this push is not one move but four.
2. **Double push** - two squares forward from the starting rank, if *both* the intermediate and target squares are empty. The intermediate square test is the classic omission.
3. **Diagonal captures** - one square diagonally forward onto an enemy piece. Again, four moves if the capture lands on the promotion rank.
4. **En passant** - a diagonal capture onto an empty square, removing an enemy pawn that is not on the destination square, and only immediately after the opponent's double push. The target square is empty. The captured piece is adjacent. Two pawns disappear from one rank. Nothing else in chess behaves like this.
5. **Promotions** - any pawn move that reaches the last rank must offer four promotion choices: **queen, rook, bishop, knight**. A promotion capture is also a capture. Historically, engines that only generated queen promotions passed shallow perft and failed deeper ones where underpromotions block stalemate or deliver check differently.

That is the full piece generation. Leapers (king, knight), sliders (bishop, rook, queen), and one piece that is essentially a state machine. Once you frame pawns this way, perft failures become less mysterious. If depth 5 is off by a few hundred, you are almost certainly missing a pawn special case or applying it one rank too wide.

### What counts as a move ?

A chess engine move is more than source and destination squares. To make and unmake correctly you need:

- from square, to square
- moving piece type and color
- capture piece type (if any) and capture square (which may differ from `to` for en passant)
- promotion piece type (if any)
- flags for double push, en passant capture, castling (kingside/queenside)
- incremental state: castling rights after the move, en passant file after the move, halfmove clock, Zobrist key delta

The move encoding is not incidental. If you omit the capture square for en passant, your unmake will restore the wrong pawn. If you omit the previous en passant file, you cannot correctly restore the position for repetition detection. Design the move struct before optimizing the generator. The debugging you will do is almost entirely make/unmake symmetry checks.

## pseudo legal first, legal second

There are two canonical architectures for move generation.

**Legal generation** emits only moves that leave the king not in check. To do this directly, you must know pins, check status, and pin rays before you emit a single move. You prune a pinned bishop to its pin diagonal, you restrict non king moves when in double check to king moves only, you skip castling when in check without ever generating it. This is efficient, every move you generate is playable, but the generator must solve the hardest constraints up front.

**pseudo legal generation** emits every move that obeys piece movement rules, ignoring whether the king is left in check. Then you filter: make the move, test whether your own king is under attack, discard the move if it is, unmake it. As the [Chess Programming Wiki](https://chessprogramming.org/Move_Generation#legality) puts it, pieces obey their normal rules of movement, but they are not checked beforehand to see if they will leave the king in check. It is left to the move-making function to test.

Almost every didactic engine and most production engines start pseudo legal. The reason correctness before performance.

The pseudo legal plus filter approach is *obviously correct* by construction. If your `isAttacked(kingSquare, them)` function is right, no illegal move survives. You do not need to reason about whether your pin detection covers the en passant double pawn disappearance case correctly at generation time, the make move plus attack test will catch it regardless, at the cost of making and unmaking a few illegal moves per node. That cost is small. In the search, ninety percent of pseudo legal moves are legal anyway. The illegal rate spikes only when in check or when pinned, and even then the waste is a constant factor, not an asymptotic one.

![Position Showcase](../../assets/images/post-content/move-generation-chess/pseudo legal-vs-legal.png)

The alternative is direct legal generation. This is faster in principle because you never make illegal moves, but it replaces a simple correctness invariant with a distributed one. The pin logic in generation, the check-evasion logic in generation, and the `isAttacked` logic must all agree on geometry. When they disagree, you generate an illegal move *and* you have no single filter to catch it. Bugs become silent search errors rather than perft mismatches.

The perft function exposes the difference. A legal generator can bulk count at the leaves:

```python
def perft_legal(board, depth):
    if depth == 0:
        return 1
    if depth == 1:
        return len(board.generate_legal_moves())
    nodes = 0
    for move in board.generate_legal_moves():
        board.make(move)
        nodes += perft_legal(board, depth - 1)
        board.unmake(move)
    return nodes

```

A pseudo legal generator folds the legality test into the loop, which is the classic pattern reproduced on the CPW:

```python
def perft_pseudo_legal(board, depth):
    if depth == 0:
        return 1
    nodes = 0
    for move in board.generate_pseudo_legal_moves():
        board.make(move)
        if not board.is_attacked(board.king_square(board.side_to_move.prev), board.side_to_move):
            # Own king not in check after the move - move was legal
            nodes += perft_pseudo_legal(board, depth - 1)
        board.unmake(move)
    return nodes
```

Note the subtlety: `is_attacked` must be called on the side that just moved, whose king may have moved. Calling it on the wrong color is a bug that passes most positions and fails any with adjacent kings or discovered checks. The pseudo legal loop also reveals the cost: every pseudo legal move is made and unmade at least once. Bulk counting recovers most of the cost by not recursing at `depth == 1`  you return `len(legal_moves)` without making them. Which is the single easiest perft speedup and the only one you need until perft is correct.

## Four special cases that ship bugs

If sliding pieces are the easy 80%, these four are the 20% that generates 80% of the bugs.

### Castling: four preconditions, often checked as two

FIDE castling preconditions are, king and the chosen rook have not moved (castling rights), the king is not currently in check, no square the king passes through is under attack, and no piece occupies the squares between king and rook. The third condition is the one most often implemented incompletely.

The [Chess Programming Wiki](https://chessprogramming.org/Castling#rules) states it precisely. No square between king's start and final square may be controlled by the enemy. For kingside, that is e1 and f1 and g1 not under attack (white). For queenside, e1, d1, c1 not under attack. And the intermediate squares must be empty. Overlooking that c1 must not be attacked on queenside is a common error that perft on Kiwipete will catch. Kiwipete has castling rights still present and pieces arranged to attack those squares.

Castling rights bookkeeping is the deeper trap. Rights are not "has the king moved" alone. They are per-wing and they are lost if:

- the king moves at all,
- the specific rook moves,
- the specific rook is captured, even if the rook has not moved.

That last case is missed often. If white captures black's h8 rook, black's kingside rights evaporate even though black's king never moved. Represent rights as four bits (`KQkq` as in FEN) and update them on every move by masking with precomputed `castling_mask[from]` and `castling_mask[to]` tables. Do not recompute rights from piece positions. A rook that returns to its home square does not regain rights.

A compact, correct legality check (Python, mailbox or bitboard agnostic):

```python
def can_castle_kingside(board, color):
    rights = board.castling_rights
    if color == WHITE and not (rights & WHITE_KING_SIDE):
        return False
    if color == BLACK and not (rights & BLACK_KING_SIDE):
        return False

    king_sq = board.king_square(color)
    # King must not be in check now
    if board.is_attacked(king_sq, opposite(color)):
        return False

    # Squares between must be empty, destination squares not attacked
    # White kingside: f1, g1 must be empty; e1, f1, g1 not attacked
    # (e1 already tested above; f1,g1 remain)
    path_empty, path_safe = castling_path(color, KINGSIDE)
    if any(board.piece_at(sq) is not None for sq in path_empty):
        return False
    if any(board.is_attacked(sq, opposite(color)) for sq in path_safe):
        return False

    return True
```

`castling_path` returns two sets: the squares that must be empty (which includes the rook's transit) and the king's transit squares that must not be attacked. Keep them separate. Queenside has more empty squares than safe squares, b1 must be empty but the king never steps there.

### En passant: the only capture that breaks the destination model

Every other capture removes the piece on the destination square. En passant removes a pawn that is not there. The capturing pawn moves diagonally onto an empty square. The captured pawn sits adjacent on the same rank the capturing pawn left. Two pawns leave one rank. If you model captures as "piece on `to` square," en passant will corrupt the board.

There are three intertwined details:

1. **Representation.** FEN stores the en passant target square. The square *behind* the double pushed pawn, on rank 3 or 6. Your board should store the en passant file (or target square) only when a pawn actually double pushed around an enemy pawn that could capture. Some engines set the ep square unconditionally on any double push. Perft implementations historically do this and compare correctly only because both sides agree, but for hashing correctness you should set it only when an enemy pawn is adjacent. The CPW recommends doing the legality test at double-push time and folding the Zobrist key update there to avoid hashing distinct positions as identical.

2. **Generation.** An en passant capture is pseudo legal if an enemy pawn is adjacent and the ep target square matches. That is trivial.

3. **Legality filtering.** This is where engines fail. An en passant capture can be illegal due to a *horizontal* pin that no other move exhibits. Consider:

```text
8/6bb/8/8/R1pP2k1/4P3/P7/K7  b - d3  (after d2-d4)
```

If black plays `c4xd3` en passant, both the c4 pawn and the d4 pawn disappear from rank 4, opening a rook line against the king. Neither pawn was pinned vertically. The illegality is only visible after both pawns are removed. A generator that checks "is the moving pawn pinned?" but not "does removing *two* pawns from the same rank expose the king?" will accept this illegal capture. The correct pseudo legal filter catches it automatically: make the double disappearance, then test `isAttacked(king)`. A direct legal generator that filters pinned pieces without considering the captured pawn's square will get it wrong.

Implementation shape for make/unmake:

```python
def make_en_passant(board, move):
    # move.from is the capturing pawn, move.to is the ep target (empty)
    board.remove_piece(move.from)
    board.place_piece(move.to, move.piece)
    # Captured pawn is on the same file as move.to, same rank as move.from
    captured_sq = square(move.to.file, move.from.rank)
    captured = board.remove_piece(captured_sq)
    board.ep_capture_square = captured_sq  # for unmake
    return captured

def unmake_en_passant(board, move, captured):
    board.remove_piece(move.to)
    board.place_piece(move.from, move.piece)
    captured_sq = square(move.to.file, move.from.rank)
    board.place_piece(captured_sq, captured)
```

If you forget to save the captured pawn type (it is always a pawn, but save it anyway for symmetry), or you compute `captured_sq` as `move.to` plus/minus one rank, you will swap colors under promotion en-passant adjacent tests and perft will diverge only on the one position that has pawns on the fifth rank with ep rights. Which is exactly Position 3 in the CPW test suite.

### Promotion: four moves where you thought there was one

When a pawn reaches the last rank, the move is not complete until you choose a piece. Perft counts each promotion choice as a distinct leaf. From the start position, promotions first appear at depth 7+ (or earlier in tactical test positions like Position 4, where underpromotions appear at depth 2). If you only generate queen promotions, perft will match at low depth and drift at mid depth, which is the most confusing failure mode because you will blame en passant or castling first.

Generate promotions as an expansion of pawn pushes and pawn captures that land on the promotion rank:

```python
PROMOTION_PIECES = [QUEEN, ROOK, BISHOP, KNIGHT]

def generate_pawn_moves(board, sq, color):
    moves = []
    forward = 1 if color == WHITE else -1
    start_rank = 1 if color == WHITE else 6
    promo_rank = 7 if color == WHITE else 0

    # Single push
    to = sq + forward * 8  # assuming 0x88 or linear mailbox; adapt for bitboards
    if board.is_empty(to):
        if to.rank == promo_rank:
            for promo in PROMOTION_PIECES:
                moves.append(Move(sq, to, promotion=promo))
        else:
            moves.append(Move(sq, to))
            # Double push, only from start rank, and only if intermediate empty
            if sq.rank == start_rank:
                to2 = to + forward * 8
                if board.is_empty(to2):
                    moves.append(Move(sq, two, flag=DOUBLE_PUSH))

    # Captures
    for df in (-1, 1):
        cap = square(sq.file + df, sq.rank + forward)
        if not on_board(cap):
            continue
        is_capture = board.piece_at(cap) is not None and board.color_at(cap) != color
        is_ep = cap == board.en_passant_target
        if is_capture or is_ep:
            if cap.rank == promo_rank:
                for promo in PROMOTION_PIECES:
                    # Capture-promotion: both a capture and a promotion
                    moves.append(Move(sq, cap, promotion=promo, capture=is_capture))
            elif is_capture or is_ep:
                moves.append(Move(sq, cap, capture=is_capture, en_passant=is_ep))

    return moves
```

Three notes on this snippet. First, the double push checks the intermediate square `to` - the `is_empty(to)` guard above, and *then* `to2`. Collapsing that into a single "is `to2` empty?" test is wrong. Second, promotion captures where the target is occupied and ep is possible on the same diagonal are distinct; generate both if both conditions hold (rare, but legal). Third, the queen was not always the answer: underpromotion to knight can give check where queen promotion does not, or avoid stalemate. The engine does not get to choose. It must generate all four.

### Double pawn push: the smallest bug with the widest blast radius

The double push rule is one sentence and one bug: the pawn moves two squares forward from its starting rank if both the target square and the square between are empty. The fix is equally small: check both. The blast radius is large because a missed intermediate check creates phantom moves that enable en passant possibilities that never should have existed, and perft divergence compounds exponentially. If perft at depth 1 matches but depth 3 diverges and the delta is a multiple of en passant counts, check double pushes first. It is the cheapest fix with the highest leverage.

## Pins and check: constraints, not generators

A pinned piece cannot move off the pin line, except along it or to capture the pinner. A king in check must get out of check. A common instinct is to write special generators for each case. A pin aware slider loop, a check-evasion generator that knows the three responses (move the king, capture the attacker, block the ray). Those are valid optimizations. They are not required for correctness, and they are a common source of bugs when they become the *only* path.

The pseudo legal approach treats both as constraints filtered by the same invariant.

### The expensive-but-correct filter

```text
generate all pseudo legal moves
for each move:
    make(move)
    if not isAttacked(ownKingSquare, opponent):
        keep move
    unmake(move)
```

That is the whole pin and check implementation. A bishop pinned on the a2-g8 diagonal that tries to move off diagonal will leave the king in check after the make. It is discarded. A move that does not resolve a check also leaves the king in check and is discarded. Double check where the king is attacked by two pieces simultaneously, falls out naturally: the only pseudo legal moves that pass the filter are king moves to safe squares, because any block or capture can at most address one attacker.

This is expensive in the sense that you make illegal moves you will discard. It is not expensive in practice once perft passes. Perft enumerates every node. Search does not. Alpha beta plus move ordering means most illegal moves in check positions are never generated because the search terminates early or the check-evasion path prunes. And even in perft, the illegal fraction is modest: in the initial position at depth 5, zero moves are illegal due to pins because no pins exist yet. In Kiwipete, where the position is deliberately dense with pins and checks, the illegal rate is still well under half.

### When to generate directly

There are two legitimate reasons to generate legal moves directly:

1. **Check evasion at search time.** When in check, only a tiny subset of pseudo legal moves are legal. Generating only king moves, captures of the attacker, and blocks of the ray (if the attacker is a slider) avoids making dozens of doomed moves. The CPW notes that special generators for getting out of check can be more efficient than generating and testing each possible move. This is true and worth doing once the engine is correct.

2. **Evasions plus attack-map reuse.** With bitboards, you already compute attacks to the king to detect check. Reusing the pinner bitboard and the check-ray bitboard to mask move generation is elegant. The CPW's `Checks and Pinned Pieces (Bitboards)` page shows the pattern:

```c
U64 attacksToKing(Square kingSq, Color kingColor) {
    U64 opPawns   = pieceBB[nBlackPawn   - kingColor];
    U64 opKnights = pieceBB[nBlackKnight - kingColor];
    U64 opRQ = pieceBB[nBlackQueen - kingColor] | pieceBB[nBlackRook - kingColor];
    U64 opBQ = pieceBB[nBlackQueen - kingColor] | pieceBB[nBlackBishop - kingColor];
    return (pawnAttacks[kingColor][kingSq] & opPawns)
         | (knightAttacks[kingSq]           & opKnights)
         | (bishopAttacks(occupied, kingSq) & opBQ)
         | (rookAttacks(occupied, kingSq)   & opRQ);
}
```

Pinned pieces then fall out of x-ray attacks:

```c
pinned = 0;
pinner = xrayRookAttacks(occupied, ownPieces, kingSq) & opRQ;
while (pinner) {
    Square sq = popLSB(&pinner);
    pinned |= obstructed(sq, kingSq) & ownPieces;
}
pinner = xrayBishopAttacks(occupied, ownPieces, kingSq) & opBQ;
while (pinner) {
    Square sq = popLSB(&pinner);
    pinned |= obstructed(sq, kingSq) & ownPieces;
}
```

The direction disjoint variant (DirGolem) does the same with ray intersections. Both are correct. Both are optimizations of the same invariant: a move is legal if the king is not attacked after it. If you implement either, keep the pseudo legal filter as an assertion in debug builds:

```python
assert set(generate_legal_direct(board)) == set(filter_pseudo_legal(board))
```

Run that assertion under perft. It is the cheapest way to catch a divergence between two implementations of the same geometry.

### Check types and their consequences

- **Single check.** Three responses: move the king to a non attacked square, capture the attacker (with a piece that is not absolutely pinned off the capture line), or interpose on the ray between attacker and king if the attacker is a sliding piece. A direct generator must handle all three. The pseudo legal filter handles them implicitly.

- **Double check.** Only king moves. This is not an optimization; it is a rule. No block or capture can address two attackers at once. A common bug is generating interpositions during double check because the "block the ray" code does not check the attacker count. Perft Position 5 is designed to expose this.

- **Discovered check.** The checker is not the piece that moved but the slider behind it. Detection by last move, testing whether the origin square was on a ray from the king is the cheapest scalar approach. With bitboards, recomputing `attacksToKing` is branchless and reuses the slider attack tables, so the per-node cost is negligible.

## Perft: your ground truth

Perft: performance test, move-path enumeration is the most valuable test in engine programming. It is not a benchmark, though it can be used as one. It is a correctness oracle.

**Definition.** Perft(n) is the number of leaf nodes at depth n when you recursively generate strictly legal moves and count leaves. Nodes are counted only at the bottom. Draws by repetition, fifty-move rule, and insufficient material are ignored. Perft stops only at checkmate or stalemate, not at theoretical draws. This makes perft trees not identical to search trees, but close enough to be decisive for move generation. The stability of the definition is what makes published tables useful: everyone counts the same way.

**The function.** The CPW gives it in three lines of C:

```c
uint64_t Perft(int depth) {
    Move moves[256];
    int n = GenerateLegalMoves(moves);
    if (depth == 1) return n; // bulk counting
    uint64_t nodes = 0;
    for (int i = 0; i < n; i++) {
        MakeMove(moves[i]);
        nodes += Perft(depth - 1);
        UndoMove(moves[i]);
    }
    return nodes;
}
```

The bulk-counting variant, returning `n` at `depth == 1` without making the leaf moves is significantly faster and better indicates raw generator speed versus make/unmake overhead. For debugging, the non-bulk version that counts at `depth == 0` is simpler to instrument for captures, checks, and en passant breakdowns. Keep both. Use bulk counting for speed, non-bulk for breakdowns.

For pseudo legal generators, the loop is:

```c
uint64_t Perft(int depth) {
    Move moves[256];
    int n = GenerateMoves(moves);
    if (depth == 0) return 1;
    uint64_t nodes = 0;
    for (int i = 0; i < n; i++) {
        MakeMove(moves[i]);
        if (!IsInCheck())
            nodes += Perft(depth - 1);
        UndoMove(moves[i]);
    }
    return nodes;
}
```

The difference, making illegal moves and filtering by `IsInCheck()` is why pseudo legal perft makes and unmakes twice for illegal branches in some implementations. The CPW's discussion of hashing and speedups matters only after correctness.

### Reference tables you can trust

The single most useful debugging asset is a set of published perft numbers for positions deliberately chosen to exercise different rule interactions. The CPW's Perft Results page is the canonical source. Do not invent numbers. Do not approximate. Compare exactly.

**Initial position** : `rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1`

| Depth | Nodes | Captures | En passant | Castles | Promotions | Checks |
| ------: | ------: | ---------: | -----------: | --------: | -----------: | -------: |
| 1 | 20 | 0 | 0 | 0 | 0 | 0 |
| 2 | 400 | 0 | 0 | 0 | 0 | 0 |
| 3 | 8,902 | 34 | 0 | 0 | 0 | 12 |
| 4 | 197,281 | 1,576 | 0 | 0 | 0 | 469 |
| 5 | 4,865,609 | 82,719 | 258 | 0 | 0 | 27,351 |
| 6 | 119,060,324 | 2,812,008 | 5,248 | 0 | 0 | 809,099 |

Higher depths verified by Steven Edwards (Symbolic) through depth 13 and Ankan Banerjee through depth 15 exist but are not needed for move-gen debugging. Depth 5 or 6 from the start position is sufficient to catch most bugs; the breakdown columns (captures, ep, checks) pinpoint which category is wrong.

**Kiwipete** : `r3k2r/p1ppqpb1/bn2pnp1/3PN3/1p2P3/2N2Q1p/PPPBBPPP/R3K2R w KQkq - 0 1`  by Peter McKenzie. Dense with promotions, castling, and pins.

| Depth | Nodes | Captures | En passant | Castles | Promotions |
| ------: | ------: | ---------: | -----------: | --------: | -----------: |
| 1 | 48 | 8 | 0 | 2 | 0 |
| 2 | 2,039 | 351 | 1 | 91 | 0 |
| 3 | 97,862 | 17,102 | 45 | 3,162 | 0 |
| 4 | 4,085,603 | 757,163 | 1,929 | 128,013 | 15,172 |
| 5 | 193,690,690 | 35,043,416 | 73,365 | 4,993,637 | 8,392 |

If your engine passes the initial position but fails Kiwipete, the bug is almost certainly castling through check, promotion undercount, or a pin that only manifests when the board is crowded.

**Position 3** : `8/2p5/3p4/KP5r/1R3p1k/8/4P1P1/8 w - - 0 1`  the en passant torture test. Almost every en passant bug, including the double-disappearance discovered check, is exposed here.

| Depth | Nodes | En passant |
| ------: | ------: | -----------: |
| 1 | 14 | 0 |
| 2 | 191 | 0 |
| 3 | 2,812 | 2 |
| 4 | 43,238 | 123 |
| 5 | 674,624 | 1,165 |
| 6 | 11,030,083 | 33,325 |

**Position 4** : `r3k2r/Pppp1ppp/1b3nbN/nP6/BBP1P3/q4N2/Pp1P2PP/R2Q1RK1 w kq - 0 1` promotions and underpromotions with checks.

| Depth | Nodes | Promotions |
| ------: | ------: | -----------: |
| 1 | 6 | 0 |
| 2 | 264 | 48 |
| 3 | 9,467 | 120 |
| 4 | 422,333 | 60,032 |

**Position 5** : `rnbq1k1r/pp1Pbppp/2p5/8/2B5/8/PPP1NnPP/RNBQK2R w KQ - 1 8`  discovered checks and double checks. Caught bugs in engines several years old at depth 3.

| Depth | Nodes |
| ------: | ------: |
| 1 | 44 |
| 2 | 1,486 |
| 3 | 62,379 |
| 4 | 2,103,487 |
| 5 | 89,941,194 |

**Position 6** : `r4rk1/1pp1qppp/p1np1n2/2b1p1B1/2B1P1b1/P1NP1N2/1PP1QPPP/R4RK1 w - - 0 10`  a symmetrical stress test with many captures.

| Depth | Nodes |
| ------: | ------: |
| 1 | 46 |
| 2 | 2,079 |
| 3 | 89,890 |
| 4 | 3,894,594 |

A seventh, often overlooked, is the ep-discovered-check position from the CPW's en passant discussion, the one with two pawns on the fifth rank and rooks aligned. Keep it in your test suite as a dedicated regression, not just as one node inside a larger perft.

----

### Divide: how to isolate the first bad move

Perft tells you *that* you are wrong. Divide tells you *where*.

Divide enumerates the legal moves at the root and prints the perft of each subtree. Stockfish's `go perft 5` on the start position prints 20 lines, one per first move, with node counts like `a2a3: 181046` and `e2e4: 405385`. When your perft diverges, compare your divide output to a reference (Stockfish, qperft by Harm Geert Muller, or `vaulet/perft.txt`). Find the first move where the counts differ. That move's subtree contains the bug.

Iterate: make that move on the board, run perft at `depth - 1` from the resulting position, divide again, and follow the divergent branch. Within a handful of steps you converge on a leaf position where perft(1) is wrong, that is, where your move list and the reference move list differ by a single move. At that point the bug is visible: a missing en passant capture, an extra illegal castling, a promotion you did not generate, or a pinned piece that moved off the line.

The workflow is mechanical:

1. Run `perft(n)` on a reference position. If it matches, go deeper or switch positions.
2. On mismatch, run `divide(n)` and diff against a trusted divide.
3. Recurse into the first divergent move's position with `perft(n-1)`.
4. When `n == 1`, compare move lists directly. The symmetric difference is the bug.

Do not try to debug by staring at generation code. Follow the divide. It turns a tree of billions of nodes into a binary search over depth.

### A perft harness you can ship

Perft is not a one-off script. It is a permanent regression. Wire it as a test, not a manual command.

```python
import chess  # for FEN parsing only; generation is yours

REFERENCE = {
    # (fen, depth): expected_nodes
    ("rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1", 5): 4865609,
    ("r3k2r/p1ppqpb1/bn2pnp1/3PN3/1p2P3/2N2Q1p/PPPBBPPP/R3K2R w KQkq - 0 1", 4): 4085603,
    ("8/2p5/3p4/KP5r/1R3p1k/8/4P1P1/8 w - - 0 1", 5): 674624,
    ("r3k2r/Pppp1ppp/1b3nbN/nP6/BBP1P3/q4N2/Pp1P2PP/R2Q1RK1 w kq - 0 1", 4): 422333,
    ("rnbq1k1r/pp1Pbppp/2p5/8/2B5/8/PPP1NnPP/RNBQK2R w KQ - 1 8", 4): 2103487,
}

def test_perft():
    for (fen, depth), expected in REFERENCE.items():
        board = Board.from_fen(fen)
        result = board.perft(depth)  # your implementation
        assert result == expected, f"perft({depth}) on {fen}: got {result}, expected {expected}"

def divide(board, depth):
    """Print per-move subtree counts for diffing against Stockfish/qperft."""
    for move in board.generate_legal_moves():
        board.make(move)
        nodes = board.perft(depth - 1)
        board.unmake(move)
        print(f"{move.uci()}: {nodes}")
```

Run this in CI. Add the ep-discovered-check regression position explicitly. When a future refactor breaks castling rights, you want the failure in a pull request, not in a tournament game where the engine castles through check.

Verification with trusted tooling is also worth doing once: Stockfish's perft and Muller's qperft should agree on every reference position. The CPW's perft results page includes the exact node counts and breakdowns; `vajolet/perft.txt` on GitHub and `perft-random.epd` by Marcel van Kervinck provide additional random positions for fuzzing.

## Making it fast, only after it is correct

Robert Hyatt's observation still holds: move generation is rarely more than ten percent of total search time. A twenty percent faster generator is a two percent faster engine. The order of operations is therefore: correct, then complete, then fast. Perft is the gate between the first two and the third.

That said, the performance design space is worth understanding, because the same attack-map infrastructure you build for fast move generation serves evaluation, king safety, and check detection.

### Attack maps and incremental update

`isAttacked(square, color)` is the most called function in the engine. In perft it is called once per pseudo legal move. In search it is called at every node to test legality and to decide extensions. Its implementation dominates move-gen cost.

Three approaches, in increasing sophistication:

1. **Recompute on the fly.** For each query, generate attacks from the target square outward and intersect with enemy pieces. With bitboards and magic bitboards (see [[Magic Bitboards]]), bishop and rook attacks are a table lookup plus a few bitwise ops. Knight and pawn attacks are array lookups. King attacks are an array lookup. The total is a handful of 64-bit operations and no persistent state. For most engines this is fast enough and trivially correct. No incremental update to maintain, no attack map to invalidate on make/unmake.

2. **Incrementally updated attack maps.** Maintain a board-wide `attackedBy[color]` bitboard updated on every make/unmake. Query becomes `attackedBy[them] & (1 << square)`. Make/unmake cost increases. Every move updates the map, but query cost drops to one AND. This wins only if queries vastly outnumber updates, which they do not in perft (one query per move, one update per make). In search the ratio is more favorable because evaluation queries the same map many times. If you go incremental, the perft-speed comparison must be done with bulk counting disabled, or you are measuring the wrong thing.

3. **Last-move check detection.** Instead of recomputing attacks to the king, test whether the last move gave check by checking direct attack from `move.to` and discovered attack from `move.from`. This avoids a full `isAttacked` at the cost of branching logic for en passant and promotion. With bitboards the savings are negligible and the branchless `isAttacked` is simpler. The CPW recommends the on-the-fly approach for bitboard engines for exactly this reason: you already compute rook and bishop attacks for move generation, so reusing them for check detection costs almost nothing.

### Bulk counting and hashing

Two perft-specific speedups dominate reported nodes-per-second numbers. Neither affects search.

- **Bulk counting** : at `depth == 1`, return `len(generateLegalMoves())` without making the moves. This skips the leaf make/unmake entirely. It is valid only with a legal generator (or with a pseudo legal generator that filters before counting). It roughly doubles speed and is the standard when comparing generator throughput.

- **Transposition hashing** : memoize perft results by position key (Zobrist). Hashing can speed perft by 1.5x to 4x but introduces a small chance of collision-based miscount. It is also a useful sanity check for Zobrist key correctness: hash perft results and compare with and without hashing, if they differ, the key has a bug, often around en passant or castling rights. The CPW notes this use explicitly.

Both are measurement confounders. When someone reports "1.5 giganodes per second per core" (Gigantua, 2021) or "4 billion nodes per second" (2025 optimization threads), the setup, bulk counting on or off, hashing on or off, language, CPU changes the number by multiples. Compare perft speed only against identical configurations.

### Magic bitboards and slider generation

```python
rook_attacks = rook_table[square][(occupied * rook_magic[square]) >> rook_shift[square]]
bishop_attacks = bishop_table[square][(occupied * bishop_magic[square]) >> bishop_shift[square]]
```

Reuse the same tables for move generation and for `isAttacked`. Do not write a second ray-marcher for one and a magic lookup for the other. The fastest way to introduce a perft bug is to have two implementations of the same geometry that disagree on edge cases. Blockers on the edge of the board, occupied squares at the attack origin.

For move enumeration, sliders contribute a bounded number of moves (at most 14 for a queen on an open board, fewer in practice). The generator loops over attack bitboards by popping least-significant bits:

```python
attacks = rook_attacks(square, occupied) & ~own_pieces
while attacks:
    to = pop_lsb(attacks)
    if board.piece_at(to):
        moves.append(Move(square, to, capture=True))
    else:
        moves.append(Move(square, to))
```

Pawns, again, are the exception: their move set depends on occupancy of specific neighboring squares, not on a precomputed attack table alone. Write pawn generation as explicit square tests, not as a table lookup intersected with occupancy, so the double-push intermediate check and the promotion expansion remain visible.

### Order-of-magnitude expectations

You do not need to hit a target nps to ship. But for calibration:

- A Python pseudo legal generator with mailbox or simple bitboard should do perft(5) from the start position (4.8M nodes) in seconds, not minutes. If it takes minutes, the bottleneck is usually Python loops over 64 squares per piece rather than the attack computation.

- A Python generator with `python-chess` as a reference comparison will be slower than a C++ engine by 100-1000x on raw perft. That is fine. Perft in Python is for correctness, not for benchmarking search speed.

- Depth scales exponentially. Perft(6) from the start position is 119M nodes, roughly 24x perft(5). Perft(7) is 3.1B. Do not run perft(7) from Python without bulk counting as a smoke test. Use perft(5) plus Kiwipete 4 and Position 3 depth 5 as the standard regression. They cover the rule interactions without requiring hours.

## What to measure before optimizing

The article's title is deliberately provocative. Move generation is 1% of the code and 20% of the bugs, but the per-unit density is higher than that. In a 10k-line engine, move generation plus make/unmake is often 300-500 lines. The bug list from a decade of TalkChess perft threads is almost invariant:

| Symptom | Divide points to | Usual cause |
| --------- | ----------------- | ------------- |
| Perft off by small constant at depth 3+ | A single first move's subtree | Missing pawn double-push intermediate check, or missing queenside castling-through-check test |
| En passant count wrong (or zero when expected) | Position 3, any depth with ep | Captured pawn not removed from correct square; make/unmake asymmetry; ep square set on wrong rank |
| Castling count wrong | Kiwipete depths 2+ | Rights not cleared on rook capture; king-through-check not tested; path not cleared of pieces |
| Promotion count wrong or zero | Position 4 depth 2+ | Only queen promotions generated; capture-promotions not expanded to 4 |
| Perft matches shallow, diverges deep | Kiwipete depth 5 or Position 5 depth 4 | Pinned piece moving off pin line (pseudo legal filter missing or `isAttacked` called on wrong color); double-check not restricted to king moves |
| Divide diffs alternate signs | Multiple subtrees off in opposite directions | `isAttacked` disagreement between generator and filter, direct legal generator and pseudo legal filter using different attack tables |

Every row in that table is a real engine bug that survived to a release, because perft was not in CI.

## Ship the generator and the test that proves it

The foundation of a chess engine is two artifacts: a move generator you can reason about and a perft harness you trust.

Choose pseudo legal generation with a make-then-test filter. It is not the fastest possible design. It is the design with the smallest gap between "looks correct" and "is correct." When you can enumerate perft(5) on five reference positions without a single node of error, you have proven that board representation, move encoding, make/unmake symmetry, attack detection, and special-case rules all agree. No other test in engine programming gives you that coverage per line of test code.

Then, and only then, optimize. Cache attack maps if profiling shows `isAttacked` dominates. Switch check evasions to a direct generator if move legality in check is on the hot path. Adopt staged move generation hash move first, then captures, then killers, then quiets to avoid generating moves that will be pruned, as the CPW's staged generation pattern describes. Each of those is a performance win that preserves correctness, because the invariant legal means the king is not attacked after the move still holds and still has a test.

## References and Further Reading

| Resource | Type | Why it matters |
| --- | --- | --- |
| [Move Generation : Chess Programming Wiki](https://www.chessprogramming.org/Move_Generation) | Documentation | Canonical pseudo legal vs. legal taxonomy, staged and chunk generation patterns |
| [Perft : Chess Programming Wiki](https://www.chessprogramming.org/Perft) | Documentation | Definition, bulk counting, pseudo legal perft pattern, divide, purposes and known issues |
| [Perft Results : Chess Programming Wiki](https://www.chessprogramming.org/Perft_Results) | Reference table | Verified node counts and breakdowns for initial position, Kiwipete, Positions 3-6 |
| [Castling : Chess Programming Wiki](https://www.chessprogramming.org/Castling) | Documentation | Four preconditions, Chess960 distinctions, rights handling |
| [En passant : Chess Programming Wiki](https://www.chessprogramming.org/En_passant) | Documentation | Target-square model, legality test, the double-disappearance discovered-check trap, IsiChess anecdote |
| [Pin : Chess Programming Wiki](https://www.chessprogramming.org/Pin) | Documentation | Absolute vs. partial vs. relative pins, relevance to legal generation |
| [Check : Chess Programming Wiki](https://www.chessprogramming.org/Check) | Documentation | Single vs. double check, detection by last move vs. attack tables, evasion constraints |
| [Checks and Pinned Pieces (Bitboards) - Chess Programming Wiki](https://www.chessprogramming.org/Checks_and_Pinned_Pieces_%28Bitboards%29) | Documentation | Bitboard attacksToKing and pinned-piece detection via x-ray attacks |
| [Stockfish : perft command](https://github.com/official-stockfish/Stockfish) | Source / Tool | `go perft` and `go perft <depth>` as the practical divide reference; also qperft by Harm Geert Muller |
| [vajolet/perft.txt : Marco Belli](https://github.com/elcabesa/vajolet/blob/master/perft.txt) | Reference data | Additional perft positions for fuzzing beyond the CPW set |
| [perft-random.epd : Marcel van Kervinck](https://www.chessprogramming.org/Perft_Results) | Reference data | Random EPD positions linked from CPW for stress-testing generators |
