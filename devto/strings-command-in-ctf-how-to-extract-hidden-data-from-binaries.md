---
title: "strings Command in CTF: Hidden Data Guide"
published: true
description: "strings in CTF: how to extract hidden flags from binaries, filter noise, and use offset flags — with real picoCTF examples."
tags: ctf, security, linux, forensics
canonical_url: https://alsavaudomila.com/strings-command-in-ctf-how-to-extract-hidden-data-from-binaries/
---

# picoCTF Ph4nt0m 1ntrud3r — Network Forensics Writeup

Category: Forensics | Difficulty: Easy | Competition: picoCTF

## Challenge Overview

The **picoCTF Ph4nt0m 1ntrud3r** challenge drops you into a classic network forensics scenario: you receive a PCAP file and need to figure out what a mystery attacker was smuggling across the wire. No binary exploitation, no cryptographic math — just you, Wireshark, and a packet capture that hides a fragmented flag across multiple packets. I'll be honest: I thought this would take me ten minutes. It took closer to ninety, mostly because I spent the first half-hour confidently doing the wrong thing.

This writeup covers the full investigation: my initial wrong approach, the rabbit hole I fell into, the exact Wireshark filters I used, the Python decoder I wrote, and what I'd do differently if I had to solve this again from scratch.

## My First (Wrong) Approach — And Why I Chose It

When I opened `evidence.pcap` in Wireshark, my gut reaction was to look at DNS traffic. In a lot of CTF forensics problems, exfiltration happens over DNS because defenders often under-monitor it. Long, weirdly-encoded subdomains are a classic data-hiding technique. I filtered on `dns` immediately and started staring at query names, convinced I was about to find something like `cGljb0NURg==.attacker.evil`.

There was nothing. Some boring A-record lookups, nothing that looked hand-crafted. I switched to HTTP, thinking maybe the flag was in a User-Agent header or a URL parameter. Still nothing suspicious. I then tried `tcp contains "picoCTF"` as a raw string search — also empty. At this point I had burned roughly 35 minutes and had zero leads.

The reason I kept chasing these paths is that they work in a lot of other CTF challenges, and pattern-matching from past experience can be a trap. I was looking for the shape of a problem I'd solved before instead of reading the actual data in front of me.

### The Rabbit Hole: Manual Packet Inspection

After the DNS and HTTP dead ends, I started scrolling through packets manually, reading payloads one by one. This is exactly the kind of approach that sounds thorough but is actually just slow. I found a few packets with short string payloads and tried to read them as ASCII flags. One of them had what looked like a partial "picoC" — I got excited, copied it out wrong because I was working from a hex dump, and spent another fifteen minutes trying to figure out why my "flag" was garbled nonsense.

That manual copy failure was the moment I finally stopped and thought about the problem differently. If the data is fragmented and Base64-encoded, manual copying from hex dumps is going to produce errors every single time. I needed to sort by something structural and then automate the extraction.

## Setting Up the Investigation Environment

Tools used for this challenge:

  * Wireshark 4.x (packet capture analysis)
  * Python 3.11 (Base64 decoding script)
  * `tshark` (command-line Wireshark for batch extraction)
  * A Linux terminal with `base64` utility for quick spot checks



Nothing exotic. The point of forensics challenges at this level is usually that the tools are simple and the insight is the hard part.

## Digging Into the PCAP with Wireshark

### Initial Triage: What Does This Traffic Even Look Like?

After abandoning the manual scroll approach, I went back to basics. In Wireshark, I opened the Statistics menu and ran **Protocol Hierarchy** first. This gives you an instant breakdown of what protocols are present in the capture without requiring you to guess. The capture was mostly TCP with a small cluster of short application-layer payloads that didn't map cleanly to any known protocol — that asymmetry was the first real signal.

Next I sorted packets by **Length** (ascending). This is a move I wish I'd made at the start. The attacker's fragments were short — consistently around 12–16 bytes of payload — while the rest of the traffic had normal-sized packets. That uniform small size is unusual and stands out immediately once you sort by length.

### Applying Wireshark Filters

Once I had a hypothesis — short payloads, possibly Base64 — I used the following display filter to isolate TCP segments with small application data:
    
    
    tcp.len > 0 and tcp.len < 20

This filtered down to a manageable set of packets. Looking at the Follow TCP Stream output on one of them:
    
    
    Wireshark > Right-click packet > Follow > TCP Stream
    
    Stream content (ASCII view):
    cGljb0NURg==
    ezF0X3c0cw==
    bnRfdGg0dA==
    XzM0c3lfdA==
    YmhfNHJfOQ==
    NjZkMGJmYg==
    fQ==

Seven short strings. Every single one ends in `=` or `==`. That trailing equals sign is the unmistakable fingerprint of Base64 padding — it appears when the input length isn't a multiple of three bytes. Seeing it once might be coincidence. Seeing it seven times in a row is a pattern that can only mean one thing.

I also used `tshark` from the command line to extract these payloads more cleanly, which avoids the copy-paste errors I'd been making earlier:
    
    
    tshark -r evidence.pcap -Y "tcp.len > 0 and tcp.len < 20" -T fields -e data.text

Output:
    
    
    cGljb0NURg==
    ezF0X3c0cw==
    bnRfdGg0dA==
    XzM0c3lfdA==
    YmhfNHJfOQ==
    NjZkMGJmYg==
    fQ==

Clean extraction, no manual copying. This is what I should have done from the beginning instead of scrolling through hex dumps by hand.

### The Importance of Timestamp Order

One subtlety worth noting: network packets don't necessarily arrive in the order they were sent. TCP handles reordering at the transport layer, but if you're extracting application-layer fragments manually, you need to sort by the original timestamp — not by the order Wireshark received them. In this challenge the packets happened to arrive in sequence, but in a real incident response scenario, out-of-order fragments are a deliberate anti-forensics technique. Always sort by time first, then extract.

## Recognizing the Base64 Pattern

Before writing the decoder, I did a quick sanity check on the first fragment using the command line:
    
    
    $ echo "cGljb0NURg==" | base64 --decode
    picoCTF

That was the moment everything clicked. `picoCTF` — the first fragment is literally the competition name and flag prefix. The attacker (or in this case, the challenge author) split the flag at a seven-character boundary and encoded each chunk separately. The decode confirms: I have the right data, I have the right encoding, and I just need to concatenate all seven decoded strings.

Let me be specific about that feeling: it's genuinely satisfying after 35 minutes of wrong guesses to see a word you recognize come out of a decoder. Not triumphant — more like the relief when you finally find your keys after tearing the house apart. The work isn't done yet but now at least you know what you're doing.

## Writing the Decoder Script

With all seven fragments confirmed, the decoder is straightforward:
    
    
    import base64
    
    # Fragments extracted from PCAP via tshark, sorted by timestamp
    cipher = [
        "cGljb0NURg==",   # fragment 1
        "ezF0X3c0cw==",   # fragment 2
        "bnRfdGg0dA==",   # fragment 3
        "XzM0c3lfdA==",   # fragment 4
        "YmhfNHJfOQ==",   # fragment 5
        "NjZkMGJmYg==",   # fragment 6
        "fQ=="            # fragment 7
    ]
    
    plain = ""
    for i, c in enumerate(cipher):
        decoded = base64.b64decode(c).decode("utf-8")
        print(f"Fragment {i+1}: {c!r:20s} => {decoded!r}")
        plain += decoded
    
    print()
    print("Assembled flag:", plain)

Execution output:
    
    
    $ python3 decode_flag.py
    Fragment 1: 'cGljb0NURg=='      => 'picoCTF'
    Fragment 2: 'ezF0X3c0cw=='      => '{1t_w4s'
    Fragment 3: 'bnRfdGg0dA=='      => 'nt_th4t'
    Fragment 4: 'XzM0c3lfdA=='      => '_34sy_t'
    Fragment 5: 'YmhfNHJfOQ=='      => 'bh_4r_9'
    Fragment 6: 'NjZkMGJmYg=='      => '66d0bfb'
    Fragment 7: 'fQ=='              => '}'
    
    Assembled flag: picoCTF{1t_w4snt_th4t_34sy_tbh_4r_966d0bfb}

Flag: `picoCTF{1t_w4snt_th4t_34sy_tbh_4r_966d0bfb}`

The flag text itself is a small joke by the challenge author — "it wasn't that easy, tbh" — which I found funnier after spending 90 minutes on what is technically an "Easy" challenge.

## Full Trial Process Table

Here is every approach I tried during this challenge, in order:

Step| Action| Command / Filter| Result| Why it failed / succeeded  
---|---|---|---|---  
1| Filter DNS traffic| `dns`| Only standard A-record lookups, nothing encoded| Wrong assumption — exfiltration wasn't DNS-based  
2| Filter HTTP traffic| `http`| No suspicious headers or URL params| Wrong protocol assumption from past CTF patterns  
3| Raw string search for flag prefix| `tcp contains "picoCTF"`| No matches| Flag was Base64-encoded, not plaintext — search missed it  
4| Manual hex dump scroll| (manual, no filter)| Found short payloads but copied incorrectly| Human error in transcribing hex; garbled output  
5| Protocol hierarchy check| Statistics > Protocol Hierarchy| Identified anomalous short TCP payloads| Right direction — structural anomaly visible  
6| Sort by packet length| Column sort in Wireshark UI| Small cluster of 12–16 byte payloads visible| Attacker's fragments isolated from normal traffic  
7| Filter short TCP payloads| `tcp.len > 0 and tcp.len < 20`| Seven packets isolated| Correct filter; exact fragments found  
8| Follow TCP stream| Right-click > Follow > TCP Stream| All seven Base64 strings visible in sequence| Confirmed data and order; saw "=" padding pattern  
9| tshark command-line extraction| `tshark -r evidence.pcap -Y "tcp.len > 0 and tcp.len < 20" -T fields -e data.text`| Clean list of seven Base64 fragments| No manual copy error; clean input for Python script  
10| Quick spot decode| `echo "cGljb0NURg==" | base64 --decode`| `picoCTF`| "Aha" moment confirmed — right encoding, right data  
11| Python decoder script| `python3 decode_flag.py`| Full flag assembled: `picoCTF{1t_w4snt_th4t_34sy_tbh_4r_966d0bfb}`| All fragments decoded and concatenated correctly  
  
## Technical Deep Dive — Why Attackers Fragment Data This Way

### Data Fragmentation as an Evasion Technique

This challenge models a real attacker behavior: splitting exfiltrated data into small chunks to evade detection. Signature-based intrusion detection systems (IDS) look for known patterns — if a full flag string or a recognizable file header appears in a single packet, an alert fires. But if that same data is split into seven fragments of 8–12 bytes each, each encoded in Base64 (which looks like random alphanumeric noise to a pattern matcher), the same IDS might let every packet through individually.

Base64 encoding adds another layer of deniability. It transforms binary or text data into a character set that looks like ordinary web traffic — Base64 appears constantly in legitimate email attachments, image data URIs, and API tokens. A network defender scanning for "weird-looking traffic" might not flag short Base64 strings without specific tuning.

### Real-World Network Forensics Parallels

In professional incident response and digital forensics, Wireshark and `tshark` are standard tools that security operations center (SOC) analysts and DFIR (Digital Forensics and Incident Response) specialists use daily. The workflow in this challenge — capture traffic, identify anomalous patterns, extract and decode payloads — mirrors what a real analyst does when investigating suspected data exfiltration.

Some concrete real-world parallels:

  * **APT exfiltration campaigns** often use DNS tunneling or HTTP with Base64-encoded payloads in headers — the same encoding technique used here, just over a different protocol
  * **Malware command-and-control (C2) traffic** frequently uses short, regular beacons with encoded payloads; identifying the "attacker's packets" by their unusual size and periodicity is a standard detection heuristic
  * **Network traffic analysis (NTA) tools** like Zeek/Bro and Suricata implement exactly the kind of length-based filtering we did manually here — they flag short TCP streams with encoded payloads as potential exfiltration candidates
  * **DFIR tools** like NetworkMiner automate the extraction of payloads from PCAP files, doing at scale what we did by hand in this challenge



The skills this challenge teaches — statistical anomaly detection in traffic, protocol filter construction, payload extraction, encoding recognition — are directly transferable to entry-level SOC analyst work. This isn't just a CTF puzzle; it's a stripped-down version of a real investigation workflow.

### Why Base64 Specifically?

Base64 is not encryption — it provides no confidentiality. Anyone who sees the encoded string can decode it trivially. The reason it shows up in CTF challenges and in real attacks is that it solves a different problem: binary data compatibility. Network protocols, email systems, and web applications are often designed to handle text. Base64 encodes arbitrary binary data as printable ASCII characters, making it safe to embed in text-only contexts. Attackers use it not to hide data from sophisticated defenders, but to get it through infrastructure that would otherwise mangle or block binary payloads.

## Reflection — How I Would Solve This Faster Next Time

Looking back at this challenge with the benefit of knowing the answer, my 90-minute solve breaks down roughly as:

  * 35 minutes: chasing DNS and HTTP false leads
  * 20 minutes: manual hex dump scrolling and failed copy attempts
  * 15 minutes: realizing I should check protocol hierarchy and sort by length
  * 10 minutes: applying the right filter and extracting fragments
  * 10 minutes: writing and running the decoder



The first 55 minutes were waste. Here's the checklist I'd follow if I had to do this again from the start:

  1. **Protocol Hierarchy first, always.** Before applying any filters, run Statistics > Protocol Hierarchy. This takes 10 seconds and tells you exactly what you're dealing with.
  2. **Sort by packet length before scrolling.** Attackers' fragments usually stand out by size. Sort ascending, look for clusters of unusually short or unusually long packets.
  3. **Use tshark for extraction, not manual copy.** The moment you're copy-pasting hex or ASCII from Wireshark by hand, you're introducing errors. Automate extraction from the start.
  4. **Spot-test the encoding before writing a full script.** A one-liner (`echo "..." | base64 --decode`) confirms your hypothesis before you invest time scripting.
  5. **Don't assume past patterns apply.** DNS exfiltration, HTTP header hiding — these work in many challenges. But the first thing to do is read the actual data, not apply heuristics from previous problems.



If I applied this checklist, I think I could solve this challenge in under 15 minutes. The solution is genuinely straightforward — the difficulty is resisting the urge to jump to conclusions and doing the unglamorous structural analysis first.

## Key Takeaways

  * **Wireshark filter to isolate short TCP payloads:** `tcp.len > 0 and tcp.len < 20`
  * **tshark command to extract payload text:** `tshark -r evidence.pcap -Y "tcp.len > 0 and tcp.len < 20" -T fields -e data.text`
  * **Base64 tell:** trailing `=` or `==` padding in all fragments
  * **Lesson:** Start with protocol-level statistics, not protocol assumptions
  * **Real-world connection:** This workflow (size anomaly detection → payload extraction → encoding analysis) is standard network forensics practice in SOC and DFIR environments



picoCTF Ph4nt0m 1ntrud3r is a well-constructed introductory forensics challenge because it teaches a real investigative pattern, not just a trick. The "easy" rating is accurate once you know what to look for — but getting to that moment of knowing takes most of the work.
