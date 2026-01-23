# WCW CyberRing PAV Decryptor

[![Python 3.6+](https://img.shields.io/badge/python-3.6+-3776AB.svg?logo=python&logoColor=white)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Recovery Rate](https://img.shields.io/badge/recovery-61%2F61%20files-brightgreen.svg)]()
[![Locked Since](https://img.shields.io/badge/inaccessible%20since-1999-critical.svg)]()

> Cryptanalysis of a proprietary 1999 video DRM system, recovering 51 minutes of
> lost media through static binary analysis and known-plaintext attack.

---

![Recovered CyberRing match - wrestlers in a CGI arena with composited fire effects, 320x240 MPEG-1](screenshot.png)

<p align="center"><em>Recovered frame from TVKCYBER.mpg — a WCW match composited into a CGI arena. This content has been locked behind dead-server DRM for 25 years.</em></p>

## Background

The **WCW Internet Powerdisk** was a promotional CD-ROM distributed with WCW Magazine
in 1999. It shipped 61 video clips — match highlights, wrestler profiles, show intros —
encrypted with a proprietary system called **PAVENCRYPT**, developed by a company called UIT.

Playback required the bundled ULI Player, which fetched per-file decryption keys from a
remote server at runtime. When UIT's infrastructure went offline around 2000, the content
became permanently inaccessible. The player displays:

> *"Decryption key not found in the server database"*

No keys exist on the disc. No documentation of the format was ever published. UIT dissolved
without a trace. The content sat encrypted for 25 years.

## Results

| Metric | Value |
|--------|-------|
| Files recovered | 61 / 61 (100%) |
| Total runtime | 51 minutes 20 seconds |
| Video format | MPEG-1, 320x240, 30fps |
| Audio format | MPEG Layer 2, 44.1 kHz, mono |
| Method | Known-plaintext cryptanalysis |
| External dependencies | None (no server, no keys, no emulation) |

## Usage

```bash
# 1. Extract the ISO
7z x WCW_R1.ISO -oiso_contents

# 2. Decrypt all PAV files → MPEG-1
python decrypt_pav.py

# 3. Convert to MP4 (optional)
python convert_mp4.py
```

**Requirements:** Python 3.6+. ffmpeg for MP4 conversion. No other dependencies.

## How It Works

### The Cipher

Each PAV file is encrypted with a **repeating-key byte-subtraction cipher**:

```
plaintext[i] = (ciphertext[i] - key[i % key_length]) mod 256
```

Keys are unique per file, 8–24 bytes of random printable ASCII. The cipher was
implemented in 25 bytes of x86 inside `PavSource.ax`, a 24 KB DirectShow source filter:

```asm
mov  cl, [ebx+edx+0x3C]     ; load key byte
sub  byte ptr [eax], cl      ; subtract from ciphertext
```

### The Attack

No brute force. No emulation. The keys are recovered mathematically:

1. **MPEG-1 PS files end with known bytes** — 0xFF padding followed by the
   Program End Code (`00 00 01 B9`)

2. **Encrypting a constant with a repeating key produces a repeating pattern** —
   the ciphertext tail has detectable periodicity equal to the key length

3. **Key recovery is algebraic** — `key[i] = (ciphertext[i] - 0xFF) mod 256`

4. **Verification is structural** — decrypted bytes must produce valid MPEG-1
   start codes (`00 00 01 BA`)

The entire attack runs in seconds with zero external input.

### File Format

```
┌─────────────────────────────────────────────┐
│ "PAVENCRYPT" (10 bytes)                     │
├─────────────────────────────────────────────┤
│ Unencrypted MPEG-1 preview (16–32 KB)       │
│ 2 video frames, no audio — thumbnail only   │
├─────────────────────────────────────────────┤
│ Encrypted MPEG-1 Program Stream             │
│ Per-file key, 8–24 bytes, subtraction cipher │
└─────────────────────────────────────────────┘
```

## Documentation

| Document | Contents |
|----------|----------|
| [FINDINGS.md](FINDINGS.md) | Full technical report — format specification, annotated disassembly, all 61 keys, cryptographic assessment |
| [TIMELINE.md](TIMELINE.md) | Step-by-step reverse engineering methodology |

## Repository Contents

```
decrypt_pav.py      Decryptor — recovers keys and outputs clean MPEG-1
convert_mp4.py      Batch transcoder — MPEG-1 → H.264/AAC MP4
analyze_pav.py      Inspector — non-destructive PAV structure analysis
PavSource.ax        Original UIT DirectShow filter (reversed)
UlPlayer.exe        Original ULI Player application
```

## Disc Contents

61 clips spanning WCW's late-1999 roster:

| Category | Count | Examples |
|----------|-------|---------|
| Match footage | 9 | Hogan vs. Goldberg, Hogan vs. Macho Man, CyberRing matches |
| Wrestler bios | 13 | Goldberg, Sting, Hogan, DDP, Flair, Nash, Steiner |
| Show intros | 5 | Monday Nitro, Thunder, Surge |
| Storyline segments | 8 | "Hacker" serial narrative |
| Promos & merch | 9 | Wrestler merchandise spots |
| Other | 17 | Nitro Girls, Nasty Boys matches, misc |

## Broader Applicability

This tool works on **any PAVENCRYPT-format file**, not just this disc. ULI Player was
licensed to multiple content distributors in the late 1990s. The attack generalizes to
any repeating-key cipher over a file format with known byte patterns.

The methodology is documented in detail for application to similar dead DRM systems
from the same era.

## License

[MIT](LICENSE)
