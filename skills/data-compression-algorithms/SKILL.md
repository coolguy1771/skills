---
name: data-compression-algorithms
description: Explain, choose between, or implement lossless/lossy data compression techniques (run-length encoding, dictionary/LZ compression, Huffman coding, predictive/delta encoding, quantization) rather than treating compression as a mystery black box. Use this when the user asks "how does zip/gzip/deflate actually work", "how does Huffman coding work", "why is my data compressing poorly", "what's the difference between lossy and lossless compression", "how would I compress this data format", is designing a custom binary/wire format and wants to shrink it, is deciding between a simple custom compression scheme vs. a general-purpose library, or is implementing/reviewing a run-length or dictionary-based encoder/decoder. Also trigger for "what does 'prefix code' mean", entropy/compressibility intuition, or questions about why images/video need different compression than text.
---

# Data Compression Algorithms

Compression exploits **redundancy** — literally, the fewer surprises data contains, the
smaller it can be made without losing information. Every classical lossless technique is a
different way of finding and exploiting a specific kind of redundancy: runs of identical
values (run-length encoding), repeated substrings (dictionary/LZ methods), or skewed symbol
frequency (Huffman coding). Lossy compression adds a fourth move: deliberately throwing away
information the consumer (usually a human eye or ear) is least likely to notice. Knowing which
kind of redundancy your data actually has is the whole game — applying the wrong technique to
data that lacks its target redundancy buys you little or nothing.

## Run-length encoding (RLE): exploit runs

If data has long stretches of the same value, replace each run with `(value, count)` instead
of storing the value repeatedly. Extremely effective on flat-color icons/logos, mediocre-to-
useless on photographic images or generic text, where exact repeats are rare.

```python
def rle_encode(data: bytes) -> list[tuple[int, int]]:
    if not data:
        return []
    runs = []
    current, count = data[0], 1
    for byte in data[1:]:
        if byte == current:
            count += 1
        else:
            runs.append((current, count))
            current, count = byte, 1
    runs.append((current, count))
    return runs

def rle_decode(runs: list[tuple[int, int]]) -> bytes:
    return bytes(value for value, count in runs for _ in range(count))
```

Real formats (e.g., TGA) pack this more tightly: a header byte's top bit distinguishes a "run"
packet from a "raw" packet (a sequence of dissimilar pixels stored uncompressed), so RLE
degrades gracefully — non-repeating stretches cost only a small header overhead instead of
expanding the data.

## Dictionary compression: exploit repeated substrings

Build a table of frequently occurring units (words, byte sequences) and replace each
occurrence with a short reference into the table. This wins whenever the same substring
recurs many times relative to the table's own size — the dictionary itself takes space, so
compression only pays off once repetition outweighs that overhead. A 200-word children's story
compresses poorly this way (dictionary is a large fraction of the content); an 80,000-word
novel compresses well (vocabulary is a few thousand words, reused constantly).

**Sliding-window compression (the basis of LZ77/deflate/gzip/zip)** generalizes this without
needing a precomputed dictionary at all: replace a repeated run with a `(distance, length)`
pair meaning "copy `length` bytes starting `distance` bytes back in the already-decoded
output."

```python
def sliding_window_encode(data: str, window: int = 32) -> list:
    """Minimal LZ77-style encoder: emits (distance, length, next_char) tuples."""
    out, i = [], 0
    while i < len(data):
        best_len, best_dist = 0, 0
        start = max(0, i - window)
        for j in range(start, i):
            length = 0
            while (i + length < len(data)
                   and data[j + length] == data[i + length]
                   and length < i - j):
                length += 1
            if length > best_len:
                best_len, best_dist = length, i - j
        if best_len >= 2:
            next_char = data[i + best_len] if i + best_len < len(data) else ""
            out.append((best_dist, best_len, next_char))
            i += best_len + 1
        else:
            out.append((0, 0, data[i]))
            i += 1
    return out
```

This is exactly the mechanism behind `.zip`'s deflate algorithm: matches are found in a
sliding window over recently seen data, and the resulting `(distance, length)` pairs are then
further compressed with Huffman coding (below) — real formats layer techniques rather than
picking just one.

## Huffman coding: exploit skewed frequency

Even without repeated substrings, if some symbols occur far more often than others, assign
**shorter bit codes to the most frequent symbols** and longer codes to rare ones — the same
idea as Morse code giving `E` a single dot. The critical constraint is the **prefix property**:
no code may be a prefix of another code. Without it, a bitstream can't be unambiguously
decoded, because the decoder wouldn't know where one code ends and the next begins.

```python
import heapq
from collections import Counter

def build_huffman_codes(text: str) -> dict[str, str]:
    freq = Counter(text)
    heap = [[weight, [symbol, ""]] for symbol, weight in freq.items()]
    heapq.heapify(heap)
    if len(heap) == 1:                     # edge case: single unique symbol
        return {heap[0][1][0]: "0"}
    while len(heap) > 1:
        lo = heapq.heappop(heap)
        hi = heapq.heappop(heap)
        for pair in lo[1:]:
            pair[1] = "0" + pair[1]
        for pair in hi[1:]:
            pair[1] = "1" + pair[1]
        heapq.heappush(heap, [lo[0] + hi[0]] + lo[1:] + hi[1:])
    return {symbol: code for symbol, code in heap[0][1:]}

def huffman_encode(text: str, codes: dict[str, str]) -> str:
    return "".join(codes[ch] for ch in text)
```

Verify the prefix property holds by construction: because the algorithm only assigns codes at
the **leaves** of a binary tree built bottom-up (merging the two lowest-frequency nodes
repeatedly), no code can ever be a proper prefix of another — every code corresponds to a
distinct leaf, and no leaf is an ancestor of another leaf. When debugging a "decoder produces
garbage / decodes something different than what was encoded" bug in a from-scratch Huffman
implementation, check first whether the code table was actually built from a valid tree
(every internal node has exactly two children) rather than an ad hoc frequency-sorted list —
an ad hoc list is exactly how the prefix property gets accidentally violated.

## Reorganizing data before compressing it

Sometimes source data has no exploitable redundancy in its raw form but does once
**transformed**. Two general moves:

- **Predictive (delta) encoding**: store the *difference* between consecutive values instead
  of the values themselves. A slowly changing sensor reading (`51, 52, 53, 54, 55, 55, 56...`)
  has almost no repeats, but its deltas (`1, 1, 1, 1, 0, 1...`) do — small, repeated integers
  that Huffman/RLE can then exploit. This is the same principle behind video codecs encoding
  inter-frame differences instead of full frames.
- **Quantization** (lossy): store data with deliberately reduced precision, when the discarded
  precision is below the threshold of what matters (e.g., dropping the last decimal digit of a
  rainfall sensor reading below the noise floor of the instrument). This trades a small,
  bounded accuracy loss for both a direct size reduction and — because rounding creates more
  duplicate/near-duplicate values — better downstream RLE/Huffman compression.

The general multistage recipe for compressing data that doesn't compress well as-is:
**reorganize first (delta-encode or quantize) to manufacture runs/duplicates, then apply
RLE/Huffman on the reorganized form.** Reach for this whenever a straightforward encoding
attempt yields disappointing ratios — the fix is often upstream of the compression step, in
how the data is represented before compression even runs.

## Why images and audio/video need a different pipeline (JPEG's key ideas)

Photographic images have no long runs and little exact repetition, so RLE and dictionary
methods don't help directly. JPEG's approach — worth understanding even if you never implement
it — layers two domain-specific insights on top of the tools above:

1. **Perceptual weighting**: convert RGB to a luminance/chrominance representation (Y = 
   brightness, Cb/Cr = color difference) because human vision is far more sensitive to
   brightness changes than color changes. Compress the chrominance channels more aggressively
   than luminance — a direct application of "discard what's least likely to be noticed."
2. **Frequency-domain transform**: the discrete cosine transform converts a block of pixel
   values into a representation of how much the block varies at different spatial
   frequencies, concentrating most of the useful signal into a few coefficients. Small,
   high-frequency (fine-detail) coefficients can then be quantized heavily or dropped, and the
   resulting sparse, redundant data compresses well with RLE and Huffman.

The pattern to take away: general-purpose lossless techniques (RLE, LZ, Huffman) are usually
the *last* stage of a domain-specific pipeline, not a substitute for understanding what kind
of redundancy your specific data format actually has.

## Common pitfalls

- **Applying RLE to data with no runs** (e.g., photographic pixel data) — it will not help,
  and a naive implementation can even expand the data slightly (each non-repeating value still
  needs a run/raw marker).
- **Building a dictionary/table without accounting for its own storage cost.** Dictionary
  compression only wins once repetition outweighs the table size; on short or highly varied
  inputs, check the actual ratio rather than assuming compression always helps.
- **Violating the prefix property in a hand-built variable-length code**, producing ambiguous
  bitstreams that decode incorrectly for certain inputs (usually only surfacing on specific
  symbol sequences, making it a nasty intermittent bug). Always derive codes from a proper
  binary tree, not an ad hoc assignment.
- **Choosing lossy compression for data where exactness matters** (financial figures, source
  code, cryptographic material) — lossy techniques are appropriate for perceptual data (image/
  audio/video) where humans, not exact-match logic, consume the result.
- **Reinventing deflate/gzip instead of using it.** The sliding-window + Huffman combination
  shown here is exactly what `zlib`/`gzip`/`zip` already implement, heavily optimized; write a
  custom scheme only when you have a specific redundancy pattern general-purpose compressors
  don't exploit well (e.g., domain-specific delta encoding before feeding to gzip).
- **Ignoring reorganization before giving up on compression.** "This data doesn't compress"
  is often "this data doesn't compress *in its current representation*" — try delta encoding
  or quantization first.

## Checklist

- [ ] Identified which kind of redundancy the data actually has (runs, repeated substrings,
      skewed frequency, or none in its current form)
- [ ] Confirmed any prefix/variable-length code is derived from an actual binary tree, not an
      ad hoc frequency-sorted assignment
- [ ] Checked whether reorganizing the data (delta encoding, quantization) before compressing
      would manufacture exploitable redundancy
- [ ] Chose lossless vs. lossy deliberately based on whether exactness matters for this data
- [ ] Considered using a mature general-purpose compressor (deflate/gzip/zstd) before building
      a custom scheme, reserving custom logic for a preprocessing/reorganization step
- [ ] Measured actual compression ratio on representative data rather than assuming a
      technique will help
