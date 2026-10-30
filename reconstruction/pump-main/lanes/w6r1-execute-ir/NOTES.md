# Lane w6r1-execute-ir (wave 6, round 1)

Target: `0x1e7c0` `pump_recovered::create_v2::execute` (original 15,576 B, 1,859 insns, 125 calls).
Approach: IR oracle (llc from the SDK on rustc's IR of the target reproduces fastbuild exactly), then
the Rust spelling that makes rustc emit the winning IR.

Base: origin/main 099b0dd (1027/1035 normalized, 276 raw), target 0.80237 at 15,584 B. During the
lane, #2154 (sibling lane w6r1-execute-sem) merged the w4 `g7` form (0.82553, 15,544 B); this branch
is rebased on that main and only adds the two edits below.

## Result

| Build | Bytes | Ratio | Official | Notes |
| --- | ---: | ---: | --- | --- |
| main 099b0dd | 15,584 | 0.80237 | 1027 / 276 | v13 store order |
| g7 (= main after #2154) | 15,544 | 0.82553 | 1027 / 276 | |
| d1 = g7 + exhaustive depth match | 15,560 | 0.82400 | 1027 / 276 | orig `jlt tag,3` form, RA worse |
| **c1b = d1 + explicit `Result` match on `control_account`** | **15,592** | **0.84194** | **1027 / 276, 0 losses** | is_holder_reward=r6, ctx=r7 as orig |

unstripped sha256 `cf4338552a3ad645d895b94f847384a102c985dadfc4e67a099147acba7c834f`.

## Mechanisms

1. **Depth select polarity (d1).** The original computes depth as `jlt tag,3; mov r9,r5`, i.e.
   `select (icmp ult tag,3), 0, payload`. `if let Nested(d)=source {d} else {0}` gives
   `icmp eq tag,3` (`jeq`/`jgt` forms). IR oracle edit `dult` (ult 3 with swapped select operands)
   gives the jlt form; the Rust spelling is the exhaustive match
   `match source {Sol|Whitelisted|Controlled=>0, Nested(d)=>d}`: MIR emits a 4-way switch whose three
   0-arms SimplifyCFG folds into a range check `icmp ult tag,3` (byte-identical to the oracle edit).
   Arm order is irrelevant. Alone it costs 0.0015 (register allocation), but it is the original shape.
2. **Error block placement drives greedy RA (c1b).** Greedy RA runs on the pre-layout block order, so
   slot-index distances (live-range sizes, hence queue priority) depend on where cold blocks sit. An
   IR-mutation hill-climb (`scripts/irsearch.py`: block moves and intra-block instruction moves,
   scored by the llc oracle) found one dominant move: the `Err` arm of `control_account(..)?`
   (`bb91.i`, a 2-instruction "load error ptr; br to return" block) moved anywhere after the
   validate join block `bb43.i` lifts d1 from 0.82400 to 0.84485 and flips the r6/r7 rotation that w4
   identified (is_holder_reward now r6, ctx/accounts r7, as in the original). Positions before
   `bb43.i` give 0.77-0.81. The Rust spelling: replace `?` with an explicit three-arm match
   `match control_account(..) { Ok(Some(r))=>.., Ok(None)=>{walk..}, Err(e)=>return Err(e) }`;
   rustc then emits the Err arm after the Ok(None)/walk blocks. (Arm order of the two Ok arms is
   byte-identical.) A blank `// layout-pad`-style line below is removed to keep every later
   `line!()` immediate unchanged (the first attempt without it lost 6 normalized windows).

Both edits are behaviour-preserving (`?` on `anchor_lang::Result` is `From::from` identity; the depth
match is exhaustive over the same variants).

## Tools

`scripts/`: `build.sh` (fastbuild or IR capture + score + target metric), `capmir.py` (IR/MIR capture,
hooking `fastbuild.run_compiler`), `trial.sh`, `irsearch.py` (IR-mutation hill climb on the llc
oracle; research only). Other helpers are the w4-createv2 scripts with lane paths.

## Ledger

Attempts are recorded under lane `w6r1-execute-ir`, dedupe keys `0x1e7c0-main-099b0dd-*`.
