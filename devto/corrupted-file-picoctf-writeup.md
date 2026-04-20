---
title: "Corrupted File picoCTF Writeup"
published: true
description: "
Magic bytes corruption, hex editing, and a JPEG hiding in plain sight — this is my full walkthrough of the picoCTF 2019 “Corrupted File” forensics challen"
tags: ctf, picoctf, forensics, security
canonical_url: https://alsavaudomila.com/corrupted-file-picoctf-writeup/
---

Magic bytes corruption, hex editing, and a JPEG hiding in plain sight — this is my full walkthrough of the **picoCTF 2019 "Corrupted File"** forensics challenge. The `file` command returned just "data." No extension, no hint about the format. I spent roughly 25 minutes confused, trying the wrong tools, before two bytes of hexdump finally made everything click. Here's the complete process, including every dead end.

## Challenge Overview: picoCTF 2019 "Corrupted File"

This challenge is from **picoCTF 2019** , Forensics category, rated Easy. The setup: you receive a single file with no extension. The challenge description says it's "corrupted." Your job is to figure out what it actually is and recover the hidden flag inside it.

There are no other hints. No suggested tools. Just a broken file and your terminal.

Field| Details  
---|---  
CTF Name| picoCTF 2019  
Challenge Name| Corrupted File  
Category| Forensics  
Difficulty| Easy  
Flag| picoCTF{r3st0r1ng_th3_by73s_b67c1558}  
  
## First Look: The File Command Returns Nothing Useful

My first instinct with any unknown file is `file`. It checks the actual byte content — not the extension — and matches known magic byte patterns. I ran it expecting at least a partial identification:
    
    
    $ file corrupted_file
    corrupted_file: data

"data." That's the `file` command's way of saying it matched absolutely nothing in its signature database. Not truncated, not partially identified — just completely unknown. My first thought was that maybe the file was totally destroyed and nothing was recoverable. I sat with that for a moment.

Then I looked at the challenge name again: _Corrupted File_. Of course it was corrupted. That was the point. Something had been deliberately broken, and that something was almost certainly the header — the magic bytes that tell every tool on your system what type of file it's looking at.

### Running strings — and finding more than I expected

Before touching any hex editor, I ran `strings` to see if any readable content survived the corruption:
    
    
    $ strings corrupted_file | head -20
    JFIF
    Exif
    ...
    picoCTF{r3st0r1ng_th3_by73s_b67c1558}

Two things jumped out immediately. First: **JFIF**. That's the JPEG File Interchange Format marker — it's specific to JPEG files and always appears just after the SOI header. Second, the flag was right there, readable as plain text near the end of the output. I could have submitted it and moved on.

But I didn't, because I wanted to understand what was actually broken. Submitting the flag without understanding the fix felt like cheating myself out of the lesson.

## Why JPEG? How I Ruled Out Every Other Format

When you see an unidentified file, the first question is: what format is it supposed to be? I didn't guess JPEG randomly — I reasoned through it by eliminating alternatives.

  * **JFIF = JPEG, full stop.** The string "JFIF" doesn't appear in PNG, GIF, PDF, or ZIP files. It's specific to JPEG. Once I saw it in strings output, there was no ambiguity.
  * **PNG ruled out immediately.** PNG files contain the ASCII string "PNG" in bytes 1–3 of their magic number (`89 50 4E 47`). Even if the first byte were corrupted, "PNG" would still show up in `strings`. It didn't.
  * **GIF ruled out.** GIF headers start with ASCII "GIF87a" or "GIF89a." Completely readable in `strings`. Absent here.
  * **PDF ruled out.** PDFs begin with `%PDF`, also readable ASCII. Not present.
  * **ZIP ruled out.** ZIP files start with "PK" in ASCII. Not present either.



The only format consistent with seeing "JFIF" in the strings output was JPEG. The body of the file was fine — it was just the two-byte SOI marker at the very beginning that had been tampered with.

## Rabbit Hole: 20 Minutes of Wrong Approaches

I didn't go straight to hexdump. I'm being honest here: I fumbled around for a while trying things that seemed reasonable but weren't actually going to fix anything.

### Attempt 1: Renaming and opening directly (~5 minutes)

My immediate reaction was to rename it to `corrupted_file.jpg` and try to open it in an image viewer. I figured maybe it just needed the extension. The image viewer opened, showed a loading spinner for a fraction of a second, then gave me an error: "Invalid or unsupported image format." Of course. File extensions are just labels — they don't affect how the actual bytes are interpreted. A JPEG viewer checks the magic bytes first, and `FF D8` was not there.

### Attempt 2: exiftool (~5 minutes)
    
    
    $ exiftool corrupted_file
    ExifTool Version Number         : 12.76
    File Name                       : corrupted_file
    File Type                       : JPEG
    File Type Extension             : jpg
    MIME Type                       : image/jpeg
    Image Width                     : 400
    Image Height                    : 300
    Color Components                : 3
    ...

This was surprising. Exiftool identified it as JPEG — dimensions and everything — even without the correct magic bytes. That's because exiftool doesn't rely solely on the first two bytes; it looks for JFIF/Exif markers deeper in the file structure. Useful confirmation that the file was JPEG, but it didn't repair anything. I still couldn't view it.

### Attempt 3: binwalk (~10 minutes)
    
    
    $ binwalk corrupted_file
    
    DECIMAL       HEXADECIMAL     DESCRIPTION
    --------------------------------------------------------------------------------
    0             0x0             Unknown data

Binwalk relies on magic byte signatures at known offsets. With the first two bytes wrong, it couldn't match anything. I tried `binwalk -e` (extract), `binwalk --dd`, different flags — nothing. Eventually I accepted that no automated tool was going to save me here. I needed to look at the raw bytes.

## hexdump: Finding the Two Wrong Bytes

I finally did what I should have done ten minutes earlier:
    
    
    $ hexdump -C corrupted_file | head -3
    00000000  5c 78 ff e0 00 10 4a 46  49 46 00 01 01 00 00 01  |\.....JFIF......|
    00000010  00 01 00 00 ff db 00 43  00 08 06 06 07 06 05 08  |.......C........|
    00000020  07 07 07 09 09 08 0a 0c  14 0d 0c 0b 0b 0c 19 12  |................|

The first two bytes: `5C 78`. Everything from byte 2 onward looked like legitimate JPEG data — `FF E0` is the APP0 marker, and there's "JFIF" right in the ASCII column at bytes 6–9.

So what is `5C 78`? In ASCII, `5C` is a backslash (`\`) and `78` is the letter `x`. Together: `\x`. That's the hex escape prefix used in Python, C, and most scripting languages. When you write `\xff\xd8` in code, the `\x` parts are supposed to be interpreted by the language and produce the binary bytes `FF D8`. But here, the `\x` prefix had been written out as literal ASCII characters instead of being processed. So instead of the binary JPEG SOI marker, the file had the text "`\x`" followed by the rest of the original sequence.

A correct JPEG SOI (Start of Image) header looks like this:
    
    
    FF D8 FF E0 00 10 4A 46 49 46 ...
    ^^^^^
    SOI marker — the two bytes that identify this as JPEG

The fix: replace bytes 0 and 1 with `FF D8`.

## The "Aha" Moment: \x as a Fingerprint of the Corruption

When I saw `5C 78` in the hexdump, I stared at it for a moment. Then it hit me. Backslash-x. _That 's escape notation._ The person who wrote the challenge (or the script that generated it) had likely worked with the bytes as a Python string like `b'\xff\xd8\xff\xe0...'`, and something went wrong in the encoding — the escape sequences got written as ASCII text rather than as binary.

It's such a specific kind of corruption that it immediately told me this was intentional challenge design. It's not random bit-flipping or file truncation — it's a precise two-byte substitution that takes the exact magic number and replaces it with its own escape-notation representation. Elegant, in a devious sort of way.

Once I understood what had happened, the fix was obvious and I felt slightly embarrassed it took me 20 minutes to get there.

## The Fix: Surgical Repair with hexedit

First, copy the file. This is non-negotiable — never edit the original:
    
    
    $ cp corrupted_file fixed_file
    $ hexedit fixed_file

In hexedit, the display shows hex values on the left and their ASCII equivalents on the right. The cursor starts at position 0x00. I typed `FF` — the display updated immediately, showing the changed byte. Then `D8` for the second byte. Then `Ctrl+X` to save and exit.

The hexdump before and after, for direct comparison:

### Before — corrupted header:
    
    
    00000000  5c 78 ff e0 00 10 4a 46  49 46 00 01 01 00 00 01  |\.....JFIF......|

### After — repaired header:
    
    
    00000000  ff d8 ff e0 00 10 4a 46  49 46 00 01 01 00 00 01  |......JFIF......|

Two bytes changed. The rest of the file: completely untouched. The JFIF marker at bytes 6–9 had been there all along, waiting for the SOI marker to precede it correctly.

Confirmation with `file`:
    
    
    $ file fixed_file
    fixed_file: JPEG image data, JFIF standard 1.01, resolution (DPI), density 1x1, segment length 16, baseline, precision 8, 400x300, components 3

From "data" to a fully described JPEG image with dimensions, resolution, and encoding details. That output felt disproportionately satisfying for what was, mechanically, a two-byte fix.

## The Flag

Opening the repaired image revealed the flag printed on it:
    
    
    picoCTF{r3st0r1ng_th3_by73s_b67c1558}

## Full Trial Process: Every Step, Honest Results

Step| Command| Result| Why it failed / What I learned  
---|---|---|---  
1| `file corrupted_file`| "data" — unidentified| Magic bytes at offset 0 don't match any known signature  
2| Renamed to .jpg, opened in image viewer| Error: invalid image format| File extensions are labels only; viewers check actual header bytes  
3| `strings corrupted_file | head -20`| Found "JFIF" and the flag as plaintext| Confirmed JPEG format; flag technically visible here already  
4| `exiftool corrupted_file`| Identified as JPEG (400×300)| Exiftool uses deeper format parsing; useful confirmation  
5| `binwalk corrupted_file` (multiple flags)| "Unknown data" — nothing extracted| Binwalk also uses magic bytes at offset 0; dead end  
6| `hexdump -C corrupted_file | head -3`| Bytes 0–1 are `5C 78` (ASCII "`\x`")| Identified the corruption: escape notation written as literal ASCII  
7| `cp corrupted_file fixed_file && hexedit fixed_file`| Changed `5C 78` → `FF D8` at offset 0| Two-byte repair of the JPEG SOI marker  
8| `file fixed_file`| Valid JPEG, 400×300, 3 components| Repair successful; format fully identified  
9| Opened fixed_file in image viewer| Flag visible in image| Challenge complete  
  
## Real-World Relevance: Magic Byte Attacks Beyond CTF

Magic byte manipulation isn't just a CTF trick. It's a real attack technique with documented abuse cases — and understanding how it works makes you a better analyst on both sides of a security investigation.

### File upload bypass attacks

Many web applications validate uploaded files by checking only the magic bytes — the assumption being that if the first few bytes say "JPEG," the file is a JPEG. Attackers exploit this by prepending valid JPEG magic bytes to a PHP shell, a Python script, or any other executable payload. The server's validator sees `FF D8` and passes the check. The file is stored. A direct HTTP request to the upload URL executes the code.

This attack class has appeared in CVEs against WordPress plugins, PHP image processing libraries (particularly when `getimagesize()` is misused as a security check), and custom file upload handlers that never perform server-side content inspection beyond the header.

### Email gateway evasion

Some email security gateways block attachments based on their detected file type. Malware authors have swapped the magic bytes of Windows executables (which start with `4D 5A`, "MZ") to make them resemble PDFs or Office documents. The gateway scans the header, sees a "document," and passes the attachment. The recipient's system — which may do additional verification — then handles it correctly as an executable.

### Polyglot files

A polyglot file is simultaneously valid in two different formats. The most well-known CTF example is a file that is both a valid JPEG (because it starts with `FF D8`) and a valid ZIP archive (because the ZIP end-of-central-directory record appears at the end and doesn't interfere with JPEG parsing). Different tools interpret the same file completely differently. This has been used in real attacks to bypass content filters that only check one format's markers.

### Anti-forensic header corruption

Malware samples sometimes deliberately corrupt their own PE headers or remove their magic bytes to confuse automated sandboxes. If a sandbox can't identify the file type, it may skip format-specific behavioral analysis. This buys the malware time in environments where human review only happens for samples that automated systems flag clearly.

### MIME sniffing and the nosniff header

Browsers can disagree with servers about file types. A server might declare `Content-Type: image/jpeg`, but if the bytes look like HTML, some browsers will sniff the content and render it as HTML — potentially executing embedded JavaScript. This is why `X-Content-Type-Options: nosniff` is a security best practice in HTTP headers. Understanding magic bytes is fundamental to understanding why that header exists.

## Magic Bytes Cheat Sheet for CTF Forensics

Having these memorized saves significant time in forensics challenges:

File Type| Magic Bytes (Hex)| ASCII / Notes  
---|---|---  
JPEG| `FF D8 FF`| Non-printable; followed by APP0 `FF E0` or APP1 `FF E1`  
PNG| `89 50 4E 47 0D 0A 1A 0A`| `.PNG....` — "PNG" is readable in strings  
GIF| `47 49 46 38`| `GIF8` — followed by "7a" or "9a"  
PDF| `25 50 44 46`| `%PDF` — fully readable ASCII  
ZIP / JAR / DOCX| `50 4B 03 04`| `PK..` — many formats are ZIP-based  
ELF (Linux binary)| `7F 45 4C 46`| `.ELF`  
Windows PE (.exe/.dll)| `4D 5A`| `MZ` — from Mark Zbikowski, original DOS designer  
SQLite database| `53 51 4C 69 74 65 20 66`| `SQLite f` — starts "SQLite format 3"  
  
## Beginner Tips for Magic Byte Challenges

  * **Start with`file`, always.** If it returns "data," the header is broken. That's your diagnostic.
  * **Run`strings` before reaching for a hex editor.** Format-specific strings like "JFIF," "PNG," "%PDF," or "GIF8" appear in even heavily corrupted files and tell you the intended type in seconds.
  * **Check only the first 8–16 bytes.** Magic bytes live at the very beginning. `hexdump -C file | head -2` is almost always sufficient to find header corruption.
  * **Copy before editing.** Always: `cp original working_copy`. Run all your edits on the copy. You may need the original as a reference.
  * **hexedit navigation:** Arrow keys to move, just type hex digits to overwrite. `Ctrl+X` saves and exits. `Ctrl+C` cancels without saving.
  * **Python one-liner if hexedit feels uncomfortable:**


    
    
    data = open('corrupted_file', 'rb').read()
    fixed = b'\xff\xd8' + data[2:]
    open('fixed_file', 'wb').write(fixed)

This replaces the first two bytes with the correct JPEG SOI marker and writes the result to a new file. Three lines. No hex editor needed.

## What I'd Do Differently Next Time

The biggest mistake I made was spending time on tools before looking at the raw bytes. The pattern for magic byte corruption challenges is straightforward once you've done it once:

  1. Run `file` — does it identify the format? If it returns "data," the header is wrong.
  2. Run `strings` — what format-specific strings appear? This tells you the intended file type without opening a hex editor.
  3. Run `hexdump -C file | head -3` — find the exact bytes at the start and compare to the known magic number for that format.
  4. Make a copy, open in hexedit (or write a Python one-liner), and fix the bytes.
  5. Verify with `file` again.



That's a five-step process I could now execute in under two minutes. At the time it took me about 25. The difference is just pattern recognition — and that comes from working through enough of these challenges to stop second-guessing the most obvious approach.

I'd also spend five minutes before any forensics CTF round memorizing the common magic bytes: JPEG, PNG, GIF, ZIP, PDF. Having that table in your head means you recognize a corrupted header instantly from the hexdump, rather than having to stop and look things up mid-solve.

## Further Reading

This problem is part of the picoCTF series. You can see the other problems [here](<https://alsavaudomila.com/category/picoctf-forensics-easy/>).

For more Forensics Tools, check out [CTF Forensics Tools: The Ultimate Guide for Beginners](<https://alsavaudomila.com/ctf-forensics-tools/>).

Here are related articles from [alsavaudomila.com](<https://alsavaudomila.com/>) that complement this challenge:

  * [RED picoCTF Writeup](<https://alsavaudomila.com/red-picoctf-writeup/>) — another forensics challenge involving PNG file analysis with zsteg and exiftool, where file format understanding is equally critical
  * [Scan Surprise picoCTF Writeup](<https://alsavaudomila.com/scan-surprise-picoctf-writeup/>) — working with PNG format and QR code extraction in a CTF context
