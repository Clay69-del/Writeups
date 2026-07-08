# Elliptic Curve Cryptography - Write-up

## Challenge

**Name:** Elliptic Curve Cryptography  
**Category:** Crypto  
**Points:** 500  
**Author:** whitequeen_frenchazo

The challenge gave a small elliptic curve and said:

> the flag is hidden somewhere in this curve (100-bit?) look at the p, a and x params

The original curve was:

```json
[
  {
    "field": {
      "p": "0x0fffffffffffffffffffffff67"
    },
    "a": "0x0fffffffffffffffffffffff64",
    "b": "0x00000000000000000000000abb",
    "order": "0x0ffffffffffff918654d8534a1",
    "subgroups": [
      {
        "x": "0x00000000000000000000000001",
        "y": "0x05a0248e58b8beaa670036b766",
        "order": "0xffffffffffff918654d8534a1",
        "cofactor": "0x1"
      }
    ]
  }
]
```

At first glance this looks like a normal ECC discrete-log style challenge. It is not. The important part is that the curve was deliberately generated to resemble a known curve-generation family.

## Short answer

The final hint points to the IETF Internet-Draft named:

```text
draft-black-numscurves-02
```

With the LYKNCTF wrapper, the expected flag is:

```text
LYKNCTF{draft-black-numscurves-02}
```

## Step 1 - Normalize the curve

The given prime is:

```text
p = 0x0fffffffffffffffffffffff67
```

Interpreting it as an integer gives:

```text
p = 1267650600228229401496703205223
```

This is exactly:

```text
p = 2^100 - 153
```

So the challenge title's `(100-bit?)` note was not random. The field prime really is a 100-bit pseudo-Mersenne prime.

The curve parameter is:

```text
a = 0x0fffffffffffffffffffffff64
```

and:

```text
a = p - 3
```

That means the curve is in short Weierstrass form:

```text
y^2 = x^3 - 3x + b mod p
```

The generator has:

```text
X(P) = 1
```

These three facts are the core of the challenge:

```text
p = 2^100 - 153
a = p - 3
X(P) = 1
```

They match the shape of the deterministic Nothing Up My Sleeve, or NUMS, Weierstrass curve-generation method.

## Step 2 - Verify this was not random

A quick sanity script confirms that the parameters are internally consistent.

```python
from sympy import isprime

p = int("0x0fffffffffffffffffffffff67", 16)
a = int("0x0fffffffffffffffffffffff64", 16)
b = int("0xabb", 16)
r = int("0x0ffffffffffff918654d8534a1", 16)
x = 1
y = int("0x05a0248e58b8beaa670036b766", 16)

print("bits(p)       =", p.bit_length())
print("2^100 - p     =", 2**100 - p)
print("p is prime    =", isprime(p))
print("a == p - 3    =", a == p - 3)

lhs = (y * y) % p
rhs = (x**3 + a*x + b) % p
print("point valid   =", lhs == rhs)

trace = p + 1 - r
print("order prime   =", isprime(r))
print("trace         =", trace)

rtwist = p + 1 + trace
print("twist prime   =", isprime(rtwist))
```

Expected output:

```text
bits(p)       = 100
2^100 - p     = 153
p is prime    = True
a == p - 3    = True
point valid   = True
order prime   = True
trace         = 1943501465635527
twist prime   = True
```

This does not directly print the flag, but it shows that the curve was crafted carefully. It follows the same design ideas as NUMS curves: a pseudo-Mersenne prime, a fixed curve form, a small generator x-coordinate, prime group order, and prime twist order.

## Step 3 - Use the first hint

The later hint provided this larger curve fragment:

```text
p    = 0xFFFFFFFF...FFFFFDC7
a    = 0xFFFFFFFF...FFFFFDC6
d    = 0x9BAA8
X(P) = 0x20
h    = 0x04
```

This is not the same 100-bit curve. It is a 512-bit twisted Edwards curve. The hint is useful because it identifies the source family.

Searching those exact values leads to the IETF Datatracker page for:

```text
Elliptic Curve Cryptography (ECC) Nothing Up My Sleeve (NUMS) Curves and Curve Generation
```

The document name is:

```text
draft-black-numscurves-02
```

In that document, the 512-bit twisted Edwards curve with:

```text
p    = 2^512 - 569
a    = p - 1
d    = 0x9BAA8
X(P) = 0x20
h    = 0x04
```

is explicitly listed as:

```text
Curve-Id: numsp512t1
```

So the purpose of the hint was not to make us solve an ECC problem. It was to make us identify the IETF draft behind the curve-generation style.

## Step 4 - Decode the final hint

The final hint was:

```text
Oops I slipped my keyboard and type IETF and Draft with #000000
```

This gives the remaining pieces directly:

```text
IETF        -> source is an IETF Internet-Draft
Draft       -> look for a draft name, not only a curve name
#000000     -> black
```

Combining these with the NUMS curve clue gives:

```text
draft + black + nums curves
```

The exact IETF draft name is:

```text
draft-black-numscurves-02
```

## Why the original 100-bit curve exists

The official draft defines 256-bit, 384-bit, and 512-bit curves. The challenge uses a tiny 100-bit curve, probably so that the parameters are short enough to inspect manually.

The miniature curve still preserves the recognizable features:

| Feature | Original challenge | NUMS draft pattern |
|---|---:|---|
| Prime form | `2^100 - 153` | `2^s - c` |
| Prime congruence | `p = 3 mod 4` | same requirement |
| Weierstrass `a` | `p - 3` | `-3 mod p` |
| Generator x | `1` | first valid small x |
| Cofactor | `1` | prime-order Weierstrass curve |

That is why the challenge prompt specifically said to look at `p`, `a`, and `x`. Those values are enough to recognize the curve-generation pattern after seeing the IETF/Black/NUMS hint.

## Final flag

```text
LYKNCTF{draft-black-numscurves-02}
```

## References

1. IETF Datatracker, `draft-black-numscurves-02`: *Elliptic Curve Cryptography (ECC) Nothing Up My Sleeve (NUMS) Curves and Curve Generation*.  
   https://datatracker.ietf.org/doc/html/draft-black-numscurves-02

2. The same draft lists the 512-bit twisted Edwards curve with `d = 0x9BAA8`, `X(P) = 0x20`, `h = 0x04` as `Curve-Id: numsp512t1`.
