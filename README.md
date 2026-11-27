# png-squeeze-01 — lossless PNG re-compressor

[![make.fun · png-squeeze-01-run-20261127](https://make.fun/badge/png-squeeze-01-run-20261127.svg)](https://make.fun/bounty/png-squeeze-01-run-20261127)
[![rank](https://make.fun/badge/png-squeeze-01-run-20261127/heavense3nt/png-squeeze-01-run-20261127/rank.svg)](https://make.fun/bounty/png-squeeze-01-run-20261127/leaderboard)
[![reward](https://make.fun/badge/png-squeeze-01-run-20261127/heavense3nt/png-squeeze-01-run-20261127/reward.svg)](https://make.fun/bounty/png-squeeze-01-run-20261127)

Entry for the [make.fun bounty "Squeeze screenshots into the smallest PNG"](https://make.fun/c/png-squeeze-01-run-20261127): one program (`main.ts`, Node 22, standard library only) that rewrites each 1280×720 RGBA screenshot of a set of 8 as a PNG, losslessly, in as few bytes as possible.

## How it works

1. **Decode** the input PNG to RGBA with a built-in decoder (same files the
   checker reads: 8-bit, non-interlaced, colour types 0/2/3/4/6).
2. **Pick the cheapest exact form** per image:
   - palette (type 3) when the image holds at most 256 colours, ordered by
     frequency and by luminance, transparent entries first so `tRNS` stays short;
   - grey (type 0) / grey+alpha (type 4) when every pixel is grey;
   - RGB (type 2) / RGBA (type 6) otherwise;
   - a `tRNS` key instead of an alpha channel when the only non-opaque pixels
     are one fully transparent value no opaque pixel uses.
   Every pixel decodes back exactly, including the colour of fully transparent
   pixels.
3. **Filter search.** For each form, several per-row filterings are built:
   unfiltered, Paeth, minimum-sum, fewest-distinct-pairs, and brute force
   (each row's filter chosen by its real level-4 deflate size given the
   previous 32 KiB). Each is compressed fully at zlib level 9; the smallest wins.
4. **Block-mixed filtering.** For channel forms, the candidate filterings are
   mixed block by block (32 rows): each block takes the candidate whose level-9
   deflate, seeded with the previous 32 KiB of assembled data as dictionary,
   is smallest. The mix is kept only when its full level-9 compression beats
   the best single filtering, so it can never lose.
5. **Final zlib choice.** The winner is re-compressed with a few zlib settings
   (memLevel 8/9, default / filtered strategy); the smallest stream is written.

The run is fully deterministic (fixed work per image, no clock, no randomness,
no network), so the two grading executions produce byte-identical outputs.

## Usage

```sh
node main.ts   # reads $MAKEFUN_INPUT_DIR (default /input), writes $MAKEFUN_OUTPUT_DIR (default /output)
```

## Measured results (kit's own scorer, practice set + 2 fresh generator draws)

| set | score (of 1,000,000) | total bytes | vs scale bytes | vs house baseline |
| --- | --- | --- | --- | --- |
| practice | **712,503** | 550,838 | 0.946× | baseline 420,399 |
| fresh set A | **722,384** | — | — | — |
| fresh set B | **724,742** | — | — | — |

Per instance on the practice set (all 8 exact, lossless verified pixel-by-pixel
by the kit checker):

| instance | theme | scale bytes | this entry | points |
| --- | --- | --- | --- | --- |
| i001 | flat | 13,547 | 12,496 | 718 |
| i002 | rich | 55,063 | 49,297 | 736 |
| i003 | alpha | 107,892 | 103,911 | 691 |
| i004 | grey | 110,521 | 105,573 | 697 |
| i005 | flat | 18,669 | 16,552 | 742 |
| i006 | rich | 185,295 | 177,726 | 694 |
| i007 | alpha | 64,285 | 60,386 | 707 |
| i008 | grey | 26,807 | 24,897 | 714 |

Wall time for a full set of 8: ~23 s CPU on this machine (limit is 60 s wall on
1 CPU in the grader), well inside budget. Two runs on the practice set produce
byte-identical outputs.

## Files

- `main.ts` — the whole entry (decoder, analysis, filter search, block mixing,
  PNG writer). Node built-ins only; nothing to install.
- `LICENSE` — MIT.

## License

MIT — see [LICENSE](LICENSE).
