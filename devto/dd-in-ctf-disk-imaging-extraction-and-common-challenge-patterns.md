---
title: "dd in CTF: Disk Imaging, Extraction, and Common Challenge Patterns"
published: true
description: "
The Day a Disk Image Broke My Brain (and dd Fixed It)
If you’re doing CTF disk imaging or forensics challenges and keep hitting a wall even after running "
tags: ctf, security, linux, forensics
canonical_url: https://alsavaudomila.com/dd-in-ctf-disk-imaging-extraction-and-common-challenge-patterns/
---

## The Day a Disk Image Broke My Brain (and dd Fixed It)

If you're doing CTF disk imaging or forensics challenges and keep hitting a wall even after running `binwalk`, there's a good chance `dd` is the missing piece. I know because I spent two hours in exactly that situation during a picoCTF forensics challenge — one of those deceptively named ones like "Disk, disk, sleuth!" — before I figured out why everything I tried wasn't working.

The file was a 500MB disk image. I had run `file`. I had run `strings | grep -i flag`. I had even tried opening it in a hex editor and scrolling manually like some kind of masochist. Nothing. I knew the flag was in there. I just couldn't reach it.

That's when I properly met `dd`.

I say "properly" because I'd used `dd` before — the classic _sudo dd if=/dev/sda of=backup.img_ muscle-memory kind. But using it as a precision extraction tool in CTF, targeting specific byte offsets inside a forensics image? That was new to me. By the end of that session, `dd` had become one of the first things I reach for in any disk imaging challenge. This article is about what I learned — not just the syntax, but the _thinking_ that makes dd genuinely useful.

## dd Syntax: The Four Parameters That Actually Matter

The full `dd` manpage is intimidating. In CTF forensics, you really only need four parameters to do 90% of the work:
    
    
    dd if=<input_file> of=<output_file> bs=<block_size> skip=<n_blocks> count=<n_blocks>

  * **if=** — your input (the disk image or binary you're carving from)
  * **of=** — your output (always write to a _new_ file — never overwrite the original)
  * **bs=** — block size. Use `bs=1` for byte-accurate extraction, `bs=512` or `bs=4096` for disk sector operations
  * **skip=** — skip this many blocks from the start before reading
  * **count=** — read this many blocks total



The mental model that changed everything for me: **when`bs=1`, skip and count are in bytes**. So `skip=4096 count=200` means "start at byte 4096, read exactly 200 bytes." That's the precision you need when binwalk hands you an offset and you need to extract exactly what lives there.

### The conv=notrunc Option (Learn This Before You Need It)

One more flag worth memorizing: `conv=notrunc`. This tells dd to write without truncating the output file. It's critical when you're _patching_ — replacing corrupted bytes in a file header without changing the rest of the file. I learned this the hard way after dd helpfully truncated a 50MB disk image down to 8 bytes because I forgot it. The error isn't obvious either; dd just silently writes 8 bytes and exits with code 0. Always check your output size with `ls -lh` after patching.

## Rabbit Hole: The 30 Minutes I Wasted Before Using dd

Here's what my actual workflow looked like before I understood dd's role in forensics — I want to be honest about this because I suspect it matches what a lot of beginners try:
    
    
    $ file mystery.img
    mystery.img: DOS/MBR boot record
    
    $ strings mystery.img | grep -i "flag\|ctf\|pico"
    (no output)
    
    $ xxd mystery.img | head -50
    00000000: eb52 9045 5854 3220 2020 2000 0201 2000  .R.EXT2    .. .
    00000010: 0000 0000 0000 29f8 b703 004e 4f20 4e41  ......)....NO NA
    # I scrolled through this for 20+ minutes
    
    $ binwalk mystery.img
    
    DECIMAL       HEXADECIMAL     DESCRIPTION
    --------------------------------------------------------------------------------
    0             0x0             DOS/MBR boot record
    1048576       0x100000        PNG image, 640 x 480, 8-bit/color RGB
    1183744       0x121000        End of Zip archive
    
    # I stared at this output for five minutes not knowing what to do with it

That last step was the breakthrough I didn't act on. Binwalk had _told me exactly where the embedded file was_ , but I didn't know what to do with those decimal numbers. I opened the image in Autopsy — eight-minute startup, found nothing relevant because Autopsy works at the filesystem layer, not the raw byte layer. I ran `foremost`, which chewed through the file for another eight minutes and returned a handful of false positives.

The fix was almost embarrassingly simple — and when I finally saw it, I actually said "oh, come on" out loud. Binwalk gives you byte offsets. dd extracts from byte offsets. They're a matched pair: binwalk is the scanner, dd is the scalpel. I ran one dd command, the PNG extracted cleanly, opened it, and there was the flag staring back at me. All that time I'd had the answer sitting right there in binwalk's output. I just hadn't known what to do with those numbers.

## Six CTF Patterns Where dd Is the Right Call

Over multiple CTF sessions I've noticed that dd challenges cluster into a handful of recognizable types. Here's how I approach each one, including the mistakes I made the first time:

### Pattern 1: Hidden File at a Known Offset (The Classic)

Binwalk finds an embedded PNG, ZIP, or ELF inside a larger binary. You grab the offset and extract with dd. Simple in theory — but the count calculation trips beginners up every time.
    
    
    $ binwalk mystery.img
    
    DECIMAL       HEXADECIMAL     DESCRIPTION
    --------------------------------------------------------------------------------
    0             0x0             DOS MBR boot record
    1048576       0x100000        PNG image, 640 x 480, 8-bit/color RGB
    1183744       0x121000        End of Zip archive
    
    # count = end_offset - start_offset = 1183744 - 1048576 = 135168
    $ dd if=mystery.img of=extracted.png bs=1 skip=1048576 count=135168
    135168+0 records in
    135168+0 records out
    135168 bytes (135 kB, 132 KiB) copied, 0.412 s, 328 kB/s
    
    $ file extracted.png
    extracted.png: PNG image data, 640 x 480, 8-bit/color RGB, non-interlaced

If binwalk doesn't give you a clear end offset, just omit count entirely. dd will read from your skip point to end of file, and you can open the result to find where the embedded content actually ends. Messier, but it works.

### Pattern 2: Corrupted File Header Repair

The challenge gives you a "broken" PNG that image viewers refuse to open. A quick peek in a hex editor shows the magic bytes at offset 0 are wrong — maybe `00 00 00 00 4E 47 0D 0A` instead of the correct PNG signature `89 50 4E 47 0D 0A 1A 0A`.
    
    
    # Create a file containing the correct PNG magic bytes
    $ printf '\x89\x50\x4e\x47\x0d\x0a\x1a\x0a' > correct_header.bin
    
    # Patch the header — conv=notrunc is mandatory
    $ dd if=correct_header.bin of=broken.png bs=1 count=8 conv=notrunc
    8+0 records in
    8+0 records out
    8 bytes copied, 0.000074 s, 108 kB/s
    
    $ file broken.png
    broken.png: PNG image data, 1920 x 1080, 8-bit/color RGBA, non-interlaced

Without `conv=notrunc`, dd writes 8 bytes and then truncates the output file to 8 bytes. You've just destroyed your working copy. Keep the original untouched and always write to a new filename first time around.

### Pattern 3: Flag Hidden After a Marker String

Sometimes the flag isn't in a proper file format — it's raw bytes appended after a known boundary. The trick is using grep with byte-offset flags to locate the marker, then dd to extract what follows.
    
    
    # -b: print byte offset, -o: only matching text, -a: treat binary as text
    $ grep -boa "END_HEADER" data.bin
    204800:END_HEADER
    
    # Skip past the marker: offset 204800 + len("END_HEADER") = 204810
    $ dd if=data.bin of=after_marker.bin bs=1 skip=204810
    
    $ strings after_marker.bin | head -3
    picoCTF{h1dd3n_4ft3r_th3_m4rk3r_ab12cd}

The `grep -boa` pattern is genuinely useful for binary files — more reliable than hex searching manually, and it gives you machine-readable offsets you can feed directly to dd's skip parameter.

### Pattern 4: Partition Extraction from a Disk Image

This one has an extra calculation step. `fdisk` reports partition boundaries in sectors (512 bytes each), and you translate that directly to dd parameters by setting `bs=512`.
    
    
    $ fdisk -l disk.img
    Disk disk.img: 100 MiB, 104857600 bytes, 204800 sectors
    Units: sectors of 1 * 512 = 512 bytes
    
    Device      Boot  Start    End  Sectors  Size  Type
    disk.img1         2048   43007   40960   20M   Linux filesystem
    disk.img2        43008  204799  161792   79M   Linux filesystem
    
    # Extract partition 2: skip and count are in sectors because bs=512
    $ dd if=disk.img of=partition2.img bs=512 skip=43008 count=161792
    161792+0 records in
    161792+0 records out
    
    $ file partition2.img
    partition2.img: Linux rev 1.0 ext2 filesystem data

This is one of the few cases where I don't use `bs=1`. The sector-aligned arithmetic maps cleanly from fdisk output, and performance matters when you're carving 80MB from a large image.

### Pattern 5: Splitting a Binary into Multiple Embedded Objects

Binwalk identifies multiple embedded files at different offsets. You extract each one independently — this is where the count calculation becomes routine but error-prone if you rush it.
    
    
    $ binwalk multi.bin
    
    DECIMAL       HEXADECIMAL     DESCRIPTION
    280           0x118           JPEG image data
    4096          0x1000          Zip archive data
    8192          0x2000          ELF 64-bit LSB executable
    
    $ dd if=multi.bin of=image.jpg   bs=1 skip=280  count=3816
    $ dd if=multi.bin of=archive.zip bs=1 skip=4096 count=4096
    $ dd if=multi.bin of=binary.elf  bs=1 skip=8192

### Pattern 6: Truncated File Recovery

The challenge gives you a file that's cut off mid-stream. Sometimes the header is intact and you just need to extract the valid region. Other times the footer is missing and you can construct a minimal valid one. Either way, dd lets you work on byte-precise regions without touching the original. I had a picoCTF challenge where a ZIP was truncated — the central directory was missing, but the local file headers were intact. I extracted each file record individually with dd and manually reconstructed the archive. Tedious, but it worked.

## dd vs Other Tools: When to Switch

dd wins when you know exactly where to look. Here's how I make the call in practice:

Situation| My First Choice| Why Not dd?  
---|---|---  
Known byte offset from binwalk| **dd**|  —  
Need to find offsets first| binwalk → then dd| dd needs you to know where to look  
Carving many unknown files from disk| foremost or PhotoRec| dd requires manual calculation per file  
Repairing a corrupted header| **dd + conv=notrunc**|  —  
Full filesystem investigation| Autopsy / Sleuth Kit| dd doesn't parse filesystems  
Quick "just extract everything" pass| binwalk -e| Auto-extract is faster for first-look recon  
  
The trap I kept falling into early on: thinking binwalk -e and dd are alternatives. They're not. Use binwalk -e for quick recon; use dd when you need clean, specific extraction. The auto-extract output is often messy — wrong file lengths, nested archives that half-extracted, corrupted headers. When the flag isn't in binwalk's auto-extract output, that's your cue to switch to manual dd extraction with exact offsets.

## Full Trial Process Table

Step| Action| Command| Result| Why it failed / succeeded  
---|---|---|---|---  
1| File identification| file mystery.img| DOS/MBR boot record| Partial — needed deeper scan  
2| String search| strings mystery.img | grep flag| No output| Flag was in binary data, not ASCII plaintext  
3| Hex editor scroll| xxd mystery.img | head -200| Boot sector data| Wasted 20 minutes scrolling with no clear target  
4| Auto-extract attempt| binwalk -e mystery.img| Some files extracted, not the flag| Auto-extract missed the embedded PNG at correct offset  
5| Filesystem tool| Autopsy (GUI)| No flag found| Works at filesystem layer — flag was between partitions in raw space  
6| File carver| foremost -i mystery.img| 8 minutes, false positives only| foremost uses signatures without precise offsets — wrong tool for this job  
7| Manual binwalk scan| binwalk mystery.img| PNG at offset 1048576| This was the key I already had — just didn't act on it  
8| dd extraction| dd if=mystery.img of=out.png bs=1 skip=1048576 count=135168| Clean PNG extracted| Correct approach — should have done this at step 4  
9| View result| eog out.png| Flag visible in image| picoCTF{…} — done  
  
## Why Block-Level Thinking Matters Beyond CTF

dd exists because sometimes you need to work with raw data before any filesystem abstraction gets in the way. Digital forensic investigators use it to create bit-perfect disk images that preserve deleted files, slack space, and unallocated regions that a normal file copy would miss. Malware analysts use it to carve memory regions out of VM snapshots. Embedded systems engineers use it to flash firmware directly to block devices.

The CTF-relevant insight: challenge authors often construct files that aren't valid by any filesystem standard — they're carefully crafted byte sequences with hidden content between real structures. A tool that works at the filesystem layer will miss things that live in the raw bytes. dd doesn't care about filesystems. It reads bytes. That's exactly why it finds things other tools can't.

## How I'd Solve It Faster Next Time

If I'm dropped into a forensics challenge with an unknown binary or disk image today, here's my actual first-three-minutes workflow — hard-won from doing it the slow way first:
    
    
    # Step 1: What is this thing?
    file target.img
    
    # Step 2: What's embedded inside it?
    binwalk target.img
    
    # Step 3: If binwalk shows interesting offsets, act immediately
    dd if=target.img of=extracted bs=1 skip=<offset>
    
    # Step 4: What did we get?
    file extracted
    strings extracted | head -20
    exiftool extracted  # if it looks like an image

I no longer reach for `strings` on the original file first — it's too noisy on large binary images. Binwalk gives you a structured map, and dd lets you act on it precisely. That two-step combination cuts my time on this challenge class by at least half compared to my original "try everything" approach.

One lesson I want to emphasize because I learned it the hard way: **never use the input filename as your output filename**. I once ran `dd if=mystery.img of=mystery.img ...` by accident — autocomplete betrayed me — and overwrote the only copy of the challenge file. The challenge server was in maintenance at the time. That was a rough afternoon. Always write to a new filename. `out.bin`, `extracted.png`, anything that isn't the original.

## Further Reading

If you want to go deeper on CTF forensics tools overall, [CTF Forensics Tools: The Ultimate Guide for Beginners](<https://alsavaudomila.com/ctf-forensics-tools/>) covers the full toolkit — dd fits into a larger ecosystem alongside binwalk, foremost, Autopsy, and Sleuth Kit, and understanding when to reach for each one is half the battle in forensics challenges.

Here are related articles from [alsavaudomila.com](<https://alsavaudomila.com/>) that pair well with this topic:

Before you reach for dd, you need binwalk to tell you where to look — the article on [binwalk in CTF](<https://alsavaudomila.com/binwalk-in-ctf/>) explains how to read its scan output accurately, which offsets to trust versus ignore, and how the auto-extract mode differs from manual dd-based extraction.

The [file command](<https://alsavaudomila.com/file-command-ctf/>) walkthrough covers what happens before dd enters the picture: understanding how `file` fingerprints data (and how CTF challenge authors fool it) shapes which extraction approach you take from the start.

Once dd extracts a clean image and you need to investigate its filesystem, the [Sleuth Kit and Autopsy guide](<https://alsavaudomila.com/sleuth-kit-autopsy-ctf/>) covers how to mount and browse partition contents — the natural next step after the raw extraction that dd handles.
