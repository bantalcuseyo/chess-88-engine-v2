# chess-88-engine-v2

A chess engine in one file: `main.py` is 3,826 bytes of Python 3 (stdlib only),
a readable 0x88 engine compressed into an lzma/base85 payload and unpacked at
start-up. Written for the tiny-chess-01 line protocol.

## Protocol

Reads newline-delimited JSON on stdin and writes exactly one reply line per
line read:

```
hello                                   -> {"ready":true}
{"seq","game","fen","legal_moves",...}  -> {"seq":<same>,"move":"<uci>"}
                                           {"seq":<same>,"resign":true}   (no legal move)
```

The chosen move is always cross-checked against `legal_moves`; if the search
move is absent, `legal_moves[0]` is played. Deterministic: a fixed node budget
per move, no clock, no randomness. Nothing is printed except replies.

## Engine

- 0x88 board (`list[int]` of 128), pieces 1..6 white / negative black.
- Incremental evaluation: material + piece-square table, updated inside
  `make`/`unmake` by table lookup (`sc` is always White's view).
- Move generation: pawn pushes and double pushes, captures, queen and knight
  promotions, knight/king offsets, sliding rays, castling with the
  attacked-square test. Legality by make/test/unmake.
- Search: iterative deepening to 200,000 nodes per move, PVS (first move full
  window, others null window), quiescence over captures and promotions,
  killer moves per ply, MVV-LVA-ish ordering, depth cap 6. Only a completed
  depth is accepted, so a partial iteration never overwrites the best move.
  Mate is `29000 - ply`, stalemate 0.

## Verification (this commit)

| check | result |
| --- | --- |
| perft startpos 1-3 | 20 / 400 / 8902 exact |
| perft kiwipete 1-3 | 48 / 2039 / 97862 exact |
| perft positions 3,4,5,6 | match reference with rook/bishop underpromotions excluded |
| fuzz vs python-chess | 240 positions, legal sets and resulting FENs equal |
| time, 200k nodes | 1.8-2.3 s per move on one VPS core |
| size | `main.py` 3,826 B (cap 4,096), tree < 8,192 B |

Rook and bishop underpromotions are intentionally not generated, which is the
only difference from a full move generator.

## Use

```sh
printf 'hello\n{"seq":1,"game":1,"fen":"rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1","legal_moves":["e2e4","d2d4"],"status":"your_move"}\n' | python3 main.py
```

```
{"ready":true}
{"seq":1,"move":"e2e4"}
```

The second reply shows the guard at work: the searched move was not in the
supplied list, so `legal_moves[0]` was played.

## License

MIT, see `LICENSE` (SPDX-License-Identifier: MIT).
