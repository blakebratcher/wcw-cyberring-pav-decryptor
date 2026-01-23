# WCW CyberRing PAV Decryptor

[![Python 3.6+](https://img.shields.io/badge/python-3.6+-3776AB.svg?logo=python&logoColor=white)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Recovery Rate](https://img.shields.io/badge/files%20recovered-61%2F61-brightgreen.svg)]()
[![Inaccessible](https://img.shields.io/badge/locked_for-25_years-critical.svg)]()

> Cryptanalysis of a proprietary 1999 video DRM system, recovering 51 minutes of
> lost media through static binary analysis and known-plaintext attack.

---

![Recovered CyberRing match - wrestlers in a CGI arena with composited fire effects, 320x240 MPEG-1](screenshot.png)

<p align="center"><em>Frame from a recovered "CyberRing" match — WCW wrestlers filmed against blue screen, composited into a CGI arena with fire effects. This is what 1999 interactive entertainment looked like.</em></p>

## Background

The **WCW Internet Powerdisk** was a promotional CD-ROM distributed with WCW Magazine
in 1999. It shipped 61 video clips — match highlights, wrestler profiles, show intros —
encrypted with a proprietary system called **PAVENCRYPT**, developed by a company called UIT.

Playback required the bundled ULI Player, which fetched per-file decryption keys from a
remote server at runtime. When UIT's server infrastructure went offline around 2000, the
content became permanently inaccessible. The player would only display:

> *"Decryption key not found in the server database"*

No keys exist on the disc. No documentation of the format was ever published. UIT appears
to have dissolved entirely. For 25 years, the content has been unplayable.

## Recovered Content

The recovered videos are available on the Internet Archive:

**[https://archive.org/details/wcw-cyberring-powerdisk-1999](https://archive.org/details/wcw-cyberring-powerdisk-1999)**

61 clips totaling 51 minutes 20 seconds, converted to H.264/AAC MP4. Original content
produced by WCW (Turner Broadcasting). IP now held by WWE.

## Usage

```bash
# 1. Extract the ISO
7z x WCW_R1.ISO -oiso_contents

# 2. Decrypt all PAV files to MPEG-1
python decrypt_pav.py

# 3. Convert to MP4 (optional, requires ffmpeg)
python convert_mp4.py
```

**Requirements:** Python 3.6+ (no third-party packages). ffmpeg for optional MP4 conversion.

## How It Works

### The Cipher

Each `.PAV` file is encrypted with a **repeating-key byte-subtraction cipher**:

```
plaintext[i] = (ciphertext[i] - key[i % key_length]) mod 256
```

Keys are unique per file, 8–24 bytes of printable ASCII, randomly generated at
encoding time. The decryption routine lives inside `PavSource.ax`, a 24 KB
DirectShow source filter built in April 1999:

```asm
mov  cl, [ebx+edx+0x3C]     ; load key[key_index]
sub  byte ptr [eax], cl      ; plaintext = ciphertext - key_byte
```

### The Attack

No brute force. No emulation. No server. The keys are recovered mathematically:

1. **MPEG-1 files end with known bytes** — `0xFF` padding followed by the
   Program End Code (`00 00 01 B9`)

2. **A repeating key over a constant produces a repeating pattern** — the
   encrypted tail has detectable periodicity equal to the key length

3. **Key recovery is algebraic** — `key[i] = (ciphertext[i] - 0xFF) mod 256`

4. **Verification is structural** — decrypted output must begin with a valid
   MPEG-1 Pack Start Code (`00 00 01 BA`)

The entire attack executes in seconds with no external input.

### File Format

```
┌─────────────────────────────────────────────────┐
│ "PAVENCRYPT" magic (10 bytes)                   │
├─────────────────────────────────────────────────┤
│ Unencrypted MPEG-1 preview (16–32 KB)           │
│ 2 video frames, no audio — serves as thumbnail  │
├─────────────────────────────────────────────────┤
│ Encrypted MPEG-1 Program Stream                 │
│ Per-file subtraction key, 8–24 bytes            │
│                                                 │
│ Video: 320×240, 30fps, ~1.29 Mbps               │
│ Audio: MP2, 44.1 kHz, mono, 64 kbps            │
└─────────────────────────────────────────────────┘
```

## Disc Contents

61 clips spanning WCW's late-1999 roster:

| Category | Count | Highlights |
|----------|------:|------------|
| Match footage | 9 | Hogan vs. Goldberg, Hogan vs. Macho Man, CyberRing matches |
| Wrestler bios | 13 | Goldberg, Sting, Hogan, DDP, Flair, Nash, Steiner |
| Show intros | 5 | Monday Nitro, Thunder, Surge |
| Storyline segments | 8 | "Hacker" serial narrative |
| Promos & merchandise | 9 | Wrestler shirts and merchandise spots |
| Other segments | 17 | Nitro Girls, Nasty Boys matches, Rey Mysterio, Konnan |

## Repository

```
decrypt_pav.py       Decryptor — key recovery + decryption to MPEG-1
convert_mp4.py       Transcoder — batch MPEG-1 to H.264/AAC MP4
analyze_pav.py       Inspector — non-destructive PAV structure analysis
PavSource.ax         Original DirectShow filter binary (reversed)
UlPlayer.exe         Original ULI Player application
FINDINGS.md          Technical report: format spec, disassembly, all 61 keys
TIMELINE.md          Reverse engineering methodology, step by step
```

## Broader Applicability

This tool works on any PAVENCRYPT-format file. ULI Player was licensed to multiple
content distributors in the late 1990s. The attack generalizes to any repeating-key
cipher applied to a file format with predictable byte patterns — which describes
most proprietary DRM schemes from that era.

## License

[MIT](LICENSE)
