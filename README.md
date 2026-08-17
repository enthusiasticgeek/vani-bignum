# vani-bignum

Arbitrary-precision integer library for the [vāṇी compiler](https://github.com/enthusiasticgeek/vani-compiler).
Numeric foundation for the planned symbolic-math tier (`vani-symbolic`, `vani-polyalgebra`)
in [kosh-index/ROADMAP.md](https://github.com/enthusiasticgeek/kosh-index/blob/main/ROADMAP.md).

`BigInt` owns a `Vec<i64>` of base-1e9 digit limbs, so it's **not** Copy -- every
function takes `ref`/`mut ref BigInt` and returns `BigInt` by value, unlike the
Copy-struct sibling libraries ([vani-complex](https://github.com/enthusiasticgeek/vani-complex),
[vani-interval](https://github.com/enthusiasticgeek/vani-interval)). Closer in shape to the
`Vec`-owning-struct libraries ([vani-sparse](https://github.com/enthusiasticgeek/vani-sparse)).

**API reference / tutorial:** <https://enthusiasticgeek.github.io/vani-bignum/>

## Add to your project

```toml
# vani.toml
[deps]
bignum = { registry = "kosh", version = "^0.2" }
```

```sh
vanic add bignum
vanic build
```

## What's included (v0.2.0 — integers + rationals; see TODO.md)

| Module | Functions |
|---|---|
| Construction / IO | `bn_zero`, `bn_from_i64`, `bn_from_str`, `bn_to_str`, `bn_to_i64` |
| Sign / predicates | `bn_sign`, `bn_is_zero`, `bn_neg`, `bn_abs` |
| Comparison | `bn_cmp`, `bn_eq`, `bn_ne`, `bn_lt`, `bn_le`, `bn_gt`, `bn_ge`, `implement Eq for BigInt` (`==`/`!=` work directly) |
| Arithmetic | `bn_add`, `bn_sub`, `bn_mul`, `bn_div_mod`, `bn_div`, `bn_mod` |
| GCD / exponentiation | `bn_gcd`, `bn_pow_i64` (squaring), `bn_pow_mod` (modular, reduces after every multiply) |
| Rational numbers (v0.2.0) | `struct Rational { num: BigInt, den: BigInt }`, `rat_new` (reduces to lowest terms via `bn_gcd`), `rat_from_i64`, `rat_to_str`, `rat_is_zero`, `rat_neg`, `rat_abs`, `rat_add`, `rat_sub`, `rat_mul`, `rat_div`, `rat_cmp`, `rat_eq`/`rat_ne`/`rat_lt`/`rat_le`/`rat_gt`/`rat_ge` |

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

## What this library does NOT provide

- Compiler builtins that already exist and must NOT be reimplemented here:
  `push` `pop` `len` `set` `vec` `str_byte_at` `str_pad_left` `i64_to_str`

## License

MIT
