---
title: "DISKO 1 picoCTF Writeup"
published: true
description: "picoCTF DISKO 1 — why mounting, binwalk, and Sleuth Kit all come up empty, and how strings reads raw FAT16 sectors to find the flag in two seconds."
tags: ctf, picoctf, forensics, linux
canonical_url: https://alsavaudomila.com/disko-1-picoctf-writeup/
---

I spent over an hour doing everything wrong on **picoCTF DISKO 1** before the flag basically yelled at me. The challenge gives you a compressed FAT32 disk image and asks you to find a hidden flag. My instinct was to mount it, analyze partitions, and dig through the filesystem like a proper forensics investigator. The flag had other ideas. This writeup is mostly about that wasted hour — because the lesson here isn't the two-line solution, it's understanding why every other approach fails first.

---

## Challenge Overview

**CTF**: picoCTF
**Challenge name**: DISKO 1
**Category**: Forensics
**Difficulty**: Easy
**Given file**: `disko-1.dd.gz`

The challenge description is sparse — something about a disk image with a hidden flag. No hints about what filesystem, what tool, or where in the image the flag lives. Just a `.dd.gz` file and go.

I ran `file` on it first:

```
$ file disko-1.dd.gz
disko-1.dd.gz: gzip compressed data, was "disko-1.dd", last modified: ..., from Unix
```

Straightforward — gzip-compressed disk image. I gunzipped it and checked what we had:

```
$ gunzip disko-1.dd.gz
$ file disko-1.dd
disko-1.dd: DOS/MBR boot record, code offset 0x58+2, OEM-ID "mkfs.fat", sectors/cluster 4,
root entries 512, sectors 20480 (volumes <=32 MB), Media descriptor 0xf8,
sectors/FAT 20, sectors/track 32, heads 64, serial number 0x672d4c8c,
unlabeled, FAT (16 bit)
```

FAT16 disk image. From here I made what felt like a completely reasonable decision — and then kept making bad decisions for the next hour.

## The Rabbit Hole: An Hour of Doing It the Hard Way

### Attempt 1: Mounting the image (~15 minutes lost)

My first instinct on any disk image challenge is to mount it and browse the filesystem. It's what forensics investigators do with real disks, and I assumed the flag would be sitting in some file I could just open.

```
$ mkdir /tmp/disko
$ sudo mount -o loop disko-1.dd /tmp/disko
$ ls -la /tmp/disko/
```

The filesystem mounted fine, but the directory was completely empty. I tried `ls -la`, checked for hidden files with `ls -A`, even ran `find /tmp/disko -name "*"` — nothing. Just an empty FAT16 volume.

The reason this fails matters: mounting a FAT image only shows you the *active filesystem entries*. Any file that was deleted, or data written directly to disk sectors without going through the filesystem, is invisible via `mount`. That's actually the whole point of low-level disk forensics — the filesystem is just one view of the raw data.

Also worth noting: `sudo mount` modifies the image's access timestamps. On a real forensics case that's contamination. Always work on a copy, or use read-only mount options (`-o loop,ro`).

### Attempt 2: binwalk (~25 minutes lost)

Empty mount, so maybe there's something embedded in the image itself. binwalk is my go-to for that.

```
$ binwalk disko-1.dd

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             DOS MBR boot record, code offset: 0x58, disk signature: 0x672D4C8C
512           0x200           FAT, 16-bit, sectors/FAT: 20, sector size: 512
```

Just the FAT boot sector and partition metadata. No JPEG signatures, no zip files, no nested archives. I ran `binwalk -e disko-1.dd` anyway hoping for something, got only the raw FAT data extracted, nothing useful.

binwalk finds embedded files by their magic byte signatures. If the flag is stored as plain text in a slack space or deleted sector — no magic bytes, nothing to find.

### Attempt 3: fdisk and Sleuth Kit (~20 minutes lost)

Maybe there's a hidden partition I'm missing. I checked the partition table:

```
$ fdisk -l disko-1.dd
Disk disko-1.dd: 10 MiB, 10485760 bytes, 20480 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x672d4c8c

Device       Boot Start   End Sectors  Size Id Type
disko-1.dd1        32  20479   20448  9.9M  4 FAT16 <32M
```

One partition, FAT16, nothing hidden. I tried The Sleuth Kit anyway:

```
$ fls -r disko-1.dd
(no output)

$ fls -r -d disko-1.dd
(no output)

$ fsstat disko-1.dd
FILE SYSTEM INFORMATION
--------------------------------------------
File System Type: FAT16
...
FAT CONTENTS (in sectors)
--------------------------------------------
(no entries)
```

The FAT has zero directory entries — no active files, no deleted files, nothing. If the flag existed as a proper file, `fls -d` (show deleted) would have caught it. It didn't, because there was never a FAT entry for the flag at all.

At this point I'd been staring at this for about an hour. I stepped back and re-read the challenge name: **DISKO 1**. And then I thought about the simplest possible thing you can do with a binary file when you don't know what's in it.

## The Solution: What I Should Have Tried First

```
$ strings disko-1.dd | grep "pico"
picoCTF{1t5_ju5t_4_5tr1n9_be6031da}
```

That's it. Two seconds. I read that flag — `1t5_ju5t_4_5tr1n9` — and decoded the leet speak: 1=i, t=t, 5=s, 4=a. "it's just a string." The challenge name, the filename, and the flag text were all pointing at the same two-word answer. I just refused to listen for an hour.

### Why strings works here

`strings` doesn't care about filesystem structure, headers, or file formats. It scans the entire binary file and pulls out any sequence of printable ASCII characters above a minimum length (default: 4 characters).

FAT16 divides a disk into fixed-size sectors (512 bytes each) grouped into clusters. If you write bytes directly to a data sector using `dd` without creating a FAT entry, those bytes exist physically on disk but are completely invisible to any filesystem-aware tool. No filename, no cluster chain, no directory entry — just raw bytes at a known offset.

You can confirm exactly where the flag lives using `strings -t x`:

```
$ strings -t x disko-1.dd | grep "pico"
 29800 picoCTF{1t5_ju5t_4_5tr1n9_be6031da}

$ xxd -s 0x29800 -l 64 disko-1.dd
00029800: 7069 636f 4354 467b 3174 355f 6a75 3574  picoCTF{1t5_ju5t
00029810: 5f34 5f35 7431 7233 6e39 5f62 6536 3033  _4_5tr1n9_be603
00029820: 3164 617d 0000 0000 0000 0000 0000 0000  1da}............
```

Offset `0x29800` is sector 211 — just 36 bytes of ASCII text written directly to disk, with no FAT entry pointing to it.

## Full Trial Process

| Step | Action | Command | Result | Why it failed / succeeded |
|------|--------|---------|--------|--------------------------|
| 1 | Identify file type | `file disko-1.dd.gz` | gzip compressed disk image | ✅ Correct first step |
| 2 | Decompress | `gunzip disko-1.dd.gz` | disko-1.dd (FAT16 image) | ✅ Required before any analysis |
| 3 | Mount image | `sudo mount -o loop disko-1.dd /tmp/disko` | Empty directory | ❌ Data written to sectors, not a filesystem file |
| 4 | binwalk scan | `binwalk disko-1.dd` | Only FAT boot sector | ❌ No magic byte signatures |
| 5 | Partition inspection | `fdisk -l disko-1.dd` | Single FAT16 partition | ❌ No hidden partitions |
| 6 | Sleuth Kit file list | `fls -r disko-1.dd` | No output | ❌ No FAT directory entries exist |
| 7 | Raw string scan | `strings disko-1.dd \| grep "pico"` | Flag found immediately | ✅ Scans raw bytes regardless of filesystem |

## Beginner Tips

**The forensics tool ladder** — for disk image challenges, run these in order:

1. `file` — confirm file type
2. `strings | grep "picoCTF{"` — check for plaintext flags immediately
3. `binwalk` — scan for embedded files
4. `mount` — browse active filesystem
5. `fls` / `icat` (Sleuth Kit) — recover deleted files
6. `xxd` / hex editor — raw byte inspection

Running `strings | grep "pico"` costs five seconds. On DISKO 1, it would have saved an hour.

**Filtering strings output on large images:**

```bash
# picoCTF flags always start with "picoCTF{"
strings image.dd | grep "picoCTF{"

# Show offset of each match
strings -t x image.dd | grep "picoCTF{"
```

## What You Learn

**strings before everything else on disk images.** The reflex to mount, analyze partitions, and run forensics tools is correct for harder challenges — but `strings | grep` is a two-second check that costs nothing.

**Filesystem tools only see filesystem data.** mount, fls, and icat all rely on FAT directory entries. Data written below the filesystem layer is invisible to them.

**Real-world parallel:** Malware sometimes hides payloads in unallocated sectors or disk slack space specifically because most OS-level scanning won't catch it. Tools like `photorec` and Autopsy recover data exactly this way — by scanning raw sectors, not trusting the filesystem.

---

*Full writeup with more detail: [alsavaudomila.com](https://alsavaudomila.com/disko-1-picoctf-writeup/)*
