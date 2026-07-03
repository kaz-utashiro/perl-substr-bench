# perl-substr-bench

Benchmark and bisection to identify the Perl release and commit which
introduced the substr/UTF-8 position cache regression.

Background (Japanese): https://qiita.com/kaz-utashiro/items/d7047abb8531d5afc40a

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
