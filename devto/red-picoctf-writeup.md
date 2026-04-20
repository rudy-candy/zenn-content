---
title: "RED picoCTF Writeup"
published: true
description: "
Introduction
This is my writeup for the picoCTF challenge RED — a forensics puzzle centered on LSB steganography hidden inside a PNG image. I used exiftoo"
tags: ctf, picoctf, forensics, security
canonical_url: https://alsavaudomila.com/red-picoctf-writeup/
---

## Introduction

This is my writeup for the picoCTF challenge **RED** — a forensics puzzle centered on LSB steganography hidden inside a PNG image. I used **exiftool** to find a suspicious poem in the metadata, decoded an acrostic clue pointing to **zsteg** , and extracted a Base64-encoded flag from the LSB layer. Fair warning: I hit a wall installing zsteg before I even got started.

* * *

## Challenge Overview

**CTF:** picoCTF  
**Challenge:** RED  
**Category:** Forensics  
**Difficulty:** Easy

The challenge description was minimal — almost taunting:

> RED, RED, RED, RED  
> Download the image: red.png

A single PNG. No hints. Just red. I had no idea what I was walking into.

* * *

## Step-by-Step Walkthrough

### Step 1: Start with `file` — Because Assumptions Kill

My first instinct with any unknown file is to run `file`. Not because I expect fireworks, but because I've been burned before. Once I tried to open a "PNG" that was actually a ZIP in disguise — and I wasted 20 minutes confused why my image viewer crashed. Never again.
    
    
    $ file red.png
    red.png: PNG image data, 128 x 128, 8-bit/color RGBA, non-interlaced

Legitimate PNG, 128×128 pixels, RGBA color space. Nothing suspicious here — which somehow made it more suspicious. A 128×128 image isn't exactly a high-resolution photo. Something was packed inside.

### Step 2: `exiftool` — Where Things Got Weird

When the file itself looks clean, I dig into metadata. `exiftool` reads EXIF and other embedded data that the image itself doesn't display visually. Most of the time it's boring — camera model, GPS coordinates, timestamps. This time it was not boring at all.
    
    
    $ exiftool red.png
    ExifTool Version Number         : 13.25
    File Name                       : red.png
    Directory                       : .
    File Size                       : 796 bytes
    File Type                       : PNG
    Image Width                     : 128
    Image Height                    : 128
    Bit Depth                       : 8
    Color Type                      : RGB with Alpha
    Poem                            : Crimson heart, vibrant and bold,
    Hearts flutter at your sight.
    Evenings glow softly red,
    Cherries burst with sweet life.
    Kisses linger with your warmth.
    Love deep as merlot.
    Scarlet leaves falling softly,
    Bold in every stroke.
    Image Size                      : 128x128

A **Poem** field. I had never seen that metadata field before. My first reaction was genuine confusion — who puts a poem in a PNG? But then I read it again. Something about the structure felt deliberate. Too deliberate.

### Step 3: The CHECKLSB Acrostic — I Almost Missed It

I copied the poem into a text editor and read it a few times looking for something — a URL, a hex string, anything obviously encoded. Nothing. I almost moved on to running `strings` on the raw file when something made me slow down. The poem felt constructed rather than natural. Every line started clean and capitalized. That's not how poems flow when they're written to be felt — that's how they flow when they're written to hide something.

I read the first letters vertically.

**C** rimson heart, vibrant and bold,  
**H** earts flutter at your sight.  
**E** venings glow softly red,  
**C** herries burst with sweet life.  
**K** isses linger with your warmth.  
**L** ove deep as merlot.  
**S** carlet leaves falling softly,  
**B** old in every stroke.

**C-H-E-C-K-L-S-B.** CHECKLSB.

I genuinely laughed. Not because it was funny, but because I almost didn't see it. I had been looking for something technical — encoded characters, unusual Unicode, anything that looked like data — when the answer was a first-grade word puzzle hiding in plain sight. An acrostic: a message formed by the first letters of each line. And "CHECKLSB" is not a cryptic phrase. It's an instruction. Check the LSB — the Least Significant Bit layer of the image. The poem was telling me exactly what to do next.

### Step 4: Installing `zsteg` — The Part That Tripped Me Up

LSB steganography in PNG images — my tool of choice is `zsteg`. So I typed `sudo apt install zsteg` and got nothing. Package not found. I tried `apt search zsteg`. Still nothing. Checked Snap. No luck there either.

Turns out zsteg is not in the standard APT repositories at all. It's a Ruby gem, and you need to install it through RubyGems after setting up the Ruby development environment and ImageMagick bindings. I didn't know this going in and lost probably 10 minutes trying different apt variants before looking it up.
    
    
    $ sudo apt update
    $ sudo apt install ruby ruby-dev imagemagick libmagickwand-dev
    $ sudo gem install zsteg

The `libmagickwand-dev` dependency is the one that catches people. The gem won't compile without it. Once that's in place, `gem install zsteg` works cleanly.

### Step 5: Running `zsteg` — The Payload Surfaces

With zsteg installed, I ran it on the image:
    
    
    $ zsteg red.png
    meta Poem           .. text: "Crimson heart, vibrant and bold,\nHearts flutter at your sight.\nEvenings glow softly red,\nCherries burst with sweet life.\nKisses linger with your warmth.\nLove deep as merlot.\nScarlet leaves falling softly,\nBold in every stroke."
    b1,rgba,lsb,xy      .. text: "cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ=="
    b1,rgba,msb,xy      .. file: OpenPGP Public Key

There it is. The `b1,rgba,lsb,xy` channel contains a Base64-encoded string — twice concatenated, but that's just the decoder reading past the end of the actual data. The real payload is the first copy.

* * *

## Capture the Flag

The Base64 string `cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==` needed one more step. I wrote a quick Python snippet:
    
    
    import base64
    cipher = "cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ=="
    plain = base64.b64decode(cipher).decode()
    print(plain)
    
    
    $ python3 a.py
    picoCTF{r3d_1s_th3_ult1m4t3_cur3_f0r_54dn355_}

That moment when the flag prints cleanly to stdout — it never gets old. After the frustration of the zsteg installation, seeing a clean `picoCTF{...}` string felt disproportionately satisfying.

**Flag:** `picoCTF{r3d_1s_th3_ult1m4t3_cur3_f0r_54dn355_}`

* * *

## Full Trial Process Table

Step| Action| Command| Result| Why it failed or succeeded  
---|---|---|---|---  
1| Identify file type| `file red.png`| Valid PNG, 128×128, RGBA| Succeeded — confirmed the file is a genuine PNG, not disguised as something else  
2| Read metadata| `exiftool red.png`| Found embedded Poem field| Succeeded — unusual Poem field stood out immediately as non-standard metadata  
3| Decode acrostic| Manual reading of first letters| CHECKLSB| Succeeded — once you look for it, the pattern is obvious; easy to miss on first glance  
4a| Install zsteg via apt| `sudo apt install zsteg`| Package not found| Failed — zsteg is not in the standard APT repositories; requires RubyGems  
4b| Install Ruby dependencies| `sudo apt install ruby ruby-dev imagemagick libmagickwand-dev`| All packages installed| Succeeded — `libmagickwand-dev` is required for the gem to compile  
4c| Install zsteg via gem| `sudo gem install zsteg`| zsteg installed| Succeeded — once dependencies are in place, gem install works cleanly  
5| Run LSB steganography scan| `zsteg red.png`| Base64 string in b1,rgba,lsb,xy| Succeeded — CHECKLSB hint pointed directly to the right channel  
6| Decode Base64| `python3 a.py`| `picoCTF{r3d_1s_th3_ult1m4t3_cur3_f0r_54dn355_}`| Succeeded — standard Base64 decoding, no further obfuscation  
  
* * *

## Command Explanations

### exiftool

`exiftool` reads metadata embedded in image files — things like camera settings, GPS data, copyright information, and (in this case) custom fields that challenge authors have planted. It reads dozens of metadata formats across hundreds of file types. Running it with no flags on a file gives you a full dump of everything embedded. It's usually one of the first things I run on a forensics challenge image because metadata is cheap to hide things in and often overlooked.

### zsteg

`zsteg` is a Ruby-based tool specifically designed to detect hidden data in PNG and BMP files using steganographic techniques. It checks multiple bit-plane combinations (b1 through b8), color channel orderings (rgb, rgba, bgr, etc.), bit orders (lsb, msb), and scan directions (xy, yx). The output line `b1,rgba,lsb,xy` means: bit depth 1, RGBA channels in that order, least significant bit first, scanning left-to-right then top-to-bottom. That specific combination is one of the most common LSB hiding methods, which is why it showed up first.

### base64 (Python)

Base64 is an encoding scheme — not encryption. It converts binary data into ASCII text using a 64-character alphabet (A-Z, a-z, 0-9, +, /). The `=` padding at the end is a giveaway. Python's `base64.b64decode()` reverses this. Because it's encoding rather than encryption, no key is needed — if you recognize the format, you can decode it immediately. In CTF challenges, Base64 often appears as a final layer after the actual hiding mechanism has been defeated.

* * *

## Beginner Tips

  * **Always run`file` first.** A file extension can lie. The `file` command reads magic bytes at the start of the file — that's the truth.
  * **`exiftool` before anything else on image challenges.** Custom metadata fields like "Poem" won't show up in normal image viewers. You need to ask for them explicitly.
  * **Read slowly.** The acrostic in this challenge is easy to miss if you skim. Treat every piece of embedded text as potentially meaningful.
  * **zsteg is not in apt.** Save yourself 10 minutes of confusion: it requires `ruby-dev` and `libmagickwand-dev` before `gem install zsteg` will work.
  * **Double-check Base64 strings before decoding.** zsteg sometimes reads past the end of the actual data and duplicates it. Compare the two halves — if they're identical, use just one copy.
  * **Keep a tool checklist.** For PNG forensics: `file` → `exiftool` → `strings` → `binwalk` → `zsteg` → `pngcheck`. Working through a checklist beats staring blankly at a file.



* * *

## What You Learned / Takeaways

This challenge is a clean three-layer puzzle: metadata hiding (the poem in exiftool), linguistic encoding (the acrostic spelling CHECKLSB), and steganographic hiding (LSB data in the RGBA channel). Each layer points to the next. It's a well-designed beginner challenge because no layer requires specialized knowledge — just methodical thinking and knowing which tools to reach for.

The zsteg installation issue is worth dwelling on. "Not in apt" is a common pattern for niche security tools. When a standard package manager comes up empty, the usual next steps are: check if it's a Python package (`pip install`), a Ruby gem (`gem install`), a Go tool (`go install`), or a manual build from GitHub. Knowing the tool's ecosystem tells you where to look.

On the LSB technique itself: each color channel in an RGBA pixel is stored as an 8-bit number. The least significant bit — the rightmost 1 in the binary representation — contributes almost nothing to the visible color. A red value of 254 (11111110) and 255 (11111111) are visually indistinguishable. Flip those last bits across every pixel in a 128×128 RGBA image and you have 128 × 128 × 4 = 65,536 bits of hidden storage — enough to hold meaningful text. The image looks completely normal. No tools would flag it in transit. That's the point.

This is not just a CTF puzzle technique. I've read incident reports where malware communicated with command-and-control servers by exfiltrating data hidden in PNG images uploaded to public file-sharing sites — traffic that looked like ordinary image uploads to any network monitor not specifically checking for steganographic content. Media companies use the same underlying math for digital watermarking: embedding traceable identifiers in image files to identify the source of a leak. Both sides of the security industry use this. Knowing how to detect it matters.

**If I solved this again:** I'd go straight from exiftool to zsteg. The CHECKLSB acrostic is a strong enough hint that I wouldn't spend time on binwalk or strings first. I'd also pre-verify the zsteg installation before the challenge clock starts — tool installation under time pressure is avoidable friction.

* * *

## Further Reading

This problem is part of the picoCTF Forensics series. You can see the other problems [here](<https://alsavaudomila.com/category/picoctf-forensics-easy/>).

For more Forensics Tools, check out [CTF Forensics Tools: The Ultimate Guide for Beginners](<https://alsavaudomila.com/ctf-forensics-tools/>).

Here are related articles from [alsavaudomila.com](<https://alsavaudomila.com/>) that complement this challenge:

  * [steghide in CTF: How to Hide and Extract Data from Files](<https://alsavaudomila.com/steghide-in-ctf-how-to-hide-and-extract-data-from-files/>) — another steganography tool commonly used in CTF challenges
  * [pngcheck in CTF: How to Analyze and Repair PNG Files](<https://alsavaudomila.com/pngcheck-in-ctf-how-to-analyze-and-repair-png-files/>) — when PNG structure itself is the puzzle
