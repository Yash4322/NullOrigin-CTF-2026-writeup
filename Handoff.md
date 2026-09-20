# Handoff (Forensics)

**Author of writeup:** Arylide
**Category:** Forensics
**Files provided:** `verify`, `handoff.pcap`, `handoff.sealed`, `description.txt`

## Challenge Description

> A handoff is the moment a thing stops being one party's problem.
> It is also the moment nobody is quite watching.

We're given a stripped 64-bit ELF (`verify`), a packet capture (`handoff.pcap`), and a sealed file (`handoff.sealed`) that only opens once you already have the flag.

## Initial Recon

Running `verify` with no arguments gives a usage string, and running it with a garbage flag gives a hint baked into the binary:

```
$ ./verify
usage: ./verify <flag>

$ strings verify | grep -A2 "Two devices"
Two devices, one rendezvous port, thirty seconds of overlap during the
handoff. The screen you were shown was never one continuous thing --
it was thirty-six scanlines, correctly sorted.
```

That's the whole challenge in one sentence: something in the pcap is a *screen*, reconstructed from **36 scanlines**, and there are **two devices** talking over the **same port**.

## Decoys First

Walking the pcap protocol by protocol turns up flag-shaped strings almost immediately — which is the point. `verify` explicitly states malformed guesses don't cost you an attempt, but *wrong, correctly-formatted* guesses do, so these need to be recognized as bait rather than submitted:

- **HTTP** (`GET /status`) — a `User-Agent` string and a `status: idle` response body each embed a flag:
  - `Null0rigin{handoff_incomplete}`
  - `Null0rigin{wrong_stream_wrong_flag}`
- **FTP** session — a plaintext `flag.txt` transfer:
  - `Null0rigin{the_call_dropped_first}`
- **DNS** — one query with three suspicious, base32-shaped subdomain labels:
```
  jz2wy3bqojuwo2lopnrh.kztgmvzgkzc7nzxxix3c.ojxwczddmfzxi7i.tun.io
```
  Concatenating the labels and base32-decoding yields a fourth decoy:
```python
  import base64
  labels = "jz2wy3bqojuwo2lopnrh" "kztgmvzgkzc7nzxxix3c" "ojxwczddmfzxi7i"
  print(base64.b32decode(labels.upper() + "="))
  # b'Null0rigin{buffered_not_broadcast}'
```

All four are well-formed and thematically plausible — none of them are it.

## The Real Signal: RTP Video

Filtering the pcap for UDP traffic on port `5004` reveals RTP packets from **two distinct SSRCs**:

| SSRC | Marker bit | Payload Type |
|---|---|---|
| `0x7e11a0c1` | set | 96 |
| `0x7e115ca0` | clear | 96 |

Both streams send **sequence numbers 0–35** (36 packets each — the "thirty-six scanlines"), with matching RTP timestamps, each carrying a **1092-byte payload of pure `0x00`/`0xFF` bytes** — a 1-byte-per-pixel monochrome bitmap.

Reassembling each stream by sequence number into a 1092×36 image (one scanline per packet, in order — the "correctly sorted" part) and rendering it reveals readable rasterized text:

```python
import struct
from PIL import Image

# parse pcap -> UDP/5004 packets -> group RTP payloads by SSRC, keyed by seq
# (standard pcap global header is 24 bytes, each record has a 16-byte header)
# ...
rows = [streams[ssrc][seq] for seq in range(36)]
w = len(rows[0])
img = Image.new('L', (w, 36))
img.putdata(b''.join(rows))
img.resize((w * 3, 36 * 8), Image.NEAREST).save('reconstructed.png')
```

- **Stream A** (`0x7e11a0c1`, marker bit **set**) renders:
```
  Null0rigin{the_right_feed_never_lied}
```
- **Stream B** (`0x7e115ca0`, marker bit **clear**) renders:
```
  Null0rigin{the_ghost_signal_answered}
```

The marker bit conventionally flags the start of a genuine talkspurt/frame in RTP — mechanically pointing at Stream A as the authentic feed, and thematically the two rendered strings say the same thing out loud: trust "the right feed," not the "ghost signal" left over from the handoff.

## Flag

```
Null0rigin{the_right_feed_never_lied}
```

Verified:

```
$ ./verify 'Null0rigin{the_right_feed_never_lied}'
CORRECT.  handoff solved.
```

## Takeaways

- Don't submit the first flag-shaped string you find — `verify`'s "malformed guesses are free" rule is a strong tell that the real challenge is separating real signal from red herrings, not finding *a* flag.
- RTP's marker bit and per-SSRC sequencing are exactly the metadata you need to disambiguate two overlapping streams on one port.
- When a hint says "scanlines," treat a payload as raw pixel data before looking for anything more exotic.
