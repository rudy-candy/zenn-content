---
title: "Scan Surprise picoCTF Writeup"
published: true
description: "
picoCTF forensics challenges come in all shapes, but Scan Surprise from the General Skills category is one where the difficulty isn’t the technique — it’s"
tags: ctf, picoctf, linux, security
canonical_url: https://alsavaudomila.com/scan-surprise-picoctf-writeup/
---

picoCTF forensics challenges come in all shapes, but Scan Surprise from the General Skills category is one where the difficulty isn't the technique — it's knowing which tool exists. I solved this in under two minutes once I figured that out, but getting there took longer than I'd like to admit.

## The Challenge

The challenge gives you a ZIP file. Unzip it and you get a `flag.png`. Open it and you're looking at a QR code.
    
    
    $ unzip challenge.zip
    Archive:  challenge.zip
       creating: home/ctf-player/drop-in/
     extracting: home/ctf-player/drop-in/flag.png

Okay, it's a QR code. First instinct: pull out my phone and scan it. That works — phone cameras read QR codes fine. But in a CTF environment where you're working in a terminal, there's a cleaner way, and figuring that out is the whole point of this challenge.

## The Part Where I Wasted Time

I knew QR codes existed as a challenge type in CTF forensics, but I didn't know there was a dedicated command-line decoder for them. My first thought was to write a Python script using `opencv` or `pyzbar`. I started down that path:
    
    
    # What I tried first (unnecessary)
    pip install pyzbar
    pip install Pillow
    
    # Then started writing:
    from pyzbar.pyzbar import decode
    from PIL import Image
    ...

A few minutes in, I stopped and searched for "qr code cli linux" — and immediately found `zbarimg`. It's a standalone command-line tool that reads QR codes and barcodes from image files directly. No Python, no script, no library imports needed.

That's the actual "surprise" in this challenge: there's already a tool for this. Once you know `zbarimg` exists, the challenge collapses into a single command.

## Installing zbarimg

If you don't have it already:
    
    
    # Debian/Ubuntu (including picoCTF's webshell environment)
    sudo apt install zbar-tools
    
    # Verify it's working
    zbarimg --version

The package is `zbar-tools`, not `zbarimg` — that's the command name, not the package. This tripped me up the first time I tried to install it.

## The Solution
    
    
    $ cd home/ctf-player/drop-in/
    $ zbarimg flag.png
    QR-Code:picoCTF{p33k_@_b00_3f7cf1ae}
    scanned 1 barcode symbols from 1 images in 0 seconds

Flag: **picoCTF{p33k_@_b00_3f7cf1ae}**

The flag text — `p33k_@_b00` — is leet speak for "peek-a-boo." The challenge name "Scan Surprise" is a nod to that. Small detail, but it made me smile when I noticed it.

## What This Challenge Is Actually Testing

Scan Surprise isn't testing your knowledge of QR code internals or image processing. It's testing tool awareness — specifically, whether you know that `zbarimg` exists.

This comes up more than you'd think in CTF forensics. A lot of time gets lost not because the technique is hard, but because competitors don't know a purpose-built tool exists and end up writing scripts from scratch. Knowing the toolbox matters as much as knowing the theory.

Why not just use a phone camera? In a competition, you want reproducible, copy-pasteable output in your terminal — not a screenshot of your phone screen. `zbarimg` gives you that. It also becomes essential when challenges deliberately break QR codes in ways that confuse phone cameras but can be fixed with preprocessing.

## What Harder Versions of This Challenge Look Like

Scan Surprise is the baseline — a clean QR code that zbarimg reads without any preprocessing. Once you've seen this pattern, you'll encounter versions where it's deliberately harder:

  * **Inverted colors** — the QR code has white modules on a black background. zbarimg returns "0 barcodes" because the ISO standard assumes dark-on-light. Fix: `convert -negate` before scanning.
  * **Low resolution** — the image is too small for the decoder to reliably parse. Fix: upscale with `convert -resize` using `-filter point` to keep edges sharp.
  * **QR code inside a video** — a QR code appears in a frame of a video file. Fix: extract frames with ffmpeg, then run zbarimg on the frames.



In all of those cases, the tool is still zbarimg — you just have to preprocess the image first. Scan Surprise establishes the foundation; the harder variants are the same workflow with an extra step in front.

## Further Reading

Here are related articles from [alsavaudomila.com](<https://alsavaudomila.com/>) that build on this challenge:

If you want to go deeper on `zbarimg` itself — including the full diagnostic workflow for when it returns "0 barcodes" — the tool guide at [zbarimg in CTF](<https://alsavaudomila.com/zbarimg-in-ctf-qr-barcode-decoding-techniques-and-common-challenge-patterns/>) covers each failure mode with working commands.

For a broader map of which forensics tools to use depending on file type, [CTF Forensics Tools: The Ultimate Guide for Beginners](<https://alsavaudomila.com/ctf-forensics-tools/>) covers the full decision process from file identification through to extraction.
