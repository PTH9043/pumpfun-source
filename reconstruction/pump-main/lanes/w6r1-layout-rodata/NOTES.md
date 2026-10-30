# Lane w6r1-layout-rodata (wave 6, round 1): rodata / data layout for raw

Base: origin/main c2f6d6509 (`programs/pump` identical to 099b0dd: 1027/1035 normalized, 276 raw).
Builds and scratch: `~/.cache/pumpfun-reconstruction/lanes/w6r1-layout-rodata/` (`scripts/trial.sh`
wraps fastbuild + score + the w3-raw-layout data metrics; `scripts/capture.py` captures
`-C no-prepopulate-passes` IR, experiment only).

## Data layout on base (w3-raw-layout metrics)

- 1021 matched windows had every data lddw at the original's in-section offset; 2 did not
  (`0x10c20` TryFromIntError Debug, `0x101448` GlobalVolumeAccumulator deserialize).
- `.data.rel.ro`: 6 non-`.text` slot diffs, all from one swapped pair at 0x220: the original has the
  borsh `src/de/mod.rs:637:21` Location (`ArrayDropGuard::fill_buffer`, `init_count += 1`) before the
  `&()` Debug vtable; ours had the vtable first. w3 attributed it to U-block hash order.
- `.rodata`: 31993/32031 aligned; the only difference is the `creator/admin_cto.rs` path string after
  the `No canonical pump pool for this mint; curve only` msg! literal (both first created inside the
  unmatched `admin_cto::execute`, so it belongs to that window's reconstruction).

## Finding: the swapped pair is a TryFromSliceError Debug artefact, not hash order (`afe4e465`)

Pre-opt IR (`capture.py ... -C no-prepopulate-passes`) shows the `&()` Debug vtable (`vtable.9`) is first
referenced by the cross-crate-inlinable local copy of `<core::array::TryFromSliceError as Debug>::fmt`
(`_ZN67_...`, which sorts before `fill_buffer`'s `_ZN70_$LT$$u5b...`); the next user is
TryFromIntError Debug (`_ZN70_$LT$core..num...`), which sorts after `fill_buffer`. The TFSE fmt mono item
exists only because seven `slice.try_into().unwrap()` sites need `TryFromSliceError: Debug`
(`Result::unwrap` is MIR-inlined, its `E` Debug vtable references the fmt). The fmt itself is dead after
LLVM (all conversions have fixed length), so it never appears in either binary, but the vtable it
created first survives for TryFromIntError. The original therefore has no TFSE Debug mono item.

Change: those seven sites use `.try_into().unwrap_or_default()` (quote_control::state x2,
pump_quote::vault_amount x2, set_reserved_fee_recipients, DistributionCurve::try_deserialize_unchecked x2).
Every one converts a fixed-length range of the right length, so the `Err` arm is unreachable and no
runtime path changes. `.text` differs from base only in the two lddw operands; `.rodata` is unchanged.

| metric | base | this lane |
| --- | ---: | ---: |
| official normalized / raw | 1027 / 276 | 1027 / 276 (0 losses) |
| windows with every data lddw at the original offset | 1021 | **1023** (all checked) |
| `.data.rel.ro` non-`.text` slot diffs (of 1294) | 6 | **0** |
| `.rodata` aligned bytes (of 32031) | 31993 | 31993 |
| model raw, unmatched fixed, full original order, data as built | 1017 | **1019** |
