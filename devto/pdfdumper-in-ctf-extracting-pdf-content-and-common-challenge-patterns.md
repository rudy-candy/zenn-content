---
title: "pdfdumper in CTF: Extracting PDF Content and Common Challenge Patterns"
published: true
description: "
🔍 pdfdumper in CTF: Extracting PDF Content and Common Challenge Patterns
If you’ve landed here searching “pdfdumper CTF” or “PDF forensics challenge hidde"
tags: ctf, security, linux, forensics
canonical_url: https://alsavaudomila.com/pdfdumper-in-ctf-extracting-pdf-content-and-common-challenge-patterns/
---

# 🔍 pdfdumper in CTF: Extracting PDF Content and Common Challenge Patterns

If you've landed here searching "pdfdumper CTF" or "PDF forensics challenge hidden data," you're probably staring at a PDF file that looks completely blank — no images, no text, nothing — and wondering where the flag could possibly be hiding. I've been there. The answer is almost always inside the PDF's object structure, and `pdfdumper` is the fastest way to see all of it at once. This article walks through exactly how I found a flag hidden inside a JavaScript stream in a PDF file, including the 15 minutes I wasted going down the wrong path first.

## This Article at a Glance

pdfdumper is a command-line tool that extracts all internal objects from a PDF file and writes them as separate files — streams, JavaScript, fonts, metadata, everything. In CTF forensics challenges, it's the fastest way to see a PDF's complete internal structure in one command. This article covers: when to reach for pdfdumper versus other PDF tools, how to identify which extracted object contains the flag, the Rabbit Hole of treating a PDF as an archive when it isn't, and what the actual decision workflow looks like from file → flag.

## Introduction: A PDF With Nothing Visible and a Flag Somewhere Inside

The challenge — a CTF forensics problem called **Hidden in Plain PDF** — gave me a single PDF file. Opening it in a viewer showed a blank white page — no text, no images, just white. The problem statement said "find the hidden flag." That's it. No hints about format, no mention of encoding. The category was Forensics and the point value suggested medium difficulty, which made me expect something more elaborate than what it turned out to be.

My first instinct, which turned out to be wrong, was that the PDF wasn't really a PDF at all — that it was a disguised archive or an image file with a .pdf extension. This is a real CTF technique, so the instinct wasn't completely irrational. But I acted on it before I actually checked, and that cost me 15 minutes.

I ran `binwalk` on the file looking for embedded archives or images. It found PDF markers and some internal stream data — nothing that looked like a hidden zip or PNG. I tried `file challenge.pdf` to confirm it was actually a PDF. It was. The file was a legitimate PDF. I just hadn't looked inside it yet.

When I finally switched from "is this a disguised file?" to "what's actually inside this PDF?", everything moved fast. That switch in framing — from file format suspicion to structural analysis — is the actual lesson of this challenge.

## What is pdfdumper? (And How It's Different From Other PDF Tools)

### What pdfdumper does

pdfdumper is part of the `pdfminer` suite. It extracts every object inside a PDF — streams, metadata, fonts, JavaScript, form data — and saves them as individual files in a directory. The key advantage over alternatives is speed of overview: one command gives you everything, sorted by object number, without needing to know in advance what you're looking for.
    
    
    $ pdfdumper challenge.pdf --all -d output_dir/

After running this, `output_dir/` contained:
    
    
    output_dir/
    ├── obj1.txt      # catalog
    ├── obj2.txt      # page tree
    ├── obj3.txt      # page object
    ├── obj4.txt      # font resource
    ...
    ├── obj22.js      # ← this one
    ├── obj23.txt     # metadata
    └── obj24.txt     # cross-reference stream

`obj22.js` immediately stood out — a JavaScript object inside a PDF. In real-world malware, JavaScript in PDFs is used to exploit readers. In CTF, it's used to hide encoded data. I opened it:
    
    
    $ cat output_dir/obj22.js
    "ZmxhZ3tQREZfanNfc3RyZWFtX2hpZGRlbl9kYXRhfQ=="

That's Base64. One decode later:
    
    
    $ echo "ZmxhZ3tQREZfanNfc3RyZWFtX2hpZGRlbl9kYXRhfQ==" | base64 -d
    flag{PDF_js_stream_hidden_data}

The flag had been sitting in a JavaScript stream the entire time. The PDF was a real PDF — it just had a non-rendering JS object embedded in it that most viewers silently ignore.

### pdfdumper vs pdf-parser.py vs binwalk — the actual decision

Tool| What it shows| When to use it in CTF| Weakness  
---|---|---|---  
`pdfdumper`| All objects dumped to individual files — instant overview| First tool for any unknown PDF. Fastest path to seeing everything at once.| Output can be large; requires navigating files  
`pdf-parser.py`| Object-by-object inspection with filtering| After pdfdumper, when you know which object type to look for (`--type /JavaScript`)| Slower for broad exploration — you need to know what you're hunting  
`binwalk`| Embedded file signatures inside the binary| Only when you suspect the PDF is actually a disguised archive or has a file appended after the EOF marker| Useless for data hidden inside proper PDF object streams  
`strings`| Raw printable strings in the file| Quick sanity check — does anything look like Base64 or a flag format?| No PDF structure awareness — can miss encoded streams  
  
The critical mistake I made was using `binwalk` first — a tool designed to find files hidden _after_ or _within_ a binary — on a PDF that had data hidden _inside_ its structure. binwalk can't see inside PDF object streams. It's the wrong tool for that job, full stop.

## How to Use pdfdumper: The Actual Workflow

### Step 1 — Quick string check before anything else
    
    
    $ strings challenge.pdf | grep -i "flag\|ctf\|base64\|javascript"

This takes two seconds. If it finds a readable flag directly, you're done immediately. If it finds "JavaScript" or suspicious Base64-looking strings, you know what to hunt for. If it finds nothing useful, move on to full structural analysis.

### Step 2 — Dump all objects with pdfdumper
    
    
    $ mkdir output_dir
    $ pdfdumper challenge.pdf --all -d output_dir/
    $ ls output_dir/

Look at the file extensions in the output. Non-standard extensions — `.js`, `.py`, anything that isn't `.txt` or `.xml` — are immediate red flags. In this challenge, `obj22.js` was the only `.js` file and it was the answer.

### Step 3 — Check JavaScript objects and stream content
    
    
    # Check all JS objects
    $ cat output_dir/*.js
    
    # Or if pdfdumper didn't separate by extension, check all stream objects
    $ for f in output_dir/*; do echo "=== $f ==="; cat "$f"; echo; done | head -200

### Step 4 — Decode whatever you find

Common encodings inside PDF streams in CTF: Base64, hex, zlib-compressed data. The most reliable approach:
    
    
    # Base64
    $ echo "ENCODED_STRING" | base64 -d
    
    # Hex
    $ echo "HEXSTRING" | xxd -r -p
    
    # Zlib-compressed stream (if pdfdumper didn't auto-decompress)
    $ python3 -c "import zlib, sys; print(zlib.decompress(sys.stdin.buffer.read()))" < stream_file

## The Rabbit Hole: Why I Wasted 15 Minutes on binwalk

The mental model I had going in was: "blank PDF = something is being hidden inside another file format." This is a legitimate CTF technique — you can append a zip after a PDF's EOF marker and `binwalk -e` will extract it. So the assumption wasn't absurd. But I didn't verify it before acting on it.

binwalk found PDF internal structure markers and flagged some compressed stream data — which looked suspicious but was just normal PDF content. I spent time trying to interpret those results as evidence of embedded files when they were actually just the PDF working as intended. The right move would have been to run `pdfdumper` immediately and look at the actual content instead of trying to infer it from binary signatures.

The pattern to avoid: **don't choose your analysis tool based on a hypothesis about file format disguise before you've checked the file's structure**. A blank PDF is more likely hiding data inside its own structure than pretending to be an archive. Check the structure first, then suspect disguise if the structure looks suspicious.

Wrong approach| What I thought| Why it failed| What I should have done  
---|---|---|---  
binwalk -e challenge.pdf| "This PDF might be a zip in disguise"| Data was in a PDF JS stream, not appended after EOF| Run pdfdumper first, check object structure  
Searching for embedded PNGs| "Blank page = image hidden in file"| No image objects in the PDF at all| Check what object types actually exist  
file + hexdump first 20 bytes| "Is the magic number wrong?"| File was a legitimate PDF — valid magic number| Magic number being correct doesn't mean content is visible  
  
## Capture the Flag: The .js Extension That Changed Everything

When pdfdumper wrote `obj22.js` to the output directory and I saw that `.js` extension, something clicked. Every other object was `.txt` or metadata. A JavaScript object in a PDF that renders as a blank white page has no legitimate reason to be there — that's not how PDFs with visible content work. It's either a malware vector (in real-world PDFs) or a hiding place for encoded data (in CTF).

Opening the file showed a single quoted string: `"ZmxhZ3tQREZfanNfc3RyZWFtX2hpZGRlbl9kYXRhfQ=="`. The `==` padding at the end confirmed Base64 immediately. The decode was one command:
    
    
    $ echo "ZmxhZ3tQREZfanNfc3RyZWFtX2hpZGRlbl9kYXRhfQ==" | base64 -d
    flag{PDF_js_stream_hidden_data}

Twenty-two minutes in total — 15 of which were the binwalk detour, 5 for running pdfdumper and scanning the output, and about 2 for the decode. If I'd gone straight to pdfdumper, this would have been a 7-minute challenge.

## Full Trial Process Table

Step| Action| Command| Result| Why it failed / succeeded  
---|---|---|---|---  
1| Open the PDF visually| —| Blank white page| No visible content — flag must be hidden in structure or metadata  
2| Check file type| `file challenge.pdf`| Legitimate PDF confirmed| Not a disguised archive — file type suspicion eliminated  
3| binwalk for embedded files| `binwalk -e challenge.pdf`| ❌ No hidden archives found| Data was inside PDF object stream — binwalk can't see that  
4| Try strings for quick flag scan| `strings challenge.pdf | grep flag`| ❌ Nothing readable| Data was Base64-encoded — not plaintext  
5| Dump all PDF objects| `pdfdumper challenge.pdf --all -d out/`| ✅ obj22.js identified| .js extension in a blank PDF = immediate suspect  
6| Inspect JS object| `cat out/obj22.js`| ✅ Base64 string found| Single quoted string with == padding confirmed Base64  
7| Decode Base64| `echo "Zmxh..." | base64 -d`| ✅ Flag obtained| Straightforward decode — no additional encoding layers  
  
## Command Reference

Command| Purpose| When to Use| Notes  
---|---|---|---  
`pdfdumper file.pdf --all -d out/`| Dump all objects to directory| First tool for any unknown PDF| Check file extensions in output dir  
`pdf-parser.py --type /JavaScript file.pdf`| Filter for JavaScript objects only| When pdfdumper shows a JS object exists and you want details| Part of pdf-parser toolkit by Didier Stevens  
`pdf-parser.py --object 22 file.pdf`| Inspect specific object by number| After identifying suspicious object number| More detailed than pdfdumper output  
`strings file.pdf | grep -i "flag\|base64\|ctf"`| Quick flag scan| Always first — takes 1 second| Won't find encoded data but finds plaintext flags instantly  
`echo "STRING" | base64 -d`| Decode Base64| When stream content looks like Base64 (== padding, A-Za-z0-9+/)| Try zlib decompress if this fails  
  
## Beginner Tips

### Installing pdfdumper
    
    
    pip install pdfminer.six

Note: pdfdumper is part of `pdfminer.six` (the Python 3 port of pdfminer). Installing `pdfminer` alone may give you the Python 2 version which behaves differently. Always install `pdfminer.six`.

### What object types to look at first

When pdfdumper produces output, look in this order:

  1. Any `.js` files — JavaScript in a PDF with no visible content is always suspicious
  2. Any stream objects that aren't fonts — `obj*.txt` files with unusual size
  3. Metadata objects — sometimes flags are hidden in XMP metadata or document properties
  4. Form objects — interactive PDF forms can contain hidden fields



### pdfdumper returns an error?
    
    
    # If pdfdumper fails on a malformed PDF, try:
    $ pdf-parser.py challenge.pdf
    
    # Or use qpdf to repair first:
    $ qpdf --qdf challenge.pdf repaired.pdf
    $ pdfdumper repaired.pdf --all -d out/

CTF challenge PDFs are sometimes intentionally malformed (broken cross-reference tables, invalid object lengths). If pdfdumper fails, `pdf-parser.py` is often more tolerant of malformed files and worth trying next.

## What You Learn From This Challenge

The technical skill this challenge teaches is PDF object structure — the fact that a PDF isn't a single blob of content but a collection of numbered objects, each with a type and potentially a stream of data. JavaScript objects, form data, embedded fonts, metadata — all of these exist as discrete objects that most PDF viewers render (or silently ignore) without exposing to the user. pdfdumper makes all of it visible.

In the real world, this matters for malware analysis. PDFs with malicious JavaScript embedded in streams are one of the most common phishing delivery mechanisms. The same technique used to hide a CTF flag — encoded data in a JS stream that renders as a blank page — is used by attackers to deliver payloads that execute when the PDF is opened. Forensic investigators analyzing suspicious PDFs use exactly these tools and this workflow.

### Next Time I'd Solve This in Under 5 Minutes

Run this sequence immediately on any suspicious PDF:
    
    
    # 1. Quick string check (10 seconds)
    strings challenge.pdf | grep -i "flag\|ctf\|base64"
    
    # 2. Dump everything (30 seconds)
    pdfdumper challenge.pdf --all -d out/ && ls -la out/
    
    # 3. Check non-standard extensions first
    cat out/*.js 2>/dev/null || echo "no JS objects"
    
    # 4. Scan all stream content for Base64 patterns
    grep -l "[A-Za-z0-9+/]\{40,\}=*" out/*

The rule I internalized: **a blank PDF is hiding something inside its structure, not pretending to be a different file type**. Check the structure with pdfdumper before reaching for binwalk. binwalk is for files hidden after or around the PDF — pdfdumper is for data inside the PDF. They're solving different problems.

## Further Reading

This article is part of the **Forensics Tools** series. You can see the other tools covered in the series here: [CTF Forensics Tools: The Ultimate Guide for Beginners](<https://alsavaudomila.com/ctf-forensics-tools/>). Introducing the **pdfdumper** command, how to use it in CTF, and common challenge patterns involving hidden PDF object content.

Here are related articles from [alsavaudomila.com](<https://alsavaudomila.com/>) that complement what you've learned here about pdfdumper:

The most natural next step after extracting PDF objects is understanding what other kinds of files can hide data in their internal structure. [binwalk in CTF: How to Analyze Binaries and Extract Hidden Files](<https://alsavaudomila.com/binwalk-in-ctf-how-to-analyze-binaries-and-extract-hidden-files/>) covers the complementary case — when a file really _is_ a disguised archive or has files appended after its legitimate EOF marker. Knowing when to use binwalk versus pdfdumper is a key decision point in PDF forensics challenges.

PDF metadata is a separate hiding place from PDF object streams — and it's often overlooked. [exiftool in CTF: How to Analyze Metadata and Find Hidden Data](<https://alsavaudomila.com/exiftool-in-ctf-how-to-analyze-metadata-and-find-hidden-data/>) covers reading document properties, creation timestamps, and author fields that challenge designers sometimes use to embed flags without touching the PDF's visible content at all.

If the PDF challenge involves an embedded image with structural problems — a common pattern where a PNG inside a PDF has a corrupted chunk — [pngcheck in CTF: How to Analyze and Repair PNG Files](<https://alsavaudomila.com/pngcheck-in-ctf-how-to-analyze-and-repair-png-files/>) covers the tools and techniques for diagnosing and fixing PNG structure issues after you've extracted the image with pdfdumper.
