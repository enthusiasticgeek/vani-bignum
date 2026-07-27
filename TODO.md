# vani-bignum — TODO

> Compiler builtins that already exist and must NOT be reimplemented:
> `push` `pop` `len` `set` `vec` `str_byte_at` `str_pad_left` `i64_to_str`
>
> No deps (self-contained).

---

## v0.1.0 — Implemented ✓

### Construction / IO (5 functions)
- [x] `bn_zero`, `bn_from_i64` -- build from nothing / a plain `i64`
- [x] `bn_from_str` -- decimal string parse (optional leading `-`), hand-parsed
      byte-by-byte via `str_byte_at` (no str-to-int builtin exists in vāṇी)
- [x] `bn_to_str` -- decimal string format, most-significant limb first
- [x] `bn_to_i64` -- reconstructs via real `i64` arithmetic, so it naturally
      traps if the value doesn't fit (intentional, matches "everything traps
      on overflow" rather than silently truncating)

### Sign / predicates (4 functions)
- [x] `bn_sign`, `bn_is_zero`, `bn_neg`, `bn_abs`

### Comparison (7 functions + 1 interface impl)
- [x] `bn_cmp` (-1/0/1), `bn_eq`, `bn_ne`, `bn_lt`, `bn_le`, `bn_gt`, `bn_ge`
- [x] `implement Eq for BigInt` -- `==`/`!=` work as operators (confirmed the
      `self: ref BigInt, other: ref BigInt` form works, not just the by-value
      form the language's own `struct_eq.vani` example shows -- no prior
      real-code precedent for a non-Copy struct's `Eq` impl existed anywhere
      in the ecosystem before this package)

### Arithmetic (6 functions)
- [x] `bn_add`, `bn_sub` -- magnitude add/sub composed with sign-rule logic
- [x] `bn_mul` -- schoolbook, base 1e9, immediate per-product carry
      propagation (never accumulates multiple un-reduced products at one
      output position -- the overflow-safety-critical design choice, see
      `src/lib.vani`'s module header and `bn_mul`'s own comment)
- [x] `bn_div_mod`, `bn_div`, `bn_mod` -- most-significant-limb-first long
      division, binary-searching each quotient digit in `[0, 1e9)` against
      the running remainder. `O(n^2 log BASE)`, not Knuth's Algorithm D
      (deliberately simpler to get provably correct -- see module header)

### GCD (1 function)
- [x] `bn_gcd` -- Euclidean algorithm via repeated `bn_div_mod`; also the
      composed multi-function validation test for this package (exactly the
      function a future `Rational` reduction step will need)

### Tests
- [x] `tests/test_construction.vani` -- round-trips through `i64` and decimal
      strings, multi-limb values (> 1e9)
- [x] `tests/test_arithmetic.vani` -- add/sub/mul cross-checked against plain
      `i64` arithmetic for small cases, PLUS carry-crossing cases at the
      limb boundary (999999999 + 1, 1000000000 - 1) and a genuinely
      multi-limb 15-digit x 15-digit multiply cross-checked against Python's
      exact result (not hand-computed -- hand arithmetic at that size was
      tried first and was wrong, confirming the value of this discipline)
- [x] `tests/test_comparison.vani` -- `bn_cmp` and every wrapper, negative-
      magnitude ordering inversion, `==`/`!=` operator sugar
- [x] `tests/test_div_mod.vani` -- small truncating-division cases (all four
      sign combinations, hand-verified), plus a large 30-digit / 9-digit
      case with values from Python (not hand-computed) verified two ways:
      against the expected quotient/remainder AND via the round-trip
      identity `bn_mul(q, b) + r == a`
- [x] `tests/test_gcd.vani` -- small hand-verified cases (including
      `gcd(0,0)=0`, `gcd(x,0)=|x|`, negative operands), plus a large case
      from Python's `math.gcd`

### Safety annotations
- [x] `#[bounded_stack(bytes=N)]` on every function, set to `vanic check`'s
      exact reported worst-case (largest: `bn_gcd`/`bn_div`/`bn_mod` at 1576
      bytes, since they compose the full `bn_div_mod -> _bn_divmod_nonneg ->
      bn_sub -> bn_add -> _bn_mag_sub -> _bn_strip` chain)
- [x] `#[wcet(cycles=N)]` on every fixed-shape function (`bn_zero`,
      `bn_sign`, `bn_is_zero`, `_bn_with_sign`); every other function has a
      `Vec`-length- or value-dependent loop bound and gets a WCET *formula*
      comment instead, matching `vani-calculus`'s `bisect` convention
- [x] Full `vanic check` (real SMT verification, not just `--no-verify`)
      passes clean on every test file

### Known issues
- [x] ~~`--backend=c` could abort with "loop bound out of vec range" on
      `_bn_mag_add`~~ -- was a real C-backend optimizer-hint bug (BUG-3),
      **fixed upstream 2026-07-24**. Full test suite + example verified
      clean on both backends after the fix.

---

## v0.1.1 (2026-07-25)

- [x] Added `#[bounded_stack(bytes = 257)]` to `implement Eq for BigInt`'s
      `eq` method, dropping the `--allow-partial-safety-coverage` escape
      hatch v0.1.0 needed -- vani-compiler's BUG-4 (parser rejected
      `#[attr]`-prefixed methods inside `implement` blocks) is now fixed.
      `vanic audit-safety` reports full coverage with no gaps.
- [x] Full test suite + example re-verified on both backends after the
      BUG-3 fix (previously LLVM-only).

---

## v0.1.2 (2026-07-26)

- [x] `bn_pow_i64(a, exp)` -- exponentiation by squaring, `exp >= 0`
      enforced via `assert` (no `Rational` type yet to represent a
      negative-exponent result). `exp = 0` returns 1 for any `a`
      (including 0), matching the standard `0^0 = 1` convention.
      `#[bounded_stack(bytes = 696)]`, `vanic check`'s exact reported
      worst-case (chain `bn_pow_i64 -> bn_mul -> _bn_strip`); no
      `#[wcet]` -- same "loop count is data-dependent" situation as
      `bn_gcd`.
- [x] `bn_pow_mod(a, exp, m)` -- modular exponentiation, same squaring
      loop but reduces mod `m` after every multiply so intermediate
      magnitudes stay bounded by `m` instead of growing with `exp`.
      `#[bounded_stack(bytes = 1904)]` (chain through `bn_mod ->
      bn_div_mod -> _bn_divmod_nonneg -> ... -> _bn_strip`).
- [x] `tests/test_pow.vani` -- 0-exponent identity (incl. `0^0`), small
      even/odd exponents, a multi-limb result (`2^100`, Python-computed),
      negative-base sign propagation, and `bn_pow_mod` cross-checked
      against Python's `pow(a, e, m)` for two cases plus the `m = 1`
      edge case. Full suite + `vanic audit-safety` re-verified on both
      backends after the change.

## v0.2.0 (2026-07-27)

- [x] `struct Rational { num: BigInt, den: BigInt }` -- always kept in
      lowest terms with `den > 0` (sign lives entirely in `num`), the
      canonical normal form every function assumes on input and restores
      on output, same "one canonical form" discipline as `BigInt`'s own
      zero=`sign0`/`limbs=empty` invariant.
- [x] `rat_new(num, den)` -- smart constructor: reduces via `bn_gcd` and
      normalizes `den`'s sign (negating both if `den` arrived negative).
      `den` must be nonzero (`assert`, same precondition class as
      `bn_div_mod`'s divisor). `rat_from_i64` skips the reduction (`n/1`
      is trivially already in lowest terms).
- [x] `rat_add`/`rat_sub`/`rat_mul`/`rat_div`/`rat_neg`/`rat_abs` --
      composed entirely from existing `BigInt` operations (the standard
      cross-multiply formulas), each result re-reduced via `rat_new`
      except `rat_neg`/`rat_abs` (negating/abs-ing the numerator can't
      change the gcd with `den`, so these skip the redundant reduction).
- [x] `rat_cmp` + wrappers (`rat_eq`/`rat_ne`/`rat_lt`/`rat_le`/`rat_gt`/
      `rat_ge`) -- cross-multiply comparison, correct without a sign
      case-split (unlike `BigInt`'s own `bn_cmp`) because the normal form
      guarantees both denominators are already positive. No `implement Eq
      for Rational` operator sugar in this pass -- `BigInt`'s own
      `interface Eq` is hard-coded to `self: ref BigInt`, so `Rational`
      would need its own separately-named interface, and it's unclear
      whether `==`/`!=` sugar recognizes a non-`Eq`-named interface;
      skipped rather than risk getting it subtly wrong for a nice-to-have
      (`rat_eq`/`rat_ne` cover the same need as plain functions, matching
      `bn_eq`/`bn_ne`'s own role alongside `BigInt`'s `Eq` impl).
- [x] `rat_to_str` -- always-explicit `"num/den"` format (even `den=1`
      shows as `"5/1"`, not simplified to `"5"`), matching this package's
      preference for an unambiguous representation over a prettier
      special-cased one.
- [x] `tests/test_rational.vani` -- reduction from both a negative
      numerator and a negative denominator (both must land on `-3/4`),
      arithmetic cross-checked against Python's `fractions.Fraction`, and
      a large-number case where the reduction happens to land on a whole
      number (`den=1`) -- caught during development that my first
      hand-guessed expected value for this case was wrong; recomputed
      with Python rather than trusting the hand guess. Full suite +
      `vanic audit-safety` re-verified on both backends.

## Future

- Fixed-size-array (`[i64; N]`) sort-style dual API was considered and
  rejected for this package -- `BigInt`'s size is inherently dynamic
  (arbitrary precision), so there's no analogous fixed-size form.
- Long division is `O(n^2 log BASE)` (binary search per limb), not Knuth's
  Algorithm D's `O(n^2)` single-digit-estimate approach -- a legitimate
  future optimization if profiling ever shows division dominating a real
  workload; not needed at this library's current scope.
