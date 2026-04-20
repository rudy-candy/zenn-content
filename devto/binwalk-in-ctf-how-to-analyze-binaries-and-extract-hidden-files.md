---
title: "binwalk in CTF: Spot False Positives Fast"
published: true
description: "binwalk floods your terminal with DER certificate hits on a plain PNG. Here's how to separate real embedded files from noise and extract only what matters."
tags: ctf, security, linux, forensics
canonical_url: https://alsavaudomila.com/binwalk-in-ctf-how-to-analyze-binaries-and-extract-hidden-files/
---

**binwalk** is a binary analysis tool that scans files for embedded signatures — ZIPs inside PNGs, compressed firmware blobs, appended archives. In CTF forensics it's usually one of the first tools you reach for. But the real skill isn't running it; it's reading the output correctly. While working on the "Digging for Treasure" challenge at BurnerCTF 2025, I ran binwalk and watched more than ten detections scroll across the screen. My first thought was "this is a goldmine." Thirty minutes later, when I realized every single one of them was a false positive, I learned firsthand just how dangerous it is to blindly trust tool output.

Rather than listing binwalk commands one by one, this article focuses on the noise problem you actually face in CTF scenarios and the decision-making process for finding the signal that matters.

## Real Output from BurnerCTF 2025: Separating Noise from Signal

Here is the actual output from running binwalk on the "Digging for Treasure" challenge.
    
    
    $ binwalk treasure.png
    
    DECIMAL       HEXADECIMAL     DESCRIPTION
    ------------------------------------------------------------------------
    0             0x0             PNG image, 1536 x 1024, 8-bit/color RGB, non-interlaced
    3860          0xF14           Certificate in DER format (x509 v3), header length: 4, sequence length: 1573
    5440          0x1540          Certificate in DER format (x509 v3), header length: 4, sequence length: 1746
    7193          0x1C19          Certificate in DER format (x509 v3), header length: 4, sequence length: 1455
    8688          0x21F0          Object signature in DER format (PKCS header length: 4, sequence length: 5983
    8857          0x2299          Certificate in DER format (x509 v3), header length: 4, sequence length: 1573
    10634         0x298A          Certificate in DER format (x509 v3), header length: 4, sequence length: 1716
    12354         0x3042          Certificate in DER format (x509 v3), header length: 4, sequence length: 1421
    15075         0x3AE3          Zlib compressed data, default compression
    2706518       0x294C56        TIFF image data, big-endian, offset of first image directory: 8

At first glance, it looks like the PNG contains six DER certificates, zlib data, and an embedded TIFF. In reality, every detection from offset 3860 through 15075 was a false positive.

### Why So Many DER Certificates Get Detected

PNG files store image data compressed with zlib inside IDAT chunks. Within that compressed binary data, patterns can appear that happen to match the magic bytes of a DER certificate (sequences starting with `30 82`). Since binwalk determines file types through binary signature matching without considering context, it reports every byte sequence that looks certificate-like.

The two bytes of the DER sequence tag (`0x30`) and the length field (`0x82`) appear frequently in compressed data. This is the root cause of the false positive flood inside IDAT chunks.

### Criteria for Finding Real Signals

In the output above, the only detection actually worth investigating was the last line: the TIFF at offset `0x294C56` (2,706,518 bytes). Here is the reasoning behind that call.

First, check the context of each offset. Detections clustered near the beginning of the file (within the range where IDAT chunks exist) are likely false positives inside compressed data. On the other hand, an isolated detection of a different format near the end of the file — close to the total file size — is likely data appended after the file's end.

Next, compare against the file size. Run `ls -la treasure.png` to check the file size and see whether 2,706,518 bytes corresponds to near the end of the file. If data exists beyond the PNG's native IEND chunk, it was clearly appended.

Also check the consistency of detected formats. If six DER certificates are detected in a row and they are densely packed within small offset intervals, you should assume binwalk is scanning through a single compressed data block.

## The -e Option Trap: Avoiding Extraction Chaos

Using binwalk's extraction option `-e` dumps everything detected into an `_extracted/` folder. But running it against output riddled with false positives generates a flood of useless files and turns the folder into a mess.
    
    
    # A common mistake
    $ binwalk -e treasure.png
    
    # Result: _extracted/ gets filled with unwanted files
    $ ls _extracted/treasure.png.extracted/
    F14         F14.der
    1540        1540.der
    1C19        1C19.der
    21F0        21F0.der
    2299        2299.der
    298A        298A.der
    3042        3042.der
    3AE3        3AE3.zlib
    294C56.tiff
    294C56
    
    # Manual extraction from a specific offset using dd
    # Extract the TIFF from offset 0x294C56 (2706518 bytes)
    $ dd if=treasure.png bs=1 skip=2706518 of=extracted.tiff
    
    # Or use binwalk's --dd option to extract only a specific type
    $ binwalk --dd='tiff image:tiff' treasure.png

## Core binwalk Commands and When to Use Them
    
    
    # Scan a file for signatures
    $ binwalk file.bin
    
    # Detailed entropy analysis (-B)
    $ binwalk -B file.bin
    
    # Output an entropy graph (high-entropy regions = likely encrypted or compressed data)
    $ binwalk -E file.bin
    
    # Extract detected files (watch out for false positives)
    $ binwalk -e file.bin
    
    # Recursive extraction (re-scan extracted files)
    $ binwalk -Me file.bin
    
    # Search for a specific signature only
    $ binwalk -R "\x50\x4b\x03\x04" file.bin   # Search for ZIP signature
    
    # Scan a firmware image (common in CTF hardware/misc challenges)
    $ binwalk firmware.bin
    
    # Recursive extraction when the image contains filesystems like squashfs or cpio
    $ binwalk -Me firmware.bin

## How Magic Numbers and Binary Signatures Work

Understanding how binwalk identifies file types is key to spotting false positives. Most file formats have a fixed signature (magic number) at the start of the file. PNG, for example, uses `89 50 4E 47 0D 0A 1A 0A` (`\x89PNG\r\n\x1a\n`), and ZIP uses `50 4B 03 04` (`PK\x03\x04`).

binwalk matches signature patterns from its internal database against every offset in a file using a sliding window. This approach is powerful, but in regions with byte sequences that look random — like compressed or encrypted data — coincidental matches happen all the time. In the case of DER format, the sequence tag `0x30 0x82` appears frequently in binary data that has nothing to do with certificates, and binwalk reports every single one of those matches.

## When Not to Use binwalk: Choosing the Right Tool

If LSB steganography is suspected, use `zsteg`. The technique of hiding data in the least significant bits of PNG pixels cannot be detected by binwalk's signature scanning — binwalk found nothing on the "RED" picoCTF challenge where zsteg recovered the flag immediately. For metadata inspection, use `exiftool`; GPS coordinates, comment fields, and custom XMP tags are invisible to binwalk because they don't appear as binary signatures. To diagnose file corruption, reach for `pngcheck` or the `file` command first — running binwalk on a deliberately corrupted PNG often produces misleading output.

binwalk's full signature database and source are maintained at the [ReFirmLabs/binwalk GitHub repository](<https://github.com/ReFirmLabs/binwalk>). The `src/binwalk/magic/` directory lists every pattern binwalk scans for, which is useful when you need to understand why a specific false positive is being triggered.

## binwalk vs foremost vs strings: Comparison Table

Situation| Recommended Tool| Reason  
---|---|---  
Suspected embedded file in a different format| binwalk -e| Signature scanning with automatic extraction  
Appended data at the end of a file| binwalk (check offset) + dd| Identify the exact extraction point, then extract manually  
Recovering files from a disk image| foremost| File carving that ignores filesystem structure  
Searching for printable strings in a binary| strings| Fast search for flag strings or config values  
Analyzing filesystem structure in firmware| binwalk -Me| Recursive extraction unpacks squashfs/cramfs  
LSB steganography suspected| zsteg| binwalk does not detect bit-level manipulation of pixel data  
Flag hidden in EXIF metadata| exiftool| Metadata fields do not appear as binary signatures  
  
## Practical binwalk Workflow for CTF
    
    
    # Step 1: Start with the file command to get basic info
    $ file challenge.png
    
    # Step 2: Check the file size (you'll compare this against offsets later)
    $ ls -la challenge.png
    
    # Step 3: Scan with binwalk
    $ binwalk challenge.png
    
    # Step 4: Interpret the output
    # - Focus on detections at offsets close to the file size
    # - Be skeptical of DER/certificate detections clustered near the beginning
    # - Prioritize isolated detections of a different format at a unique offset
    
    # Step 5: Manually extract only from promising offsets
    $ dd if=challenge.png bs=1 skip=2706518 of=candidate.tiff
    
    # Step 6: Use strings to search for text if needed
    $ strings candidate.tiff | grep -i "ctf\|flag"

## Tips for Reducing binwalk False Positives

Using entropy analysis (the `-E` option) lets you visually map high-entropy regions (compressed or encrypted data) within a file. Combining it with the `strings` command is also effective — if you spot file header strings like "JFIF", "Exif", or "PK", use those offsets as a starting point.
    
    
    # Inspect bytes around offset 0x294C56
    $ xxd challenge.png | grep -A 3 "00294c"
    
    # Or use dd to check the bytes around that area
    $ dd if=challenge.png bs=1 skip=2706510 count=32 | xxd

## Further Reading

For more CTF forensics tools and decision guides, see the [CTF Forensics Tools: The Complete Guide](<https://alsavaudomila.com/ctf-forensics-tools-the-complete-guide-for-beginners/>).
