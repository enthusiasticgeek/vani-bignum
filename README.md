# vani-bignum

Arbitrary-precision integer library for the [vāṇी compiler](https://github.com/enthusiasticgeek/vani-compiler).
Numeric foundation for the planned symbolic-math tier (`vani-symbolic`, `vani-polyalgebra`)
in [kosh-index/ROADMAP.md](https://github.com/enthusiasticgeek/kosh-index/blob/main/ROADMAP.md).

`BigInt` owns a `Vec<i64>` of base-1e9 digit limbs, so it's **not** Copy -- every
function takes `ref`/`mut ref BigInt` and returns `BigInt` by value, unlike the
Copy-struct sibling libraries ([vani-complex](https://github.com/enthusiasticgeek/vani-complex),
[vani-interval](https://github.com/enthusiasticgeek/vani-interval)). Closer in shape to the
`Vec`-owning-struct libraries ([vani-sparse](https://github.com/enthusiasticgeek/vani-sparse)).

## Add to your project

```toml
# vani.toml
[deps]
bignum = { registry = "kosh", version = "^0.1" }
```

```sh
vanic add bignum
vanic build
```

## What's included (v0.1.0 — integers only; see TODO.md)

| Module | Functions |
|---|---|
| Construction / IO | `bn_zero`, `bn_from_i64`, `bn_from_str`, `bn_to_str`, `bn_to_i64` |
| Sign / predicates | `bn_sign`, `bn_is_zero`, `bn_neg`, `bn_abs` |
| Comparison | `bn_cmp`, `bn_eq`, `bn_ne`, `bn_lt`, `bn_le`, `bn_gt`, `bn_ge`, `implement Eq for BigInt` (`==`/`!=` work directly) |
| Arithmetic | `bn_add`, `bn_sub`, `bn_mul`, `bn_div_mod`, `bn_div`, `bn_mod` |
| GCD | `bn_gcd` |

## Encoding

```
struct BigInt { limbs: Vec<i64>, sign: i64 }
```

Little-endian base-1,000,000,000 (1e9) limbs (`limbs[0]` is least significant).
Base 1e9 is chosen because vāṇी has no integer type wider than `i64` (confirmed --
no `i128`) and `+`/`-`/`*` trap on overflow rather than wrap, so every intermediate
value in a limb operation must provably fit in `i64`; a single limb product is
under 1e18, safely inside range. It also makes decimal string conversion trivial
(9 digits per limb, no base-conversion math).

`sign` is `-1`, `0`, or `1` (matching the compiler's own `i64_signum` convention).
Zero is the canonical form `sign=0, limbs=vec()` (empty, not `[0]`); every nonzero
value has a nonzero most-significant limb (no leading zero limbs).

**Division convention**: `bn_div_mod` truncates toward zero (quotient sign =
`sign(a) * sign(b)`, remainder sign = `sign(a)`), deliberately matching vāṇी's
native `i64` `/`/`%` semantics rather than floor division.

## What this library does NOT provide (yet)

- **Rational numbers.** Planned as v0.2.0 (`{ num: BigInt, den: BigInt }`, reduced
  via `bn_gcd`) once this integer layer has had real usage. Not started.
- **`bn_pow_i64` / modular exponentiation.** Straightforward to add once needed
  (exponentiation by squaring via `bn_mul`) -- deferred, not foundational.
- Compiler builtins that already exist and must NOT be reimplemented here:
  `push` `pop` `len` `set` `vec` `str_byte_at` `str_pad_left` `i64_to_str`

## Known issue: `--backend=c`

This package's tests pass cleanly under the default LLVM backend (`vanic run`,
`vanic build`, `vanic test`). Under `--backend=c`, `_bn_mag_add` (and functions
that call it, i.e. most of the library) can abort with `loop bound out of vec
range` -- a real compiler bug in the C backend's `while`-loop bounds-checking
optimizer hint, not a bug in this package. Tracked upstream as BUG-3 in
[vani-compiler/docs/TODO_CURRENT.md](https://github.com/enthusiasticgeek/vani-compiler/blob/main/docs/TODO_CURRENT.md).
Use the default LLVM backend.

## License

MIT
