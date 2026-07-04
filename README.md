# perl-substr-bench

Benchmark and bisection to identify the Perl release and commit which
introduced the substr/UTF-8 position cache regression.

Background (Japanese): https://qiita.com/kaz-utashiro/items/d7047abb8531d5afc40a

Reported: https://github.com/Perl/perl5/issues/24531

## Benchmark

100 substr() calls at increasing positions on a 20M-char UTF-8 string
(see [bench.yml](.github/workflows/bench.yml)).  Time is the best of
3 runs on ubuntu-latest.

## Results

[Full run](https://github.com/kaz-utashiro/perl-substr-bench/actions/runs/28644075592)

| perl | time (sec) | verdict |
|---|---:|---|
| 5.32.1 | 0.0554 | ok |
| 5.34.0 | 0.0555 | ok |
| 5.34.1 | 0.0554 | ok |
| 5.34.2 | 0.0551 | ok |
| 5.34.3 | 0.0552 | ok |
| 5.36.0 | 1.4468 | **SLOW** |
| 5.36.1 | 1.4479 | **SLOW** |
| 5.36.2 | 1.4444 | **SLOW** |
| 5.36.3 | 1.5519 | **SLOW** |
| 5.38.0 | 1.4066 | **SLOW** |
| 5.38.1 | 1.5910 | **SLOW** |
| 5.38.2 | 1.4056 | **SLOW** |
| 5.38.3 | 1.4324 | **SLOW** |
| 5.38.4 | 1.2352 | **SLOW** |
| 5.38.5 | 1.5915 | **SLOW** |
| 5.40.0 | 1.4459 | **SLOW** |
| 5.40.1 | 1.4502 | **SLOW** |
| 5.40.2 | 1.4474 | **SLOW** |
| 5.40.3 | 1.4454 | **SLOW** |
| 5.40.4 | 1.4451 | **SLOW** |
| 5.42.0 | 1.4326 | **SLOW** |
| 5.42.1 | 1.4741 | **SLOW** |
| 5.42.2 | 1.4770 | **SLOW** |
| 5.44.0-RC1 (built from source) | 1.4817 | **SLOW** |
| blead 2026-07-03 (built from source) | 1.6005 | **SLOW** |

The regression was introduced between 5.34 and 5.36.

## Bisection

`Porting/bisect.pl` between v5.34.0 and v5.36.0
(see [bisect.yml](.github/workflows/bisect.yml),
[run log](https://github.com/kaz-utashiro/perl-substr-bench/actions/runs/28645286418)):

```
e6e9dd290698d47a0db9e1d676d2b82e0bb0a52b is the first bad commit

    Do not cache utf8 offsets for non-canonical lengths

    In particular, if the length is beyond the end, it should not be
    stored as the end.
```

https://github.com/Perl/perl5/commit/e6e9dd290698d47a0db9e1d676d2b82e0bb0a52b
(perl/perl5#18727, first shipped in 5.36.0)

The same code structure is present in blead as of 2026-07: results
computed via `S_sv_pos_u2b_midway` never set `canonical_position`,
so they are never cached.  Verified empirically by building blead and
v5.44.0-RC1 from source ([run](https://github.com/kaz-utashiro/perl-substr-bench/actions/runs/28648001769)): both are still affected.

## Related: `@-` / `@+` performance (separate issue)

The byte-to-char direction (`sv_pos_b2u`, used when reading `@-` /
`@+` after a match on a utf8 string) is a separate, much older
performance issue.  Measured with 12k matches
([run](https://github.com/kaz-utashiro/perl-substr-bench/actions/runs/28709426205),
see [atmark.yml](.github/workflows/atmark.yml)):

| perl | `@-`/`@+` (sec) | pos() (sec) | ratio |
|---|---:|---:|---:|
| 5.12.5 | 1.45 | 0.153 | 9x |
| 5.14.4 – 5.16.3 | 2.9 – 3.3 | 0.15 | 20x |
| 5.18.4 | 6.5 | 0.068 | 95x |
| 5.20.3 – 5.36.3 | 5.7 – 8.4 | 0.003 | ~2000x |
| 5.38.0 – 5.42.2 | 0.46 – 0.57 | 0.003 | 150 – 190x |

pos() was also affected in the 5.12–5.16 era, was half-fixed in 5.18
and fully fixed in 5.20.  `@-`/`@+` never was: it got slower in 5.14
and again in 5.18, improved ~12x in 5.38.0 (unrelated to the substr
regression which started in 5.36.0), and is still two orders of
magnitude slower than the pos()-based alternative.

## Proposed fix

[sv-canonical-position.patch](sv-canonical-position.patch) (against
blead 57f455a) sets `canonical_position` on the `S_sv_pos_u2b_midway`
paths in `S_sv_pos_u2b_cached()`: unconditionally where the offset is
bounded by a cached position, and `uoffset <= mg_len` where it is
bounded by the known end, so out-of-range positions are still never
cached (preserving the fix for perl/perl5#18588).

Verified: benchmark restored to pre-5.36 speed (1.60s -> 0.010s),
perl core test suite passes (the only failure, t/run/locale.t test 49,
fails identically without the patch on macOS), `${^UTF8CACHE} = -1`
assertion mode random stress passes, and cut results are byte-identical
across 5.34.1 / 5.42.2 / patched blead.
