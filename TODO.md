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
- [x] Documented (not fixed): `--backend=c` can abort with "loop bound out
      of vec range" on `_bn_mag_add` and anything that calls it -- a real
      C-backend optimizer-hint bug, tracked as BUG-3 in
      `vani-compiler/docs/TODO_CURRENT.md`. Default LLVM backend unaffected;
      full test suite verified clean under it (both `--no-verify` and full
      SMT `vanic check`).

---

## Future

- **v0.2.0: Rational numbers** -- `struct Rational { num: BigInt, den: BigInt
  }`, reduced via `bn_gcd`, arithmetic composed from `BigInt`'s operations.
  The natural next step once this integer layer has had real usage; not
  started.
- `bn_pow_i64` (exponentiation by squaring) / modular exponentiation --
  straightforward once needed, deferred as non-foundational.
- Fixed-size-array (`[i64; N]`) sort-style dual API was considered and
  rejected for this package -- `BigInt`'s size is inherently dynamic
  (arbitrary precision), so there's no analogous fixed-size form.
- Long division is `O(n^2 log BASE)` (binary search per limb), not Knuth's
  Algorithm D's `O(n^2)` single-digit-estimate approach -- a legitimate
  future optimization if profiling ever shows division dominating a real
  workload; not needed at this library's current scope.
