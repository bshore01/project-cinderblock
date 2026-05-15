# Build Log — wilsontest.c on Ubuntu 24.04

## Environment

- OS: Ubuntu 24.04 (WSL2, kernel 6.6.114.1-microsoft-standard-WSL2)
- GCC: 15.2.0 (Ubuntu 15.2.0-16ubuntu1)
- GMP: 6.3.0+dfsg-5ubuntu2 (`libgmp-dev`)

---

## Step 1 — Install dependencies

```
sudo apt-get install -y build-essential libgmp-dev
```

Installs GCC, make, binutils, and the GMP development headers/library.

**Note on GMP header location:** On Ubuntu 24.04 the header lands at
`/usr/include/x86_64-linux-gnu/gmp.h` (multiarch path), not the legacy
`/usr/include/gmp.h`. GCC knows this path automatically.

---

## Step 2 — Source inspection and include fix assessment

`wilsontest.c` uses `#include "gmp.h"` (quoted form). With a quoted include,
GCC first looks in the source file's directory; if not found it falls back to
the system include paths, which include the multiarch directory above. Because
there is no local `gmp.h` in the project directory, the fallback succeeds and
**no source change was required**.

---

## Step 3 — Compile

```
gcc -O2 -o wilsontest wilsontest.c -lgmp -lm
```

Result: **success** — five `[-Wunused-result]` warnings about unchecked
`scanf`/`fscanf` return values; no errors.

The warnings are benign: the interactive prompts are not safety-critical and
the code handles bad input via `assert` elsewhere.

---

## Step 4 — Smoke test run

Searched all primes in [5, 600], processing up to 1000 at once, no full
residue dump:

```
printf '5 600\n1000\n0\n' | ./wilsontest
```

Output (condensed):
```
Not found unfinished workunit.
...
Testing p=5...593,  p==5  mod 12, time=0 sec.
Testing p=7...577,  p==1  mod 3,  time=0 sec.
Testing p=11...599, p==11 mod 12, time=0 sec.
Done the st=5-en=600 interval. Time=0 sec.
```

Completed in under one second.

---

## Step 5 — Verify Wilson prime detection

The program writes primes where `(p-1)! ≡ -1 + k·p (mod p²)` with `|k| ≤ 100`
to `wilson.txt`. Wilson primes proper satisfy `k = 0`, written as `p -1+0p`.

From `wilson.txt`:

```
5 -1+0p
13 -1+0p
563 -1+0p
```

All three known Wilson primes (5, 13, 563) are correctly identified. No
spurious Wilson primes appear in the range.

---

## Step 6 — Rejection test: [600, 100000]

Verified that the program produces **no Wilson prime hits** beyond 563 in a
range where none are known to exist. `wilson.txt` was cleared first so only
results from this run appear.

```
rm -f wilson.txt && printf '600 100000\n1000\n0\n' | ./wilsontest
```

Output (condensed):
```
Not found unfinished workunit.
Testing p=601...19051,  p==1  mod 3,  time=0 sec.
Testing p=617...39989,  p==5  mod 12, time=0 sec.
Testing p=647...39779,  p==11 mod 12, time=0 sec.
...
Done the st=600-en=100000 interval. Time=0 sec.
```

Completed in under one second. 141 near-Wilson primes (|k| ≤ 100) were
written to `wilson.txt`, but:

```
$ grep '+0p' wilson.txt
(no output)
```

**Result: zero Wilson primes in [600, 100000].** This is the correct answer —
the next Wilson prime after 563, if one exists, has not been found up to at
least 2×10¹³ (per the README). The program correctly finds no false positives.

---

---

## wilsontest2.c

### Step W2-1 — Compile

```
gcc -O2 -o wilsontest2 wilsontest2.c -lgmp -lm
```

Result: **success** — same five `[-Wunused-result]` warnings as wilsontest.c,
no errors. No source changes were required (`"gmp.h"` resolves via the same
GCC fallback path).

### Step W2-2 — Full detection and rejection test: [5, 100000]

Ran a single pass covering the entire range, verifying both that the three
known Wilson primes are found and that no others appear up to 100000.

```
rm -f wilson.txt && printf '5 100000\n1000\n0\n' | ./wilsontest2
```

Completed in under one second.

```
$ grep '+0p' wilson.txt
5 -1+0p
13 -1+0p
563 -1+0p
```

Exactly the three known Wilson primes. No false positives.

### Step W2-3 — Cross-check against wilsontest on [10000, 20000]

Both implementations were run in separate directories so their `wilson.txt`
files do not interfere, using identical inputs:

```
# in /tmp/wt1/
printf '10000 20000\n1000\n0\n' | /path/wilsontest

# in /tmp/wt2/
printf '10000 20000\n1000\n0\n' | /path/wilsontest2
```

Both completed in under one second. 15 near-Wilson primes were reported by
each. The outputs are **byte-for-byte identical** (same primes, same k values,
same line order):

```
16421 -1+90p
10567 -1-78p
11047 -1-8p
12799 -1+17p
12889 -1+95p
13297 -1-62p
14419 -1+7p
10739 -1-18p
12659 -1+100p
13043 -1-46p
13967 -1-42p
14519 -1+99p
15959 -1-68p
16319 -1-25p
16823 -1-39p
```

```
diff /tmp/wt1/wilson.txt /tmp/wt2/wilson.txt
(no output — identical)
```

No Wilson primes (k=0) appear in [10000, 20000], consistent with both
previous tests.

---

## Summary

| Step | Command | Result |
|------|---------|--------|
| Install packages | `sudo apt-get install -y build-essential libgmp-dev` | OK |
| Compile wilsontest | `gcc -O2 -o wilsontest wilsontest.c -lgmp -lm` | OK (warnings only) |
| Run 5–600 | `printf '5 600\n1000\n0\n' \| ./wilsontest` | OK, <1 sec |
| Verify detection | `grep '+0p' wilson.txt` | 5, 13, 563 confirmed |
| Run 600–100000 | `printf '600 100000\n1000\n0\n' \| ./wilsontest` | OK, <1 sec |
| Verify rejection | `grep '+0p' wilson.txt` | No output — zero false positives |
| Compile wilsontest2 | `gcc -O2 -o wilsontest2 wilsontest2.c -lgmp -lm` | OK (warnings only) |
| Run 5–100000 (v2) | `printf '5 100000\n1000\n0\n' \| ./wilsontest2` | OK, <1 sec |
| Verify detection (v2) | `grep '+0p' wilson.txt` | 5, 13, 563 confirmed; no others |
| Cross-check 10000–20000 | `diff wt1/wilson.txt wt2/wilson.txt` | Byte-for-byte identical |

No source modifications were necessary for either program.

---

## pw13.c build

### Step P-1 — Extract NTT library

```
python3 -c "import tarfile; tarfile.open('ntt-0.1.2.tar.bz2').extractall('.')"
```

Extracted 17 files into `ntt-0.1.2/`:

| Type | Files |
|------|-------|
| Sources | `fft_array.c`, `fft_base.c`, `fft_main.c`, `intmult.c`, `memory.c`, `misc.c`, `modarith.c`, `profile.c`, `tunetab.c` |
| Headers | `ntt.h`, `ntt-internal.h` |
| Other | `makefile`, `run.py`, `test.c`, `test.h`, `tune.c`, `COPYING` |

Note: `bzip2` was not installed; Python's built-in `tarfile` module handled the extraction.

The NTT library's headers use only `<gmp.h>` (public API) plus OpenMP and
standard C headers — no GMP internals.

---

### Step P-2 — First compile attempt (expected to fail)

Command (original embedded compile line, minus the hardcoded
`-I/home/gerbicz/gmp-5.0.2 -L/home/gerbicz/gmp-5.0.2` paths):

```
gcc -fopenmp -m64 -fgnu89-inline -std=c99 -O2 -lgmp -lm -Wall \
  -o pw13 pw13.c \
  ntt-0.1.2/profile.c ntt-0.1.2/misc.c ntt-0.1.2/modarith.c ntt-0.1.2/memory.c \
  ntt-0.1.2/fft_main.c ntt-0.1.2/fft_base.c ntt-0.1.2/fft_array.c \
  ntt-0.1.2/intmult.c ntt-0.1.2/tunetab.c
```

**Errors (two distinct problems):**

1. `pw13.c:79: fatal error: gmp-5.0.4/gmp.h: No such file or directory` —
   includes use path-prefixed `"gmp-5.0.4/gmp.h"` etc.

2. `ntt-internal.h:372: error: implicit declaration of function 'MUL128'` —
   a preprocessor consistency bug in the NTT library: `AVOID_128_BIT` is
   always **defined** (as `0` at line 25 via `#ifndef`), so `#ifdef
   AVOID_128_BIT` guards at lines 370 and 422 always fire, but the `MUL128`/
   `DIV128` macro definitions live in a `#if AVOID_128_BIT` (value-check) block
   that evaluates false. The `#ifdef` should be `#if` at those two sites.

---

### Step P-3 — Minimal source changes

#### Fix 1: `ntt-0.1.2/ntt-internal.h` — two `#ifdef` → `#if` corrections

Lines 370 and 422: changed `#ifdef AVOID_128_BIT` to `#if AVOID_128_BIT` so
the value (0 = use `__uint128_t` path) is checked consistently everywhere.
On GCC 15 with `__int128` available, `AVOID_128_BIT=0` is correct and the
assembly `MUL128`/`DIV128` macros are rightly skipped.

#### Fix 2: `pw13.c` — includes and compatibility shim

**Include changes (lines 79–84):**

```c
// Before:
#include "gmp-5.0.4/gmp.h"
#include "gmp-5.0.4/gmp-impl.h"
#include "omp.h"
#include "gmp-5.0.4/longlong.h"

// After:
#include <gmp.h>
// #include "gmp-5.0.4/gmp-impl.h"  -- replaced by compat shim below
#include <omp.h>
// #include "gmp-5.0.4/longlong.h"  -- replaced by compat shim below
```

**Compatibility shim inserted after the `#ifdef __unix__` block:**

```c
/* ---- GMP 6.x / Ubuntu 24.04 compatibility shim ---- */

// longlong.h -> GCC builtin
#define count_leading_zeros(count, x) \
    ((count) = (int)__builtin_clzll((unsigned long long)(x)))

// gmp-impl.h trivial constant
#ifndef MP_LIMB_T_MAX
#define MP_LIMB_T_MAX  (~(mp_limb_t)0)
#endif

// gmp-impl.h struct accessor macros -> direct __mpz_struct field access
// (_mp_size, _mp_d, _mp_alloc are part of GMP's public ABI;
//  _mpz_realloc is declared in the public gmp.h)
#define SIZ(x)       ((x)->_mp_size)
#define PTR(x)       ((x)->_mp_d)
#define MPZ_REALLOC(x, n) \
    (((mp_size_t)(n) <= (x)->_mp_alloc) \
     ? (x)->_mp_d \
     : (mp_ptr)_mpz_realloc((x), (mp_size_t)(n)))

// gmp-impl.h limb-array macros -> trivial equivalents
#define MPN_COPY(d, s, n)  mpn_copyi((d), (s), (n))
#define MPN_NORMALIZE(d, n) \
    do { while ((n) > 0 && (d)[(n)-1] == 0) (n)--; } while (0)
```

**Why direct struct access for SIZ/PTR/MPZ_REALLOC:** The `__mpz_struct`
fields `_mp_size`, `_mp_d`, and `_mp_alloc` are part of GMP's stable public
ABI (documented as such; applications may access them). This is simpler and
more direct than `mpz_import`/`mpz_export`, which would require a temporary
buffer and add unnecessary copies. `_mpz_realloc` is declared in the public
`gmp.h`.

---

### Step P-4 — Second compile attempt

**New error:** `MPN_NORMALIZE` and `MPN_COPY` — two more `gmp-impl.h` macros
found during compilation (at lines 3447 and 3581). Added to the shim above.

**New error:** linker failure — all GMP symbols undefined. Cause: `-lgmp -lm`
were placed *before* the source files in the command. `ld.bfd` requires
libraries to follow the objects that reference them.

**Fix:** move `-lgmp -lm` to the end of the command line.

**Final working compile command:**

```
gcc -fopenmp -m64 -fgnu89-inline -std=c99 -O2 -Wall \
  -o pw13 pw13.c \
  ntt-0.1.2/profile.c ntt-0.1.2/misc.c ntt-0.1.2/modarith.c ntt-0.1.2/memory.c \
  ntt-0.1.2/fft_main.c ntt-0.1.2/fft_base.c ntt-0.1.2/fft_array.c \
  ntt-0.1.2/intmult.c ntt-0.1.2/tunetab.c \
  -lgmp -lm
```

Result: **success** — 52 warnings (pre-existing style issues: misleading
indentation, unused variables, unchecked `scanf`/`fread` returns), zero errors.
Binary: `pw13` (334 KB).

---

### Step P-5 — Smoke test

pw13.c has a different interactive prompt sequence than wilsontest.c:

| # | Prompt | Value used |
|---|--------|------------|
| 1 | Number of cores | 4 |
| 2 | Exponent `e` (must be in allowed list) | 2 (Wilson mode) |
| 3 | Start and end of range | `5 100` |
| 4 | Primes per batch (interval) | 1000 |
| 5 | Print all residues to file (0/1) | 0 |
| 6 | Use large savefile (0/1) | 0 |
| 7 | Wall time limit (seconds) | 86400 |

```
printf '4\n2\n5 100\n1000\n0\n0\n86400\n' | ./pw13
```

Note: pw13.c enforces a minimum start of `3*e+1 = 7` for `e=2`, so p=5 is
outside its search range. p=13 was correctly identified: `13 -1+0p`.

Completed in 3 seconds (includes one-time `setuppow1024()` precomputation).

---

### Step P-6 — Cross-validation against wilsontest on [10000, 20000]

```
printf '4\n2\n10000 20000\n1000\n0\n0\n86400\n' | ./pw13
```

Sorted output compared against the wilsontest reference from Session 1:

```
diff <(sort wilson.txt) <(sort /tmp/wt1/wilson.txt)
(no output — identical)
```

All 15 near-Wilson primes in [10000, 20000] match exactly between pw13 and
wilsontest — same primes, same k-values. No Wilson primes (k=0) in this range,
consistent with previous tests.

---

### Summary of changes made

| File | Change |
|------|--------|
| `ntt-0.1.2/ntt-internal.h` | Lines 370, 422: `#ifdef AVOID_128_BIT` → `#if AVOID_128_BIT` (bug fix: consistency with value-based check at line 58) |
| `pw13.c` | Lines 79–84: replace path-prefixed GMP includes with `<gmp.h>` / `<omp.h>`, comment out `gmp-impl.h` and `longlong.h` |
| `pw13.c` | Insert 25-line compatibility shim defining `count_leading_zeros`, `MP_LIMB_T_MAX`, `SIZ`, `PTR`, `MPZ_REALLOC`, `MPN_COPY`, `MPN_NORMALIZE` |
