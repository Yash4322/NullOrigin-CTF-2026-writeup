# driftwood

**Category:** Steganography / Forensics
**Author:** Arylide
**Difficulty:** Medium
**Stage:** 31 of 8 — *Null0rigin: Stego Chain*

## Description

> Sleeve 4. Winding gear, no. 3 shaft, from the yard. Water damaged, emulsion
> lifting at the corner.
>
> The plate room does not write on plates. When something goes *into* a
> plate, it goes in as a **works entry** — and a works entry is always the
> same shape. Learn to trust the stamp and the check, and you won't waste a
> week. This room is full of handwriting. None of it is worth the paper.

You are given a single image, `driftwood.png`, and an index card describing
how data is stored in it. Not everything that looks like a message is the
message.

## Files

| File | Description |
|---|---|
| `driftwood.png` | The carrier image (900×620, RGBA) |
| `plate-card.txt` | Index card describing the "works entry" format |
| `description.txt` | Empty — flavor only |

## Objective

Recover the flag hidden inside `driftwood.png` and use it as the key for the
next stage in the chain.

## Format

```
Null0rigin{lowercase_words_with_underscores}
```

## Hints

- The plate card's "works entry" layout is a spec, not flavor text — read it
  as a literal binary format: a 3-byte stamp, a 1-byte version, a 2-byte
  length (little-endian), a 4-byte check (little-endian), then the payload.
- "The check is the ordinary one" — nothing exotic.
- Not every bit-plane in this image is meaningful. If what you extract
  doesn't begin with the stamp and the check doesn't validate, it's not the
  answer — keep looking at other channels, bit orders, and scan directions.
- Keep an untouched copy of the file. Re-saving or re-encoding this image
  will destroy the hidden data. Verify integrity against `SHA256SUMS.txt`.

## Flag

Submit the recovered flag to progress to `32-safelight`.
