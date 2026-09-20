# safelight

**Category:** Steganography / Forensics
**Author:** Arylide
**Difficulty:** Medium

## Description

> Carry this. It is not reissued.
>
> Every run on this line is keyed off the run before it. There is
> no key book, there is no key of the day, and nobody is coming
> to give you one. The seal you took off the last run IS the key
> to this one. That is the whole of the discipline.

Two frames came off the same plate — one a negative, one a print.
They should be perfect inverses of each other. They almost are.

Find what doesn't invert, and figure out what to do with it.

## Files

```
negative.png
print.png
keycard.txt
```

`keycard.txt` explains the key schedule used for this stage and every
stage after it — read it carefully, it's not flavor text.

## Flag format

```
Null0rigin{lowercase_words_with_underscores}
```

## Notes

- This stage is keyed off the flag from the previous stage in the chain.
  If you're running `safelight` standalone, use the seed value provided
  by the challenge host.
- Don't trust everything that's easy to read. Some things in these files
  are true, some are decoys, and both were made to look convincing.
- There is no verifier binary. Wrong key gives you nothing — no partial
  credit, no signal, no error message pointing you in the right
  direction. If it doesn't come out clean, it's wrong.

## Hints

<details>
<summary>Hint 1</summary>

Check the two images against each other pixel-for-pixel. Perfect
inverses agree everywhere. Yours won't.

</details>

<details>
<summary>Hint 2</summary>

Not every place they disagree means the same thing. Some disagreements
are just pictures. One kind isn't.

</details>

<details>
<summary>Hint 3</summary>

Once you've isolated the real signal, `keycard.txt` tells you exactly
how to turn a seal into a key, and a key into a stream.

</details>
