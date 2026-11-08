# Salted Hash Cracking — Lab 08

A hands-on walkthrough of recovering **salted SHA-1 password hashes** from a leaked
database table. The project is built as a progressive Jupyter notebook: each stage
implements one cracking technique **from scratch** to show *how* and *why* it works,
and the strategy escalates only when the previous approach hits a computational wall.

---

## Scenario

A database table was leaked containing only password **hashes**, not plaintext
passwords. Standard rainbow tables failed, which pointed to the passwords being
**salted**. Analysis confirmed a single shared salt across all accounts. The goal was
to recover as many of the original passwords as possible.

### What we were given

- **6 accounts**, each with a salted SHA-1 hash (40 hex characters → 160-bit → SHA-1).
- All hashes use the **same salt**, hidden somewhere in the lab material.
- Hash construction (from a leaked JS hint `CommonHash($salt,$pass)`):
  **`SHA1(salt + password)`** — salt prepended.
- **Password policy:** passwords set before May 2010 are **digits only, 5–7 chars**;
  passwords set after are **alphanumeric, 5–7 chars**.
- Salt recovered by test-and-confirm: **`www.exploringsecurity.com`**.

---

## Methodology — the attack funnel

The approach balances two levers:

- **Strategy** — use knowledge to *shrink* the search (nearly free).
- **Compute** — use raw power to *search* what can't be shrunk (expensive).

Strategy is always tried first; compute is escalated only when strategy is exhausted,
and only as far as the remaining keyspace demands. The result is a strictly
cost-ordered sequence, pivoting at each wall:

| # | Technique | Lever | What it exploits |
|---|-----------|-------|------------------|
| 1 | **Mask attack** (digit brute force) | strategy | the date-based password policy |
| 2 | **Dictionary attack** (rockyou) | strategy | humans pick real/common passwords |
| 3 | **Dictionary + rules** | strategy | humans mangle words (case, leet, suffixes) |
| 4 | **CPU brute force** | compute | small keyspaces only (≤5 char) |
| 5 | **GPU brute force** (hashcat) | compute | large keyspaces via parallelism |

Each pivot is driven by a **keyspace calculation** that proves the previous method
can't finish, answering the question "get *smarter* or get *more powerful*?"

---

## Results

All six passwords recovered.

| Account | Role | Password | Len | Cracked by |
|---------|------|----------|-----|------------|
| Tomtom | user | `54321` | 5 | mask (digits) |
| security | user | `121298` | 6 | mask (digits) |
| superman | user | `qwerty` | 6 | dictionary (rockyou) |
| Mark123 | user | `Mark123` | 7 | dictionary (verbatim in rockyou) |
| Sparky | **Admin** | `Admin09` | 7 | GPU brute force (hashcat) |
| JillC | user | `Q2w13sA` | 7 | GPU brute force (hashcat) |

**Observation:** the weak passwords (short digits, `qwerty`, username-as-password)
fell in seconds to cheap attacks. The two that followed better practice — 7-char,
mixed-case, no dictionary structure — survived everything until the full GPU brute
force, demonstrating that length + character diversity is the defense that holds.

---

## Repository contents

```
saltlab_tutorial.ipynb   # the progressive, runnable cracking notebook (main file)
hashes.txt               # the 6 hashes in hashcat mode-120 format (hash:salt)
README.md                # this file
```

---

## Requirements & setup

- **Python 3.8+** and **Jupyter** (`pip install jupyterlab`)
- The notebook itself uses only the standard library (`hashlib`, `itertools`,
  `multiprocessing`) — no third-party packages required for the core logic.
- **A wordlist** for the dictionary stages — this project uses `rockyou.txt`
  from [SecLists](https://github.com/danielmiessler/SecLists):

  ```bash
  # macOS / Linux
  curl -L -o rockyou.txt.tar.gz \
    https://github.com/danielmiessler/SecLists/raw/master/Passwords/Leaked-Databases/rockyou.txt.tar.gz
  tar -xzf rockyou.txt.tar.gz
  ```

- **(Optional) hashcat** for the GPU brute-force stage: `brew install hashcat` (macOS).

---

## Running the notebook

```bash
jupyter notebook saltlab_tutorial.ipynb
```

Run the cells top to bottom. The notebook is organized as stages:

1. **Hashing basics** — fingerprint the algorithm from hash length (SHA-1).
2. **Salt discovery** — recover the salt by testing candidates against the
   digit-only hashes (no password known in advance).
3. **Mask attack** — crack the pre-2010 digit passwords.
4. **Dictionary attack** — crack common passwords from rockyou.
5. **Dictionary + rules** — reach mangled variants of base words.
6. **CPU brute force** — exhaust small keyspaces; measure the feasibility wall.

> When loading rockyou, open it with `encoding="latin-1"` (it contains non-UTF-8
> bytes), and filter to the 5–7 alphanumeric policy to skip impossible candidates.

---

## Finishing with hashcat (GPU)

CPU brute force handles ≤5-char spaces but stalls on 6–7 char mixed-case
(`62⁷ ≈ 3.5 trillion`). hashcat runs the same brute force on the GPU at billions of
hashes/sec, making it feasible. Hash mode **120** is `sha1($salt.$pass)` — our exact
construction.

```bash
# Dictionary (sanity check + easy wins)
hashcat -m 120 -a 0 hashes.txt rockyou.txt

# Full alphanumeric brute force (quote masks in zsh!)
hashcat -m 120 -a 3 -O hashes.txt -1 '?l?u?d' '?1?1?1?1?1?1'      # 6-char
hashcat -m 120 -a 3 -O hashes.txt -1 '?l?u?d' '?1?1?1?1?1?1?1'    # 7-char

# Show all recovered passwords
hashcat -m 120 hashes.txt --show
```

On an Apple M3 Pro the 7-char sweep ran at ~2.17 GH/s and finished in ~26 minutes —
roughly **400× faster** than the CPU notebook, which is the entire reason GPUs
dominate password cracking.

---

## What this project demonstrates

- How to fingerprint a hash algorithm and recover an unknown salt.
- Why salting defeats rainbow tables but **not** a targeted search when the salt is
  guessable and shared.
- The cost-ordered attack funnel and *when* to pivot between strategy and compute.
- The CPU-vs-GPU gap and why password length is the defense that actually scales.

---
