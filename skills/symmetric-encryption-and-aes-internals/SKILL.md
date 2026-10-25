---
name: symmetric-encryption-and-aes-internals
description: Explain or reason about how symmetric-key ciphers like AES actually work internally (transposition, substitution, S-boxes, key expansion, round structure, block chaining/IV, diffusion, the avalanche effect) rather than treating encryption as an opaque library call. Use this when the user asks "how does AES actually work", "why do we need an IV/nonce", "what does ECB vs CBC mean and why is ECB insecure", "why does changing one bit of plaintext scramble the whole ciphertext", "what's the difference between a block cipher and a stream cipher", is debugging a "identical plaintext blocks produce identical ciphertext" bug (an ECB-mode symptom), is choosing between AES modes (ECB/CBC/CTR/GCM), or wants a simplified/from-scratch implementation of a substitution-permutation cipher for teaching or a toy project. Also trigger for questions about Kerckhoffs's principle, brute-force key-space math, frequency analysis, or known-plaintext/crib attacks.
---

# Symmetric Encryption and AES Internals

Symmetric encryption means the same key locks and unlocks the data. Every modern block
cipher, AES included, is built from exactly two primitive operations repeated many times:
**transposition** (reorder the data) and **substitution** (replace the data). Neither
primitive is strong alone — attackers defeat transposition with anagramming and substitution
with frequency analysis — but combining many rounds of both, mixed with the key via XOR,
produces a cipher that's fast, symmetric, and (for now) computationally infeasible to break
without the key. Understanding this construction is what lets you reason about *why* a mode
of operation is broken, not just recite "don't use ECB."

## Kerckhoffs's principle: the algorithm is never the secret

A cipher's security must depend only on the secrecy of the **key**, never on secrecy of the
method. AES's full specification is public; every implementation detail is public; the only
unknown to an attacker is the key. Any design that relies on attackers not knowing the
algorithm ("security through obscurity") collapses the moment the method leaks — and it
always eventually leaks. When reviewing a homegrown crypto scheme, the first question is:
"if the attacker had the full source code, would this still be secure?" If not, it's not real
security, it's an obfuscation delay.

## The two primitives

**Transposition** — reorder existing bits/bytes without changing their values. Defeated by
brute-force permutation search and by known-plaintext ("crib") attacks: if an attacker has
one plaintext/ciphertext pair encrypted with the same key material, recovering the reordering
pattern is often easy from that pair alone.

**Substitution** — replace each byte with another byte from a lookup table (an S-box).
Simple substitution (one fixed table) is broken by **frequency analysis**: in English text,
`E` and `TH` occur far more often than other letters/pairs, so the most frequent ciphertext
symbol is a strong guess for the most frequent plaintext symbol. This is why single-table
substitution ciphers get weaker the longer the message is — more text means more statistical
signal for the attacker. **Polyalphabetic substitution** (varying the table per position,
e.g. via a repeating key) defeats simple frequency analysis, but is itself weak if the key is
short relative to the message, because the substitution pattern still repeats.

AES's S-box is a single fixed, publicly known substitution table, but it's designed so that
similar input bytes produce very different output bytes — this local scrambling is what gets
amplified into whole-block diffusion over many rounds (see below).

## AES round structure

AES encrypts data in fixed 128-bit (16-byte) blocks, arranged as a 4x4 grid of bytes. A
key-expansion routine derives 11 round keys (for 128-bit keys) from the original key using
the same substitution/transposition/XOR toolkit, so the huge amount of "randomness" needed
across all rounds can be regenerated from one short key instead of stored. Each of AES's 10
rounds performs four steps:

1. **SubBytes** — substitute every byte via the S-box.
2. **ShiftRows** — transpose bytes within each row (shift by different amounts per row).
3. **MixColumns** — combine all four bytes of each column together via XOR and a fixed binary
   multiplication, so every output byte in a column depends on every input byte of that
   column.
4. **AddRoundKey** — XOR the entire grid with that round's derived key.

XOR is the workhorse because it's cheap in hardware/software *and* trivially reversible:
`(a XOR b) XOR b == a`. That reversibility is exactly why a symmetric cipher can use the same
key for encryption and decryption — every step in the round (substitution via table lookup,
transposition, and XOR) has a well-defined inverse, so running the rounds backward with the
same round keys recovers the plaintext.

```python
# Minimal illustration of one AES-like round's shape (not real AES — for building intuition
# about how the four steps compose, not as a usable cipher).
def sub_bytes(block, sbox):
    return bytes(sbox[b] for b in block)

def shift_rows(block):  # 4x4 grid stored row-major; row i shifted left by i
    rows = [block[i*4:(i+1)*4] for i in range(4)]
    return b"".join(row[i:] + row[:i] for i, row in enumerate(rows))

def add_round_key(block, round_key):
    return bytes(b ^ k for b, k in zip(block, round_key))

def encrypt_round(block, round_key, sbox):
    block = sub_bytes(block, sbox)
    block = shift_rows(block)
    # (MixColumns omitted here — real diffusion needs it; see "Why diffusion needs all four steps")
    block = add_round_key(block, round_key)
    return block
```

## Diffusion and the avalanche effect — why these steps, in this combination

- **Diffusion**: after enough rounds, every output bit depends on every input bit (of both
  plaintext and key). ShiftRows spreads bytes across columns; MixColumns then mixes each
  column together — the two steps *together* are what let a change anywhere reach every byte
  within a few rounds. Skip either step and diffusion stalls: SubBytes+AddRoundKey alone is
  just substitution repeated in place, and ShiftRows alone just permutes without mixing values.
- **Avalanche effect**: a one-bit change in plaintext (or key) should flip roughly half the
  output bits, unpredictably. This is diffusion's observable signature and the property that
  makes ciphertext look like noise even for highly patterned, predictable plaintext (e.g. a
  form email template with only a name and account number varying).

If you're asked "why does AES have so many steps for something that's just XOR and lookup
tables," the answer is: each step is individually cheap (fast in hardware and software), and
diffusion/avalanche only emerge from *repeating and combining* cheap steps — no single step
provides real security by itself.

## Block chaining: why a block cipher alone isn't enough

AES only defines how to scramble one 16-byte block. A **mode of operation** defines how to
chain many blocks into a full message, and the mode you pick has real security consequences:

- **ECB (Electronic Codebook)** — encrypt every block independently with the same key.
  **Insecure for anything but single-block data**: identical plaintext blocks always produce
  identical ciphertext blocks, so patterns in the plaintext (repeated headers, a large solid
  color region in an image, a form-letter template) remain visible in the ciphertext. This is
  the textbook "penguin image encrypted with ECB still shows the penguin" bug.
- **CBC (Cipher Block Chaining)** — XOR each plaintext block with the *previous ciphertext
  block* before encrypting; the first block is XORed with a random **initialization vector
  (IV)**, sometimes called a starting variable, sent alongside the ciphertext. Because each
  block's encryption now depends on everything before it, identical plaintext blocks produce
  different ciphertext — this is what actually defeats the ECB pattern-leak and large-scale
  key-reuse frequency analysis. A CBC IV must be unpredictable and never reused with the same
  key for different messages; reusing an IV reintroduces exactly the pattern leak CBC exists
  to prevent.
- **CTR (Counter)** / **GCM (Galois/Counter Mode)** — turn the block cipher into a stream
  cipher by encrypting a counter (or counter+nonce) and XORing the result with the plaintext.
  GCM additionally produces an authentication tag, protecting against tampering — a plain
  cipher (AES-CBC alone) only provides confidentiality, not integrity; without a MAC, an
  attacker can flip ciphertext bits and cause predictable plaintext changes on decryption.

```python
# Why ECB leaks structure: same 16-byte plaintext block -> same ciphertext block, every time.
# CBC breaks this by chaining in the previous ciphertext (or IV, for the first block).
def cbc_encrypt_blocks(plaintext_blocks, key, iv, encrypt_block):
    ciphertext_blocks = []
    prev = iv
    for block in plaintext_blocks:
        mixed = bytes(p ^ c for p, c in zip(block, prev))
        ct = encrypt_block(mixed, key)
        ciphertext_blocks.append(ct)
        prev = ct  # chain: next block's XOR input depends on this block's output
    return ciphertext_blocks
```

## Brute-force math: sizing key strength

A 128-bit key has 2^128 possible values — roughly 3.4 x 10^38. Even at one billion key
attempts per second, exhausting half the key space (the expected point of finding the right
key by random search) takes on the order of 10^21 years. This is the correct way to reason
about "is N bits enough": compute attacker throughput x available time and compare to key
space, don't rely on intuition about how big the number "feels." Note that key size is not
the only variable — a flawed key-derivation function (e.g., deriving an AES key from a short,
guessable password with no salt/stretching) can make the *effective* search space far smaller
than the nominal key size suggests; see `password-hashing-and-cracking-defenses` for that
failure mode specifically.

## Common pitfalls

- **Using ECB mode** because it's the library default or "simplest" — it leaks plaintext
  structure whenever any block repeats. Default to an authenticated mode (GCM) unless you have
  a specific, understood reason not to.
- **Reusing an IV/nonce** across messages encrypted with the same key in CBC/CTR/GCM — this
  reintroduces the pattern-leak problem chaining exists to solve, and in GCM specifically,
  nonce reuse can fully break both confidentiality and the authentication tag.
- **Treating encryption as providing integrity.** Plain AES-CBC gives confidentiality only; an
  attacker who can't read the plaintext can often still flip bits and produce a predictable
  change after decryption. Use an authenticated mode (GCM) or add a MAC.
- **Assuming "more rounds/bigger key" always means "more secure" in a way that matters.** Past
  a certain point (AES-128 is already computationally infeasible to brute-force), the
  realistic attack surface is implementation flaws (timing attacks, bad key derivation, IV
  reuse, side channels) — not brute force against the cipher itself.
- **Rolling your own cipher or mode.** The transposition/substitution/diffusion design space
  is full of subtle mistakes (weak S-boxes, insufficient rounds, correlated round keys) that
  only show up under serious cryptanalysis. Use vetted implementations (AES-GCM, ChaCha20-
  Poly1305) rather than reinventing the primitives described here.

## Checklist

- [ ] Identified which mode of operation is in use, and confirmed it isn't ECB for
      multi-block data
- [ ] Confirmed the IV/nonce is randomly generated per-message and never reused with the
      same key
- [ ] Confirmed the scheme provides integrity (authenticated mode or explicit MAC), not just
      confidentiality
- [ ] Sized key strength by computing attacker throughput x time vs. key space, not by
      intuition
- [ ] Checked whether the actual key material is derived from a high-entropy source, not a
      short/guessable password without proper stretching (see password-hashing skill)
- [ ] Used a vetted library/primitive rather than a custom substitution-permutation design
