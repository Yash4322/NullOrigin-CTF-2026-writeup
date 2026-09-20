# CTFCeption — Write-up

**Author:** Arylide
**Challenge:** CTFCeption
**Category:** Web / Crypto / Forensics
**Target:** [CTFception Archive Protocol](https://ctfception.onrender.com/)

## Overview

The challenge presents a fictional historical archive dedicated to Lokmanya Bal Gangadhar Tilak. The page contains a terminal-like interface, an archive explorer, a carrier frequency of **25.09 MHz**, and several clues about PRX02 and a lost transmission.

The solution requires following a chain of nested artifacts:

1. Inspect the virtual filesystem.

1. Read the critical telemetry file.

1. Use the historical clue to unlock an encrypted ZIP archive.

1. Decode an XOR-protected telegraph beacon from the ZIP.

1. Submit the recovered beacon phrase to the web terminal.

The final flag is:

```
NullOrigin{L0km4ny4_G4th4_25S3pt_Sw4r4jy4_PRX02_}
```

## Initial Reconnaissance

Opening the challenge reveals the following important information:

- Archive node: `PRX02-ALPHA`

- Carrier frequency: `25.09 MHz`

- Case file: `TILAK-1908`

- Available terminal commands such as `files`, `cat`, `inspect`, `scan`, and `reconstruct`

The page also warns that every archive hides another archive. This indicates that the visible terminal is only the first layer of the challenge.

The first useful command is:

```
help
```

The terminal shows commands for listing and reading the virtual filesystem. The next step is to enumerate the available files.

```
files
```

The relevant part of the output is:

```
/archive
├── records/
│   ├── kesari_1881_manifesto.txt
│   └── swaraj_declaration_1916.txt
├── transmissions/
│   ├── kesari_wire_1908.log
│   └── signal_intercept.raw
├── evidence/
│   ├── telemetry.json              [CRITICAL]
│   └── surveillance_report_1897.txt
└── old/
    └── corrupted_sector_09.bak     [DECOY]

/downloads
├── kesari_vault_1908.zip           [ENCRYPTED VAULT]
└── extract_vault.py                 [HELPER TOOL]
```

The file `corrupted_sector_09.bak` is explicitly marked as a decoy, so it can be ignored. The critical file is `telemetry.json`.

## Reading the Telemetry

Using the terminal command below displays the telemetry record:

```
cat /archive/evidence/telemetry.json
```

The important fields are:

```json
{
  "carrier_frequency": "25.09 MHz",
  "path": "downloads/kesari_vault_1908.zip",
  "access_key_hint": "The vault passphrase is the romanized title of Tilak's monumental philosophical work composed secretly by pencil inside Mandalay Jail between 1908 and 1914 (lowercase, single word, 11 letters).",
  "telegraph_cipher_hint": "Once inside the vault, the telegraph beacon payload requires the chronicle launch coordinate key (DDMM format: 2509)."
}
```

The historical clue refers to Tilak's philosophical work **Shrimadh Bhagavad Gita Rahasya**, commonly shortened to **Gita Rahasya**. The helper script confirms the expected lowercase, single-word form:

```
gitarahasya
```

The telemetry also provides the second key, `2509`, which will be needed after opening the ZIP archive.

## Unlocking the Encrypted ZIP

The virtual filesystem provides a helper script named `extract_vault.py`. After downloading the ZIP and the helper script, the archive can be extracted with:

```bash
python3 extract_vault.py gitarahasya
```

The archive is successfully unlocked and contains three files:

```
MANDALAY_DISPATCH_1908.txt
beacon_payload.enc
telegraph_decoder.py
```

The dispatch explains the next stage and confirms that the telegraph payload must be decoded using the key `2509`.

## Decoding the Telegraph Beacon

The supplied `telegraph_decoder.py` script performs the following operations:

1. Reads `beacon_payload.enc`.

1. Base64-decodes its contents.

1. XORs the decoded bytes with the repeating key `2509`.

1. Prints the recovered beacon phrase.

Run the decoder from inside the extracted directory:

```bash
python3 telegraph_decoder.py 2509
```

The result is:

```
[+] Decoded Telegraph Beacon: LOKMANYA_GATHA_CHRONICLE_1908
[+] Terminal Command: reconstruct LOKMANYA_GATHA_CHRONICLE_1908
```

## Reconstructing the Transmission

The recovered phrase is submitted to the web terminal exactly as instructed:

```
reconstruct LOKMANYA_GATHA_CHRONICLE_1908
```

The terminal performs an AES-256-GCM reconstruction using the SHA-256 digest of the submitted beacon phrase. The authentication succeeds, and the archive is restored.

## Flag

```
NullOrigin{L0km4ny4_G4th4_25S3pt_Sw4r4jy4_PRX02_}
```

## Complete Solve Path

For reference, the entire chain is:

```
files
cat /archive/evidence/telemetry.json

Vault password:
gitarahasya

Telegraph key:
2509

Decoded beacon:
LOKMANYA_GATHA_CHRONICLE_1908

Final terminal command:
reconstruct LOKMANYA_GATHA_CHRONICLE_1908
```

## Key Takeaways

The challenge is designed as a layered archive investigation rather than a single cryptographic task. The visible terminal reveals the filesystem, the telemetry provides the historical password clue, the ZIP contains the decoder for the next layer, and the final beacon phrase is validated by the site's AES-GCM reconstruction routine.

The main lesson is to follow the intended artifact chain and distinguish useful clues from explicit decoys. The carrier frequency `25.09 MHz` is also reused as the numerical key `2509`, while the historical context points to `gitarahasya` as the vault password.

## References

[1]: https://ctfception.onrender.com/ "CTFception Archive Protocol — Lokmanya Tilak challenge"

[2]: https://github.com/danifus/pyzipper "pyzipper — Python support for AES-encrypted ZIP archives"

## Author

Write-up by **Arylide**.

> Some stories survive because someone keeps telling them.— PRX02 Archive Protocol
