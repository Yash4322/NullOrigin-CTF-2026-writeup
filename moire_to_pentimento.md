# Null0rigin Archive — Writeup: Stages 33 (moire) → 35 (pentimento)

A solve writeup for three consecutive stages of the `stego_chain` puzzle set. Each stage's flag is fed into `SHA-256` (per `keycard.txt`) to become the decryption key for the **next** stage, so these solves chain directly off one another.

---

## Shared mechanics

### Key schedule (`keycard.txt`)

Every run after the first is keyed off the previous stage's seal:

```
k      = SHA-256(previous_seal_text_exactly_as_written)   # braces included, no strip/re-case/newline

number = first 4 bytes of k, high byte first
stream = SHA-256(k || counter_BE32) for counter = 0, 1, 2, ...   # digests concatenated, cut to length
entry  = stream XOR ciphertext, byte for byte, from the front
```

### Payload format (`plate-card.txt`)

Every hidden payload, once decrypted, is a **works entry**:

| field    | size    | notes                                     |
|----------|---------|---------------------------------------------|
| `stamp`  | 3 bytes | literal `PL8`                              |
| `issue`  | 1 byte  | constant, `0x01` throughout this archive   |
| `length` | 2 bytes | little-endian, length of `entry`           |
| `check`  | 4 bytes | little-endian CRC32 of `entry`             |
| `entry`  | N bytes | the actual payload / flag text             |

The CRC32 check is what confirms a decode is *correct* rather than merely plausible.

---

## Stage 33 — moire

**Carrier:** `moire.png` — 1024×768, 1-bit halftone image.
**Key:** `SHA-256("Null0rigin{two_cells_of_equal_weight_are_not_equal}")` (the 32-safelight seal)

### The trick

`screen-notes.txt` describes a halftone screen ruled at **4×4 pixels per cell**. A cell's *weight* (count of black sub-pixels, 0–16) is dictated by the image's tone, but for "middle" weights the same weight can be arranged **two different ways** in the cell — visually identical from a distance, but distinguishable pixel-for-pixel.

Verified empirically: weights 0–4 and 12 have exactly **one** distinct 4×4 pattern each; weights 5–11 have exactly **two**. That's the one free bit per eligible cell.

```python
from PIL import Image
im = Image.open('moire.png').convert('1')
w, h = im.size
px = im.load()
data = [[1 if px[x, y] else 0 for x in range(w)] for y in range(h)]

cs = 4
cols, rows = w // cs, h // cs
cellblocks = {}
from collections import defaultdict
patterns = defaultdict(set)
for cy in range(rows):
    for cx in range(cols):
        by, bx = cy * cs, cx * cs
        block = tuple(data[by+dy][bx+dx] for dy in range(cs) for dx in range(cs))
        cellblocks[(cy, cx)] = block
        patterns[sum(block)].add(block)

# weights 0-4, 12 -> 1 pattern each; weights 5-11 -> 2 patterns each
patlist = {wt: sorted(pats) for wt, pats in patterns.items() if 5 <= wt <= 11}
```

Reading order is a **boustrophedon** ("the ox ploughs") — row-major, alternating direction each row. For every weight-5–11 cell, whichever of its two patterns appears (lexicographically smaller = 0, larger = 1) is one raw ciphertext bit.

### Solving it

1. Extract the choice bit for every eligible cell, in boustrophedon order.
2. Because the 0/1 labeling isn't fixed purely by sort order, brute-force the per-weight-class polarity (7 weight classes → 128 combinations) alongside scan direction, bit offset, and bit-packing order.
3. Pack to bytes, XOR against the `keycard.txt` keystream.
4. Search for `PL8`, parse the header, verify CRC32.

```python
import itertools, hashlib, zlib

weights = [5, 6, 7, 8, 9, 10, 11]
wt_index = {wt: i for i, wt in enumerate(weights)}

# boustrophedon row-major scan order
seq = []
for cy in range(rows):
    xr = range(cols) if cy % 2 == 0 else range(cols - 1, -1, -1)
    for cx in xr:
        seq.append((cy, cx))

seqdata = []
for (cy, cx) in seq:
    block = cellblocks[(cy, cx)]
    wt = sum(block)
    if 5 <= wt <= 11:
        idx = patlist[wt].index(block)
        seqdata.append((wt_index[wt], idx))

prev = 'Null0rigin{two_cells_of_equal_weight_are_not_equal}'
k = hashlib.sha256(prev.encode()).digest()

def keystream(nbytes):
    out = bytearray(); counter = 0
    while len(out) < nbytes:
        out.extend(hashlib.sha256(k + counter.to_bytes(4, 'big')).digest())
        counter += 1
    return bytes(out[:nbytes])

for polvec in itertools.product([0, 1], repeat=7):
    bits = [idx ^ polvec[wi] for (wi, idx) in seqdata]
    n = len(bits) - (len(bits) % 8)
    by = bytearray()
    for i in range(0, n, 8):
        v = 0
        for bit in bits[i:i+8]:
            v = (v << 1) | bit
        by.append(v)
    raw = bytes(by)
    ks = keystream(len(raw))
    dec = bytes(a ^ b for a, b in zip(raw, ks))
    idx = dec.find(b'PL8')
    if idx < 0:
        continue
    hdr = dec[idx:idx+10]
    length = hdr[4] | (hdr[5] << 8)
    check = hdr[6] | (hdr[7] << 8) | (hdr[8] << 16) | (hdr[9] << 24)
    entry = dec[idx+10:idx+10+length]
    if zlib.crc32(entry) & 0xffffffff == check:
        print("FOUND:", entry)
```

### Result

```
Null0rigin{a_carrier_laid_across_the_whole_room}
```

---

## Stage 34 — undertone

**Carriers:** `linetest.wav` (16s, known-plaintext calibration file) and `undertone.wav` (40s, the real target). Both 44.1kHz, 16-bit, stereo.
**Key:** `SHA-256("Null0rigin{a_carrier_laid_across_the_whole_room}")` (the 33-moire seal)

### The trick

`line-notes.txt` describes a **BPSK direct-sequence spread-spectrum** message hidden in the **left channel**. The right channel is just a 50Hz mains-hum "tone." Each data bit is spread across many audio samples by multiplying it with a pseudorandom ±1 chip sequence (generated the same way as the `keycard.txt` keystream, one keystream bit per audio sample). Decoding a bit = multiply the sample window by the chip sequence and sum (matched filter); the sign of the sum is the bit.

`linetest.wav` is keyed differently — on the literal text `LINE TEST` instead of a seal — and its known plaintext (`NULL0RIGIN LINE TEST 1 OF 1`) is used to reverse-engineer the exact chip rate.

### Solving it

**Step 1 — calibrate on `linetest.wav`:**

```python
import wave, numpy as np, hashlib

def load(f):
    w = wave.open(f, 'rb')
    arr = np.frombuffer(w.readframes(w.getnframes()), dtype=np.int16).reshape(-1, 2)
    return arr, w.getframerate()

arr, sr = load('linetest.wav')
left = arr[:, 0].astype(np.float64)

k = hashlib.sha256(b"LINE TEST").digest()

def keystream_bits(nbits):
    nbytes = (nbits + 7) // 8
    out = bytearray(); counter = 0
    while len(out) < nbytes:
        out.extend(hashlib.sha256(k + counter.to_bytes(4, 'big')).digest())
        counter += 1
    out = bytes(out[:nbytes])
    bits = np.zeros(nbits, dtype=np.int8)
    for i in range(nbits):
        bit = (out[i // 8] >> (7 - (i % 8))) & 1
        bits[i] = 1 if bit else -1
    return bits

# brute-force samples-per-bit by correlation match rate against known plaintext
# (known header bytes 'PL8'+issue=1, and known entry text at byte offset 10)
# ... sweep spb, pick the value that gives ~100% bit-match ...
# result: spb = 2050 gives perfect agreement, CRC32 validates
```

Confirmed: **2050 samples per bit**, matched-filter (correlate + sign) decoding, MSB-first byte packing.

**Step 2 — apply to `undertone.wav`** with the real key:

```python
arr, sr = load('undertone.wav')
left = arr[:, 0].astype(np.float64)
total_samples = len(left)

prev = 'Null0rigin{a_carrier_laid_across_the_whole_room}'
k = hashlib.sha256(prev.encode()).digest()

# sweep spb near 2050 (small timing drift over the longer 40s clip)
# for each candidate spb: correlate, threshold, pack bytes, find 'PL8', check CRC32
# spb = 2047 / 2048 / 2049 all validate identically:
```

### Result

```
Null0rigin{the_order_of_the_colours_is_the_message}
```

---

## Stage 35 — pentimento

**Carrier:** `pentimento.gif` — an 11-frame animated GIF, 480×360, each frame with its **own local 256-color palette**.
**Key:** `SHA-256("Null0rigin{the_order_of_the_colours_is_the_message}")` (the 34-undertone seal)

### The trick (and the gotcha)

`dish-notes.txt` talks about a "tray" (palette) of 256 distinct colors laid out **two to a slot** (128 pairs). The "correct" order is darkest-to-lightest, red before green before blue (i.e. sort each frame's own colors ascending by `(R, G, B)`), but the colourman is "careless" about which of the two colors in each pair goes left — that carelessness is one bit per pair, 128 bits per frame.

**Gotcha:** Pillow (PIL) caches the first frame's palette and silently reuses it on subsequent `.seek()` calls for some animated GIFs — so naive extraction via `im.palette.palette` looked *identical* across all 11 frames. The raw GIF bytes tell a different story: each frame genuinely has its own distinct Local Color Table. Parse the GIF manually to get true per-frame palettes:

```python
import hashlib, zlib

data = open('pentimento.gif', 'rb').read()
packed = data[10]
gct_flag = (packed >> 7) & 1
gct_size = 2 ** ((packed & 7) + 1)
i = 13 + (3 * gct_size if gct_flag else 0)

lcts = []
while i < len(data):
    b = data[i]
    if b == 0x21:  # extension block
        label = data[i+1]; i += 2
        if label == 0xFF:  # application extension
            blocksize = data[i]; i += 1 + blocksize
            while data[i] != 0:
                sb = data[i]; i += 1 + sb
            i += 1
        elif label == 0xF9:  # graphic control extension
            blocksize = data[i]; i += 1 + blocksize; i += 1
        elif label in (0xFE, 0x01):  # comment / plain text
            while data[i] != 0:
                sb = data[i]; i += 1 + sb
            i += 1
    elif b == 0x2C:  # image descriptor
        packed2 = data[i+9]
        lct_flag = (packed2 >> 7) & 1
        lct_size = 2 ** ((packed2 & 7) + 1) if lct_flag else 0
        i += 10
        if lct_flag:
            lcts.append(data[i:i + 3 * lct_size])
            i += 3 * lct_size
        lzw_min = data[i]; i += 1
        while data[i] != 0:
            sb = data[i]; i += 1 + sb
        i += 1
    elif b == 0x3B:  # trailer
        break

palettes = [[tuple(lct[j*3:j*3+3]) for j in range(256)] for lct in lcts]
```

Once parsed correctly, each frame's palette really is its own distinct permutation. For each frame:

```python
allbits = []
for pal in palettes:
    canonical = sorted(set(pal))          # each frame sorted independently
    bits = []
    for i in range(128):
        # physical adjacent pair always matches the canonical pair — just order can flip
        bits.append(0 if pal[2*i] == canonical[2*i] else 1)
    allbits.append(bits)

flatbits = [b for frame in allbits for b in frame]   # frame-major, slot-ascending
```

That's 11 × 128 = 1408 raw bits. XOR against the `keycard.txt` keystream and look for the works entry:

```python
def bits_to_bytes(bits):
    n = len(bits) - (len(bits) % 8)
    by = bytearray()
    for i in range(0, n, 8):
        v = 0
        for bit in bits[i:i+8]:
            v = (v << 1) | bit
        by.append(v)
    return bytes(by)

raw = bits_to_bytes(flatbits)

prev = 'Null0rigin{the_order_of_the_colours_is_the_message}'
k = hashlib.sha256(prev.encode()).digest()

def keystream(n):
    out = bytearray(); c = 0
    while len(out) < n:
        out.extend(hashlib.sha256(k + c.to_bytes(4, 'big')).digest())
        c += 1
    return bytes(out[:n])

ks = keystream(len(raw))
dec = bytes(a ^ b for a, b in zip(raw, ks))

idx = dec.find(b'PL8')
hdr = dec[idx:idx+10]
length = hdr[4] | (hdr[5] << 8)
check  = hdr[6] | (hdr[7] << 8) | (hdr[8] << 16) | (hdr[9] << 24)
entry  = dec[idx+10:idx+10+length]
assert zlib.crc32(entry) & 0xffffffff == check
print(entry)
```

Note: the pixel data itself (the actual developing-photo image content) is an explicit decoy per the notes — "you will find what they left there... and you will have nothing." The palette order is the only channel that matters.

### Result

```
Null0rigin{keep_the_grain_and_burn_the_chaff}
```

---

## Summary of seals recovered

| Stage | Carrier | Key (SHA-256 of...) | Flag |
|---|---|---|---|
| 33 — moire | `moire.png` | 32-safelight seal | `Null0rigin{a_carrier_laid_across_the_whole_room}` |
| 34 — undertone | `undertone.wav` | 33-moire seal | `Null0rigin{the_order_of_the_colours_is_the_message}` |
| 35 — pentimento | `pentimento.gif` | 34-undertone seal | `Null0rigin{keep_the_grain_and_burn_the_chaff}` |

The 35-pentimento seal above is the key material for **36-winnow**.
