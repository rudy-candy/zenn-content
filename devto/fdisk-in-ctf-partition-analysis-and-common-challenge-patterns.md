---
title: "fdisk in CTF: Partition Analysis and Common Challenge Patterns"
published: true
description: "
The Day fdisk Showed Me I Was Looking at the Wrong Layer
There’s a specific kind of CTF frustration that comes from staring at a disk image and knowing th"
tags: ctf, security, linux, forensics
canonical_url: https://alsavaudomila.com/fdisk-in-ctf-partition-analysis-and-common-challenge-patterns/
---

## The Day fdisk Showed Me I Was Looking at the Wrong Layer

There's a specific kind of CTF frustration that comes from staring at a disk image and knowing the flag is in there, but having no idea where to even start looking. I hit that wall hard on a picoCTF forensics challenge — one of the "Disk, disk, sleuth!" variants — where I spent close to an hour trying every file-level tool I knew before someone in Discord mentioned `fdisk` and completely changed how I was thinking about the problem.

I had been approaching it all wrong. I was reaching for `binwalk`, `strings`, `foremost` — tools that look at content embedded inside files. But this challenge had a _partition table_ , and the flag was sitting in a partition that no filesystem tool would touch because it was never mounted anywhere. The moment I ran `fdisk -l disk.img` and actually read the output — three partitions, one with a suspicious type ID I'd never seen before — I realized I'd been scanning the surface of the image while the answer was structured one layer deeper.

That's what this article is about. Not just the `fdisk -l` syntax — that takes thirty seconds to learn — but the _mindset shift_ that makes fdisk genuinely useful in CTF forensics. Understanding partition boundaries, sector math, and why tools like Autopsy miss things that live in unallocated space is the difference between solving this class of challenges quickly and wandering through them for hours.

## fdisk Syntax: The One Command That Actually Matters

Unlike `dd`, which has a handful of critical parameters to memorize, fdisk in CTF basically comes down to one command:
    
    
    fdisk -l disk.img

The `-l` flag lists partition information: partition numbers, start and end sectors, total sector count, sector size, and filesystem type identifiers. Everything else you need for CTF work — the sector-to-byte math, the gap calculations, the dd extraction commands — flows from reading this output correctly.

Here's what typical output looks like on a CTF disk image:
    
    
    $ fdisk -l disk.img
    Disk disk.img: 100 MiB, 104857600 bytes, 204800 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    
    Device      Boot  Start    End  Sectors  Size  Id  Type
    disk.img1         2048   43007   40960   20M  83  Linux
    disk.img2        43008  122879   79872   39M   7  HPFS/NTFS/exFAT
    disk.img3       124928  204799   79872   39M  8e  Linux LVM

The fields that matter for CTF: **Start** (first sector of the partition), **Sectors** (total sector count), and **Id** (partition type code). Notice the gap between disk.img2 ending at 122879 and disk.img3 starting at 124928 — those 2048 unallocated sectors are 1MB of space that no filesystem tool will touch unless you're explicitly looking for it. Challenge authors love that gap.

### The Sector-to-Byte Calculation (Do This Once, Do It Right)

Everything in fdisk output is in sectors. Everything in dd (the extraction tool you'll pair with fdisk) is configurable in whatever unit you choose. The simplest approach: set `bs=512` in dd, and the sector values from fdisk map directly to `skip` and `count` without any multiplication.
    
    
    # Extract partition 2 — sector values from fdisk map directly to dd parameters
    $ dd if=disk.img of=partition2.img bs=512 skip=43008 count=79872

If you ever need raw byte offsets — for a hex editor or a tool that takes byte positions — the math is: byte offset = sector × 512. disk.img2 starts at sector 43008, which is byte 22,020,096. But in practice, I almost always use `bs=512` and let the sector numbers do the work directly.

### One Detail That Trips Up Beginners Every Time

When extracting with dd, the `count` parameter is the **Sectors** column from fdisk output — not End minus Start, and not the End value itself. fdisk gives you the total sector count directly in that column; use it. Using the End sector as count will extract from the wrong position entirely, and dd won't warn you — it'll just silently produce an incorrect image. Always double-check: `count` = Sectors column value.

## Rabbit Hole: The Hour I Wasted Before Running fdisk

Here's exactly what I tried before reaching for fdisk — I want to be honest about this because I think it maps to what most beginners attempt:
    
    
    $ file disk.img
    disk.img: DOS/MBR boot record; partition 1 : ID=0x83, start-CHS (0x0,32,33),
    end-CHS (0x14,223,19), startsector 2048, 40960 sectors; partition 2 : ID=0x07...
    # Okay, partitions exist. Let me just mount it...
    
    $ sudo mount -o loop disk.img /mnt/disk
    mount: /mnt/disk: wrong fs type, bad option, bad superblock on /dev/loop0
    # Mounting the raw image without an offset doesn't work on partitioned images.
    # I spent 15 minutes trying different mount flags here.
    
    $ strings disk.img | grep -i "flag\|ctf\|pico"
    (no output)
    # Flag was inside an unmounted filesystem, not raw ASCII in the image
    
    $ binwalk disk.img
    
    DECIMAL       HEXADECIMAL     DESCRIPTION
    --------------------------------------------------------------------------------
    0             0x0             DOS/MBR boot record
    1048576       0x100000        Linux EXT2 filesystem data
    # Saw the EXT2 hit and tried extracting it with dd — got a partial image that
    # mounted but only contained the first partition's content. Missed partition 3.
    
    $ foremost -i disk.img -o output/
    # 12 minutes later: recovered some JPEG files, none relevant
    # foremost carved by signature without partition context — wrong tool here
    
    $ sudo autopsy &
    # Waited 8 minutes for the browser GUI
    # Autopsy found files in partition 1 and 2 — nothing in partition 3
    # It had silently skipped partition 3 due to the unrecognized type ID (0x8e)
    # I didn't know that's what happened until much later

That Autopsy session was the specific failure point. Partition 3 had type ID `8e` (Linux LVM), which in this challenge was a deliberate mislabel — the author set an unusual partition type to make standard tools skip over it while still keeping the actual content as a mountable ext2 filesystem. Autopsy didn't parse it as a recognized filesystem, reported it as empty, and I had no reason to dig further because Autopsy had "scanned everything."

The fix took about ninety seconds once I ran fdisk properly. Saw three partitions, noticed 8e was suspicious, extracted it with dd, ran `file` on the result — it came back as ext2. Mounted it. Flag was in `/home/user/flag.txt`. The entire challenge hinged on knowing that partition type IDs are just labels and can't be trusted.

## Five CTF Patterns Where fdisk Is the Right Starting Point

After working through multiple forensics disk image challenges, the scenarios where fdisk matters fall into recognizable patterns. Here's what each looks like and where beginners typically get stuck:

### Pattern 1: Partition Extraction with dd

The most common pattern. fdisk shows a partition with content; you extract it with dd and analyze the result. The calculation is straightforward but error-prone if you're rushing.
    
    
    $ fdisk -l disk.img
    ...
    Device      Boot  Start    End  Sectors  Size  Id  Type
    disk.img1         2048   43007   40960   20M  83  Linux
    disk.img2        43008  204799  161792   79M  83  Linux
    
    # Extract partition 2: skip=Start, count=Sectors (not End)
    $ dd if=disk.img of=part2.img bs=512 skip=43008 count=161792
    
    $ file part2.img
    part2.img: Linux rev 1.0 ext2 filesystem data
    
    $ mkdir /tmp/mnt && sudo mount part2.img /tmp/mnt
    $ ls /tmp/mnt
    flag.txt  home/  lost+found/

The mistake I made early: confusing the **End** column with the count. End is the last sector number, not the total sector count. fdisk gives you the total directly in the **Sectors** column — that's your `count` value.

### Pattern 2: Hidden or Mislabeled Partitions

CTF authors add partitions with misleading type IDs, or use unusual IDs that standard tools don't recognize and quietly skip. The tell is an unfamiliar type in the `Type` column — things like "Unknown", "W95 FAT32 (LBA)", or "Linux LVM" on an image that's clearly not a production server.
    
    
    $ fdisk -l tricky.img
    ...
    Device       Boot  Start    End  Sectors  Size  Id  Type
    tricky.img1        2048   20479   18432    9M  83  Linux
    tricky.img2       20480   40959   20480   10M  7f  Unknown
    
    # Type 0x7f is unusual — extract and verify what's actually there
    $ dd if=tricky.img of=hidden_part.img bs=512 skip=20480 count=20480
    
    $ file hidden_part.img
    hidden_part.img: Linux rev 1.0 ext2 filesystem data
    # Despite the "Unknown" label — it's actually ext2
    
    $ sudo mount hidden_part.img /tmp/hidden
    $ find /tmp/hidden -name "flag*"
    /tmp/hidden/secret/flag.txt

The rule I follow now: never trust the `Type` label. A partition's type ID is a single byte that anyone can set to anything. Always extract and run `file` on unknown or suspicious types before writing them off.

### Pattern 3: Unallocated Space Between Partitions

Gaps between partition boundaries are invisible to filesystem tools — they don't belong to any partition — but they can contain deleted files, embedded data, or raw flag strings. Spot them by checking whether each partition's End+1 equals the next partition's Start.
    
    
    $ fdisk -l gappy.img
    ...
    Device       Boot  Start    End  Sectors  Size  Id  Type
    gappy.img1         2048   20479   18432    9M  83  Linux
    gappy.img2        24576  204799  180224   88M  83  Linux
    
    # Gap: sectors 20480 to 24575 = 4096 sectors = 2MB of unallocated space
    # That's not alignment padding — 2MB is a deliberate gap in CTF context
    
    $ dd if=gappy.img of=gap.bin bs=512 skip=20480 count=4096
    
    $ strings gap.bin | grep -i "flag\|ctf\|pico"
    picoCTF{h1dd3n_1n_th3_g4p_a7b2c3}

Normal partition alignment padding is typically 2048 sectors (1MB) between sector 0 and the first partition. Gaps larger than that — especially gaps in the middle of the image — are almost always intentional in CTF challenges. Also check the tail: if the last partition ends well before the disk's total sector count, that trailing space is another standard hiding spot.

### Pattern 4: Filesystem Identification for Targeted Analysis

Different partition types contain different kinds of artifacts. Knowing what you're dealing with before you start digging saves significant time. NTFS partitions have Windows event logs, USN journals, and alternate data streams. Linux ext2/3/4 partitions have `.bash_history`, shadow files, and syslog. Swap partitions sometimes contain memory fragments — including plaintext credentials or flag strings that were briefly loaded into RAM.
    
    
    $ fdisk -l multi.img
    ...
    Device      Boot  Start    End  Sectors  Size  Id  Type
    multi.img1         2048   43007   40960   20M  83  Linux       # ext2/3/4
    multi.img2        43008  122879   79872   39M   7  NTFS        # Windows filesystem
    multi.img3       122880  143359   20480   10M  82  Linux swap  # memory fragments
    
    # Swap partition: strings can find in-memory artifacts
    $ dd if=multi.img of=swap.img bs=512 skip=122880 count=20480
    $ strings swap.img | grep -i "password\|flag\|secret" | head -20

Swap partitions are an underrated source in CTF disk imaging challenges. I've found plaintext flags in swap regions on two separate challenges where the filesystem partitions had nothing — the flag had been used in a process that paged memory to swap before the image was captured.

### Pattern 5: Corrupted or Malformed Partition Tables

Sometimes fdisk can't read the partition table cleanly — overlapping sectors, impossible values, a corrupted MBR signature. This is itself a clue: the challenge is about table forensics, not just extracting a known partition. When fdisk fails, switch to `mmls` from Sleuth Kit.
    
    
    $ fdisk -l corrupted.img
    GPT: not present
    MBR: not present
    # fdisk can't find a valid partition structure
    
    # mmls is more tolerant of malformed tables
    $ mmls corrupted.img
    DOS Partition Table
    Offset Sector: 0
    Units are in 512-byte sectors
    
          Slot      Start        End          Length       Description
    000:  Meta      0000000000   0000000000   0000000001   Primary Table (#0)
    001:  -------   0000000000   0000002047   0000002048   Unallocated
    002:  000:000   0000002048   0000043007   0000040960   Linux (0x83)
    003:  -------   0000043008   0000045055   0000002048   Unallocated
    004:  000:001   0000045056   0000204799   0000159744   Unknown Type (0xcc)

mmls explicitly labels unallocated regions and parses tables that fdisk gives up on. The key difference: fdisk relies on the MBR/GPT signature being intact to even begin reading. mmls reads the raw sector data more defensively and will reconstruct partial partition entries even from a damaged table. The partition with type 0xcc was the payload in the challenge above — fdisk returned nothing, but mmls found it and gave me exact Start and Length values I could feed directly to dd. When fdisk says "no partition table found" on a file that `file` identifies as a boot record, switch to mmls immediately.

## fdisk vs Other Tools: How I Actually Decide

fdisk reads partition tables. It doesn't carve files, parse filesystems, or recover deleted data — and reaching for it when those are your actual needs wastes time. Here's how I make the call in practice:

Situation| My First Choice| Why Not fdisk?  
---|---|---  
Disk image with unknown structure| **fdisk -l**|  —  
Corrupted or unreadable partition table| **mmls** (Sleuth Kit)| fdisk gives up on malformed tables; mmls keeps going and shows unallocated regions  
Browse filesystem contents visually| Autopsy / sudo mount| fdisk doesn't open or navigate filesystems  
Carve files without a filesystem| foremost or binwalk| fdisk only reads partition boundaries, not file signatures  
Extract a specific partition| fdisk → then **dd**|  fdisk identifies the boundaries; dd extracts — they're a matched pair  
Detect unallocated gaps| **fdisk -l** (manual gap math)| mmls labels gaps explicitly if you want them surfaced automatically  
NTFS-specific artifacts (ADS, USN journal)| Autopsy or ntfsinfo| fdisk identifies the partition type but can't read NTFS structures  
  
The failure mode I keep seeing in beginners: running Autopsy directly on the raw disk image and trusting that it finds everything. Autopsy works at the filesystem level — it will find files in recognized partitions, but it silently skips partitions with unusual type IDs and won't surface anything in unallocated gaps. Run fdisk first, identify which partitions are worth investigating, then point Autopsy at the specific extracted images rather than the raw disk.

## Full Trial Process Table

Step| Action| Command| Result| Why it failed / succeeded  
---|---|---|---|---  
1| File identification| file disk.img| DOS/MBR boot record| Confirmed it's a partitioned disk — useful, but told me nothing about what was inside each partition  
2| Direct mount| sudo mount -o loop disk.img /mnt| mount: wrong fs type| Mounting the raw image without an offset doesn't work on partitioned images — wasted 15 minutes trying different flags  
3| String search| strings disk.img | grep flag| No output| Flag was inside a mounted filesystem, not raw ASCII in the image  
4| Auto-carve| foremost -i disk.img -o out/| JPEGs recovered, no flag| foremost uses file signatures without partition context — carved false positives from the wrong region  
5| Autopsy full scan| autopsy (GUI)| Files in partitions 1–2, nothing in partition 3| Autopsy silently skipped partition 3 due to unrecognized type ID — I didn't know it had done this  
6| fdisk scan| fdisk -l disk.img| Three partitions visible; type 0x8e on partition 3 looks suspicious| Turning point — saw the full partition layout for the first time  
7| Extract partition 3| dd if=disk.img of=p3.img bs=512 skip=124928 count=79872| Clean image extracted| fdisk sector values mapped directly to dd skip/count — no calculation needed  
8| Verify filesystem type| file p3.img| Linux rev 1.0 ext2 filesystem data| Despite the "Linux LVM" label from fdisk, it's actually ext2 — the partition ID was a deliberate mislabel  
9| Mount and explore| sudo mount p3.img /tmp/p3 && ls /tmp/p3| flag.txt present in root| —  
10| Read flag| cat /tmp/p3/flag.txt| picoCTF{…}| Should have run fdisk at step 1 — everything before step 6 was wasted time  
  
## Why Partition-Level Thinking Matters Beyond CTF

Partition tables exist below the filesystem layer — they're the first structure on any block storage device, before any filesystem driver gets involved. This is why forensic investigators care about them: data in unallocated sectors, between partitions, or in partitions that were never mounted still shows up in a raw partition table scan, even if every filesystem tool reports nothing found.

In real incident response, deleted partitions are a common evidence source. An attacker can delete a partition containing exfiltrated data or tooling, but the bytes remain on disk until overwritten. fdisk and mmls often detect the ghost of a former partition entry — the data may be recoverable even after the partition record is gone. CTF challenge authors use the same principle: they create data structures at the partition layer specifically because most beginners only look at the filesystem layer.

The insight that shifts how you think about disk forensics: a disk image is not a folder. It's a byte sequence with structure at multiple layers — the partition table at the outermost level, filesystems within each partition, individual file structures within those filesystems. Each layer can hide data from the others. fdisk gives you visibility into the outermost layer, which is precisely the one beginners tend to skip.

## How I'd Solve It Faster Next Time

My first-three-minutes workflow for any disk image challenge now — hard-won from running the wrong tools in the wrong order too many times:
    
    
    # Step 1: What kind of image is this?
    file target.img
    
    # Step 2: What partitions exist, and are any suspicious?
    fdisk -l target.img
    # Look for: unusual type IDs, gaps between partitions, trailing unallocated space
    
    # Step 3: Note any gaps
    # gap = next_Start - current_End - 1 (in sectors)
    # If gap > 2048 sectors, extract it:
    dd if=target.img of=gap.bin bs=512 skip=<end_of_prev_part+1> count=<gap_size>
    strings gap.bin | grep -i "flag\|ctf"
    
    # Step 4: Extract each partition that looks interesting
    dd if=target.img of=partN.img bs=512 skip=<Start> count=<Sectors>
    
    # Step 5: Verify what's actually in each extracted partition
    file partN.img
    # Don't trust the fdisk type label — verify with file every time
    
    # Step 6: Mount or analyze
    sudo mount partN.img /tmp/mnt
    ls -la /tmp/mnt

I run `fdisk -l` before anything else now — before binwalk, before strings, before Autopsy. It takes two seconds and immediately tells me whether I'm dealing with a partitioned image (where the partition structure is the puzzle) or a raw binary (where I should switch to binwalk and dd). That decision used to cost me twenty minutes of trying the wrong tools. Now it costs two seconds.

One specific thing to watch: compare the total sector count at the top of fdisk output against the End sector of the last partition. If the last partition ends significantly before the total, that trailing space is another common hiding spot — extract it the same way you'd extract a gap.

## Further Reading

For an overview of the full forensics toolkit that fdisk fits into, [CTF Forensics Tools: The Ultimate Guide for Beginners](<https://alsavaudomila.com/ctf-forensics-tools/>) covers when to reach for each tool — fdisk, dd, binwalk, foremost, Autopsy, and Sleuth Kit all have distinct roles, and understanding that division of labor is half the battle in disk imaging challenges.

Here are related articles from [alsavaudomila.com](<https://alsavaudomila.com/>) that pair directly with this topic:

fdisk finds partition boundaries; [dd](<https://alsavaudomila.com/dd-in-ctf-forensics/>) extracts from them. These two tools are a matched pair in every disk imaging challenge — the article on dd covers the extraction workflow in detail, including the sector math, the `conv=notrunc` flag, and how to handle the count calculation without making mistakes.

Once fdisk identifies a suspicious partition and dd extracts it, [Sleuth Kit and Autopsy](<https://alsavaudomila.com/sleuth-kit-autopsy-ctf/>) handle the filesystem investigation layer — browsing file trees, recovering deleted files, and reading metadata that a simple mount won't surface. The article also covers `mmls` as a more forensically accurate alternative to fdisk when partition tables are corrupted or unusual.

For challenges where the disk image contains embedded files in addition to (or instead of) a partition structure, the [binwalk guide](<https://alsavaudomila.com/binwalk-in-ctf/>) explains how to read scan output accurately, which offsets to trust, and when manual dd extraction beats the auto-extract mode — a natural complement to the partition-level analysis that fdisk handles.
