# Lane w6r1-buyexactquote-sem (wave 6, round 1)

Target `0x1489f8` `trade::buy_exact_quote` (v3), original 7856 B. Base origin/main 099b0dd30 (1027 normalized / 276 raw):
7768 B, ratio 0.66033 (llc-oracle masked 0.8928), 66/66 calls.

Approach: assembly-first semantic reconstruction. Tools (lane-local, under `~/.cache/pumpfun-reconstruction/lanes/w6r1-buyexactquote-sem/scripts`):
`calls.py` (listing with callees named; original callees named through the score windows), `cfg.py` (block summary with
callees, Source lines and successors), `blocks.py` (block matching orig vs candidate, `DIFF=1` per-block masked diffs),
plus the w4 lane's trial/ir/hc scripts (capture hook updated for `fastbuild.run_compiler`).

## Block-level reading of the original

Callee multisets already agree. The original lays out the whitelist split as
`memcmp -> [non-whitelist: net_quote_for_budget, tokens_for_quote(net), try_from] -> [891 err -> EXIT] -> [whitelist: tokens_for_quote(available)] -> merge -> tokens>real`,
while base placed the 891 error right after net_quote_for_budget and the whitelist arm before the non-whitelist
tokens_for_quote. The merge block in the original moves fee values that were kept as one group, which suggested the
non-whitelist fees come back as one `TradeFees` value rather than four scalars.

## Trials (ledger keys `0x1489f8-main-099b0dd-*-v1`)

| trial | mechanism | bytes | ratio | masked | result |
| --- | --- | ---: | ---: | ---: | --- |
| base | main 099b0dd | 7768 | 0.6603 | 0.8928 | |
| b01 | carry the fees as a `TradeFees` value through the whitelist if/else: `let (tokens,net,net_fees)=if wl{(tokens,available,fees)}else{let (net,net_fees)=net_quote_for_budget(..)?;..}` | 7888 | 0.6876 | 0.8344 | top layout now orig-like, ctx in r6; lower half worse |
| b02 | b01 + destructure net_fees in the noncross arm | | = b01 | | identical |
| **b04** | b01 with the tuple ordered `(net_fees,tokens,net)` | 7864 | **0.8430** | **0.9617** | kept |
| b05 | b01 with `(net,tokens,net_fees)` | | = b01 | | identical |

b04 official: 1027 normalized / 276 raw, no gains or losses, unstripped sha256 59fc0aea6cd98245f62fc71d37ef2d1f8321f3c321e8e742435df9aaa5473f01.
No runtime behaviour change: the same values flow through a struct instead of four scalars.
