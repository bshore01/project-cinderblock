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
