---
title: "Analyzing ZIP Encryption: When to Act"
published: true
description: "picoCTF archive_madness with zip2john: the 20-minute wrong turn through fcrackzip how to spot AES-256 vs ZipCrypto and the custom wordlist that won."
tags: ctf, security, linux, forensics
canonical_url: https://alsavaudomila.com/zip2john-in-ctf-extracting-zip-passwords-and-common-challenge-patterns/
---

$ fcrackzip -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    

Still nothing. I checked if the file was corrupted. I re-downloaded it. Same result. I tried the verbose flag:
    
    
    $ fcrackzip -v -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    found file 'flag.txt', (size cp/uc    173/   161, flags 1, chk 8a75)
    

It found the file. It attempted passwords. It never cracked it. This was the moment I finally went back to that `0x33` version byte from the hexdump.

Version 51 in a ZIP local file header means AES encryption — specifically, it means the file uses WinZip's AE-1 or AE-2 AES extension. `fcrackzip` doesn't support AES-encrypted ZIPs. It works fine against the older ZipCrypto (traditional PKZIP encryption), but AES is a different animal entirely. The tool was silently skipping every candidate because it couldn't even verify them correctly. I'd been pouring water into a bucket with no bottom.
    
    
    $ zipinfo -v secret_archive.zip
    Archive:  secret_archive.zip
    There is no zipfile comment.
    
    End-of-central-directory record:
    -------------------------------
    ...
    Central directory entry #1:
    ---------------------------
      flag.txt
      ...
      file security status:                 encrypted
      extended local header:                no
      file last modified on (DOS date/time):  ...
      32-bit CRC value (hex):               00000000
      compressed size:                      173 bytes
      uncompressed size:                    161 bytes
      length of filename:                   8 characters
      ...
      encryption method:                    AES Encryption
      AES key size:                         256-bit

AES-256. That explained everything. `fcrackzip` was never going to work. I needed a tool that understood this encryption format — and that's exactly what `zip2john` is built for.

## What zip2john Actually Does (And Why It Matters)

Before I show the commands that worked, it's worth understanding the mechanism. `zip2john` is part of the John the Ripper suite. Its job is not to crack passwords — it's to extract a crackable hash from the ZIP file's encryption metadata.

For AES-encrypted ZIPs (the WinZip AE format), the ZIP file stores a salt value and a verification value in the local file header. These are derived from the password using PBKDF2-HMAC-SHA1. `zip2john` reads these values and formats them into a hash string that John the Ripper (or hashcat, with the right module) can work with. The hash includes the salt, the number of iterations, and the verification bytes — everything needed to test a candidate password without actually decrypting the file contents.

This matters for a non-obvious reason: AES-256 ZIP encryption is actually strong. Unlike the old ZipCrypto (which has known plaintext attacks that can bypass brute force entirely), AES-256 ZIP has no practical cryptographic weakness. You're doing a genuine dictionary or brute force attack. That means wordlist quality is everything.

## The Commands That Actually Worked

### Step 1: Extract the Hash
    
    
    $ zip2john secret_archive.zip > archive.hash
    ver 2.0 efh 5455 efh 7875 secret_archive.zip/flag.txt PKZIP Encr: TS_chk, cmplen=173, decmplen=161, crc=0000 ts=8A75 cs=8a75 type=8
    
    
    
    $ cat archive.hash
    secret_archive.zip/flag.txt:$zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...[hash data]...*$/zip2$:flag.txt:secret_archive.zip::

That hash string is what John will work with. The `$zip2$` prefix tells John this is an AES-encrypted ZIP. The fields encode the salt, key length, verification data, and encrypted content sample.

### Step 2: Build a Targeted Wordlist

Before throwing rockyou.txt at it (which would take hours on CPU), I made a small targeted wordlist from everything I knew about the challenge:
    
    
    $ cat > custom.txt << 'EOF'
    archive
    madness
    secret
    archive_madness
    picoctf
    pico
    ctf
    flag
    secret_archive
    archivemadness
    picoCTF2024
    picoCTF2025
    zip
    password
    EOF

### Step 3: Run John with the Custom Wordlist First
    
    
    $ john --wordlist=custom.txt archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    0g 0:00:00:00 DONE (2024-11-14 14:23:10) 0g/s 933.3p/s 933.3p/s 933.3p/s ..
    Session completed.

No hit. The obvious guesses weren't it. Time for John's mangling rules:
    
    
    $ john --wordlist=custom.txt --rules=Single archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    archive2024       (secret_archive.zip/flag.txt)
    1g 0:00:00:03 DONE (2024-11-14 14:23:47) 0.3030g/s 8381p/s 8381p/s 8381p/s ..
    Use the "--show" option to display all cracked passwords reliably
    Session completed.

There it was. `archive2024`. John's Single rule set had appended "2024" to "archive" — a common CTF password construction pattern. The whole crack took about three seconds once I had the right wordlist.

### Step 4: Verify with --show and Extract the Flag
    
    
    $ john --show archive.hash
    secret_archive.zip/flag.txt:archive2024:flag.txt:secret_archive.zip::
    
    1 password hash cracked, 0 left
    
    
    $ unzip -P archive2024 secret_archive.zip
    Archive:  secret_archive.zip
      inflating: flag.txt
    
    $ cat flag.txt
    picoCTF{z1p_crypt0_1s_n0t_s3cur3_w1th_w34k_p4ssw0rds}

Flag captured. 200 points. I sat there for a moment just staring at the terminal.

## The Moment It Clicked

The real turning point wasn't finding the password. It was going back to that hexdump and finally paying attention to the version byte I'd noticed and ignored. Version `0x33` (decimal 51) in a ZIP local file header is documented in the ZIP specification as indicating AES encryption. I'd seen the number. I hadn't bothered to look up what it meant.

Once I ran `zipinfo -v` and saw "AES Encryption / 256-bit" explicitly printed, everything reorganized in my head. `fcrackzip` failing silently made sense. The need for `zip2john` specifically made sense. The whole challenge was designed around knowing that distinction.

That's the thing about CTF forensics challenges — the file format usually tells you exactly what you need to know. You just have to actually read it.

## Full Trial Process: Every Step I Took

Step| Action| Command| Result| Why it failed / succeeded  
---|---|---|---|---  
1| Identify file type| `file secret_archive.zip`| ZIP archive, v2.0+| Succeeded — confirmed it's a valid ZIP  
2| Attempt extraction| `unzip secret_archive.zip`| Password prompt, then failure| Expected — file is encrypted  
3| Manual password guessing| `unzip -P <word> secret_archive.zip`| 15 attempts, all failed| Failed — password not an obvious keyword  
4| Check metadata| `strings secret_archive.zip`| Only filename visible| Failed — AES content is opaque to strings  
5| Binary inspection| `hexdump -C secret_archive.zip`| Saw version 0x33 byte| Partial — noticed AES indicator but didn't act on it  
6| fcrackzip dictionary attack| `fcrackzip -u -D -p rockyou.txt ...`| Silent, no output| Failed — fcrackzip doesn't support AES-encrypted ZIPs  
7| Confirm encryption type| `zipinfo -v secret_archive.zip`| AES-256 confirmed| Succeeded — this was the key diagnostic step  
8| Extract hash with zip2john| `zip2john secret_archive.zip > archive.hash`| $zip2$ hash generated| Succeeded — correct tool for AES ZIP  
9| John with custom wordlist| `john --wordlist=custom.txt archive.hash`| 0 hits| Failed — exact words not the password  
10| John with Single rules| `john --wordlist=custom.txt --rules=Single archive.hash`| archive2024 cracked in 3s| Succeeded — rule mangling hit the pattern  
11| Extract flag| `unzip -P archive2024 secret_archive.zip`| flag.txt extracted| Succeeded — password confirmed  
  
## Technical Background: Why AES-256 ZIP Is Strong But Passwords Aren't

AES-256 is genuinely secure encryption. The key itself — 256 bits of random data — would take longer than the age of the universe to brute force with any foreseeable hardware. The weakness isn't the algorithm. It's the password.

ZIP's AES implementation (the WinZip AE format) derives the actual encryption key from the password using PBKDF2-HMAC-SHA1 with 1000 iterations. This is deliberately slow to make brute forcing expensive — but 1000 iterations is not especially slow by modern standards. On a modern CPU you can test roughly 50,000–100,000 passwords per second. On a GPU with hashcat and the right module (`-m 13600`), that number climbs to millions per second.

The practical consequence: an AES-256 ZIP is only as secure as its password. "archive2024" is 11 characters with a simple pattern — a dictionary word plus a year. In real-world password auditing, that's classified as weak. It would fall to any competent dictionary attack within minutes even without knowing the target context.

### ZipCrypto vs. AES: Why the Distinction Matters

The older ZipCrypto (traditional PKZIP) encryption is even weaker for a different reason. It has a known-plaintext attack: if you know any 12 bytes of the plaintext (which is trivially true for ZIP files, since the local file header structure is predictable), tools like `pkcrack` can recover the encryption keys directly without knowing the password at all. This attack works regardless of password complexity.

When `zipinfo -v` shows "ZipCrypto" instead of "AES Encryption," your first move should be checking if a plaintext attack is viable — not running john at all. That's a completely different solve path.

### Real-World Context: Where This Shows Up Outside CTF

Password-protected ZIP files appear constantly in malware delivery — phishing emails often include encrypted ZIPs specifically because the password (included in the email body) prevents automated scanning tools from inspecting the payload. Security researchers and incident responders use exactly these tools — `zip2john`, john, hashcat — to recover credentials from seized devices and investigate malware samples. The CTF skill and the professional skill are the same skill.

There are also legitimate recovery scenarios: encrypted archives from former employees, old backup files with forgotten passwords, forensic examination of devices where the user is unavailable. The legal and ethical boundaries matter here — using these tools against files you don't have authorization to access is a different matter entirely.

## zip2john vs. hashcat: When to Use Which

`zip2john` \+ john is the right default when you're on a CPU-only system or when you expect a small, targeted wordlist to work quickly. John's rule sets (especially Single and Jumbo) are excellent at generating contextually plausible password variations from a seed wordlist — exactly what you want in CTF where the challenge setter chose a password related to the theme.

hashcat with `-m 13600` (WinZip) is the right choice when you need raw throughput — exhaustive wordlist runs against rockyou.txt, combinatory attacks, or mask attacks for known-length passwords with unknown character sets. GPU acceleration makes a substantial difference here, but you need to extract the hash first with `zip2john` in either case.
    
    
    # hashcat equivalent (after extracting hash with zip2john)
    $ hashcat -m 13600 -a 0 archive.hash /usr/share/wordlists/rockyou.txt
    hashcat (v6.2.6) starting...
    
    OpenCL API (OpenCL 3.0 PoCL 3.1+debian) - Platform #1 [The pocl project]
    ==========================================================================
    * Device #1: pthread-Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz, 13579/27222 MB (4096 MB allocatable), 4MCU
    
    Minimum password length supported by kernel: 0
    Maximum password length supported by kernel: 256
    
    Hashes: 1 digests; 1 unique digests, 1 unique salts
    ...
    Session..........: hashcat
    Status...........: Cracked
    Hash.Mode........: 13600 (WinZip)
    Hash.Target......: $zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...
    Time.Started.....: Thu Nov 14 14:35:02 2024 (1 min, 43 secs)
    Candidates.#1....: archive2024
    
    Started: Thu Nov 14 14:35:02 2024
    Stopped: Thu Nov 14 14:36:45 2024

## What I'd Do Differently Next Time

The biggest time sink wasn't the failed commands — it was not confirming the encryption type before choosing tools. `zipinfo -v` takes two seconds. If I'd run that before anything else, I would have skipped the entire `fcrackzip` detour and gone straight to `zip2john`.

My updated mental checklist for any encrypted ZIP in CTF:

  1. Run `zipinfo -v` immediately. Identify ZipCrypto vs. AES, and note the AES key size.
  2. If ZipCrypto: consider known-plaintext attack with pkcrack before any password cracking.
  3. If AES: run `zip2john` to extract the hash. Do not touch fcrackzip.
  4. Build a targeted wordlist from every word in the challenge title, description, and filename. Five minutes here saves an hour later.
  5. Run John with that wordlist first (`--rules=Single`). If no hit within 30 seconds, try `--rules=Jumbo`.
  6. If still stuck: rockyou.txt without rules. Then rockyou.txt with rules. Then reconsider whether password cracking is actually the path.
  7. Cap the entire attempt at 15 minutes before stepping back and asking if there's a different angle entirely.



The time limit matters. CTF challenges are designed to be solvable. If you've been running wordlists for 30 minutes with no result, the answer probably isn't "try a bigger wordlist." It's more likely that you're missing something about the challenge design.

## Quick Reference: zip2john Commands
    
    
    # Check encryption type first
    zipinfo -v target.zip
    
    # Extract hash
    zip2john target.zip > target.hash
    
    # Crack with custom wordlist + rules
    john --wordlist=custom.txt --rules=Single target.hash
    
    # Crack with rockyou
    john --wordlist=/usr/share/wordlists/rockyou.txt target.hash
    
    # Show cracked password
    john --show target.hash
    
    # hashcat equivalent (GPU)
    hashcat -m 13600 -a 0 target.hash /usr/share/wordlists/rockyou.txt
    
    # Extract with recovered password
    unzip -P <password> target.zip

## Final Thoughts

picoCTF's _archive_madness_ is a straightforward challenge once you know what you're doing — but I didn't know what I was doing for the first 20 minutes of it. The failure wasn't lack of knowledge about `zip2john`. It was not checking the basic file properties before reaching for tools.

The broader lesson, which I keep relearning in different forms: read the file before attacking it. ZIP headers, ELF headers, PNG chunks, PDF structure — they all tell you what the file actually is and what's relevant about it. Starting with `file` and then a format-specific inspector is worth more than jumping straight to attack tools.

If you found this writeup through a search for zip2john, I hope the failure log was as useful as the working commands. The 20 wasted minutes are the part worth reading.
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt secret_archive.zip
    

It ran. And ran. And ran. No output. No progress indicator. After five minutes I hit Ctrl+C and tried a smaller wordlist:
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    

Still nothing. I checked if the file was corrupted. I re-downloaded it. Same result. I tried the verbose flag:
    
    
    $ fcrackzip -v -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    found file 'flag.txt', (size cp/uc    173/   161, flags 1, chk 8a75)
    

It found the file. It attempted passwords. It never cracked it. This was the moment I finally went back to that `0x33` version byte from the hexdump.

Version 51 in a ZIP local file header means AES encryption — specifically, it means the file uses WinZip's AE-1 or AE-2 AES extension. `fcrackzip` doesn't support AES-encrypted ZIPs. It works fine against the older ZipCrypto (traditional PKZIP encryption), but AES is a different animal entirely. The tool was silently skipping every candidate because it couldn't even verify them correctly. I'd been pouring water into a bucket with no bottom.
    
    
    $ zipinfo -v secret_archive.zip
    Archive:  secret_archive.zip
    There is no zipfile comment.
    
    End-of-central-directory record:
    -------------------------------
    ...
    Central directory entry #1:
    ---------------------------
      flag.txt
      ...
      file security status:                 encrypted
      extended local header:                no
      file last modified on (DOS date/time):  ...
      32-bit CRC value (hex):               00000000
      compressed size:                      173 bytes
      uncompressed size:                    161 bytes
      length of filename:                   8 characters
      ...
      encryption method:                    AES Encryption
      AES key size:                         256-bit

AES-256. That explained everything. `fcrackzip` was never going to work. I needed a tool that understood this encryption format — and that's exactly what `zip2john` is built for.

## What zip2john Actually Does (And Why It Matters)

Before I show the commands that worked, it's worth understanding the mechanism. `zip2john` is part of the John the Ripper suite. Its job is not to crack passwords — it's to extract a crackable hash from the ZIP file's encryption metadata.

For AES-encrypted ZIPs (the WinZip AE format), the ZIP file stores a salt value and a verification value in the local file header. These are derived from the password using PBKDF2-HMAC-SHA1. `zip2john` reads these values and formats them into a hash string that John the Ripper (or hashcat, with the right module) can work with. The hash includes the salt, the number of iterations, and the verification bytes — everything needed to test a candidate password without actually decrypting the file contents.

This matters for a non-obvious reason: AES-256 ZIP encryption is actually strong. Unlike the old ZipCrypto (which has known plaintext attacks that can bypass brute force entirely), AES-256 ZIP has no practical cryptographic weakness. You're doing a genuine dictionary or brute force attack. That means wordlist quality is everything.

## The Commands That Actually Worked

### Step 1: Extract the Hash
    
    
    $ zip2john secret_archive.zip > archive.hash
    ver 2.0 efh 5455 efh 7875 secret_archive.zip/flag.txt PKZIP Encr: TS_chk, cmplen=173, decmplen=161, crc=0000 ts=8A75 cs=8a75 type=8
    
    
    
    $ cat archive.hash
    secret_archive.zip/flag.txt:$zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...[hash data]...*$/zip2$:flag.txt:secret_archive.zip::

That hash string is what John will work with. The `$zip2$` prefix tells John this is an AES-encrypted ZIP. The fields encode the salt, key length, verification data, and encrypted content sample.

### Step 2: Build a Targeted Wordlist

Before throwing rockyou.txt at it (which would take hours on CPU), I made a small targeted wordlist from everything I knew about the challenge:
    
    
    $ cat > custom.txt << 'EOF'
    archive
    madness
    secret
    archive_madness
    picoctf
    pico
    ctf
    flag
    secret_archive
    archivemadness
    picoCTF2024
    picoCTF2025
    zip
    password
    EOF

### Step 3: Run John with the Custom Wordlist First
    
    
    $ john --wordlist=custom.txt archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    0g 0:00:00:00 DONE (2024-11-14 14:23:10) 0g/s 933.3p/s 933.3p/s 933.3p/s ..
    Session completed.

No hit. The obvious guesses weren't it. Time for John's mangling rules:
    
    
    $ john --wordlist=custom.txt --rules=Single archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    archive2024       (secret_archive.zip/flag.txt)
    1g 0:00:00:03 DONE (2024-11-14 14:23:47) 0.3030g/s 8381p/s 8381p/s 8381p/s ..
    Use the "--show" option to display all cracked passwords reliably
    Session completed.

There it was. `archive2024`. John's Single rule set had appended "2024" to "archive" — a common CTF password construction pattern. The whole crack took about three seconds once I had the right wordlist.

### Step 4: Verify with --show and Extract the Flag
    
    
    $ john --show archive.hash
    secret_archive.zip/flag.txt:archive2024:flag.txt:secret_archive.zip::
    
    1 password hash cracked, 0 left
    
    
    $ unzip -P archive2024 secret_archive.zip
    Archive:  secret_archive.zip
      inflating: flag.txt
    
    $ cat flag.txt
    picoCTF{z1p_crypt0_1s_n0t_s3cur3_w1th_w34k_p4ssw0rds}

Flag captured. 200 points. I sat there for a moment just staring at the terminal.

## The Moment It Clicked

The real turning point wasn't finding the password. It was going back to that hexdump and finally paying attention to the version byte I'd noticed and ignored. Version `0x33` (decimal 51) in a ZIP local file header is documented in the ZIP specification as indicating AES encryption. I'd seen the number. I hadn't bothered to look up what it meant.

Once I ran `zipinfo -v` and saw "AES Encryption / 256-bit" explicitly printed, everything reorganized in my head. `fcrackzip` failing silently made sense. The need for `zip2john` specifically made sense. The whole challenge was designed around knowing that distinction.

That's the thing about CTF forensics challenges — the file format usually tells you exactly what you need to know. You just have to actually read it.

## Full Trial Process: Every Step I Took

Step| Action| Command| Result| Why it failed / succeeded  
---|---|---|---|---  
1| Identify file type| `file secret_archive.zip`| ZIP archive, v2.0+| Succeeded — confirmed it's a valid ZIP  
2| Attempt extraction| `unzip secret_archive.zip`| Password prompt, then failure| Expected — file is encrypted  
3| Manual password guessing| `unzip -P <word> secret_archive.zip`| 15 attempts, all failed| Failed — password not an obvious keyword  
4| Check metadata| `strings secret_archive.zip`| Only filename visible| Failed — AES content is opaque to strings  
5| Binary inspection| `hexdump -C secret_archive.zip`| Saw version 0x33 byte| Partial — noticed AES indicator but didn't act on it  
6| fcrackzip dictionary attack| `fcrackzip -u -D -p rockyou.txt ...`| Silent, no output| Failed — fcrackzip doesn't support AES-encrypted ZIPs  
7| Confirm encryption type| `zipinfo -v secret_archive.zip`| AES-256 confirmed| Succeeded — this was the key diagnostic step  
8| Extract hash with zip2john| `zip2john secret_archive.zip > archive.hash`| $zip2$ hash generated| Succeeded — correct tool for AES ZIP  
9| John with custom wordlist| `john --wordlist=custom.txt archive.hash`| 0 hits| Failed — exact words not the password  
10| John with Single rules| `john --wordlist=custom.txt --rules=Single archive.hash`| archive2024 cracked in 3s| Succeeded — rule mangling hit the pattern  
11| Extract flag| `unzip -P archive2024 secret_archive.zip`| flag.txt extracted| Succeeded — password confirmed  
  
## Technical Background: Why AES-256 ZIP Is Strong But Passwords Aren't

AES-256 is genuinely secure encryption. The key itself — 256 bits of random data — would take longer than the age of the universe to brute force with any foreseeable hardware. The weakness isn't the algorithm. It's the password.

ZIP's AES implementation (the WinZip AE format) derives the actual encryption key from the password using PBKDF2-HMAC-SHA1 with 1000 iterations. This is deliberately slow to make brute forcing expensive — but 1000 iterations is not especially slow by modern standards. On a modern CPU you can test roughly 50,000–100,000 passwords per second. On a GPU with hashcat and the right module (`-m 13600`), that number climbs to millions per second.

The practical consequence: an AES-256 ZIP is only as secure as its password. "archive2024" is 11 characters with a simple pattern — a dictionary word plus a year. In real-world password auditing, that's classified as weak. It would fall to any competent dictionary attack within minutes even without knowing the target context.

### ZipCrypto vs. AES: Why the Distinction Matters

The older ZipCrypto (traditional PKZIP) encryption is even weaker for a different reason. It has a known-plaintext attack: if you know any 12 bytes of the plaintext (which is trivially true for ZIP files, since the local file header structure is predictable), tools like `pkcrack` can recover the encryption keys directly without knowing the password at all. This attack works regardless of password complexity.

When `zipinfo -v` shows "ZipCrypto" instead of "AES Encryption," your first move should be checking if a plaintext attack is viable — not running john at all. That's a completely different solve path.

### Real-World Context: Where This Shows Up Outside CTF

Password-protected ZIP files appear constantly in malware delivery — phishing emails often include encrypted ZIPs specifically because the password (included in the email body) prevents automated scanning tools from inspecting the payload. Security researchers and incident responders use exactly these tools — `zip2john`, john, hashcat — to recover credentials from seized devices and investigate malware samples. The CTF skill and the professional skill are the same skill.

There are also legitimate recovery scenarios: encrypted archives from former employees, old backup files with forgotten passwords, forensic examination of devices where the user is unavailable. The legal and ethical boundaries matter here — using these tools against files you don't have authorization to access is a different matter entirely.

## zip2john vs. hashcat: When to Use Which

`zip2john` \+ john is the right default when you're on a CPU-only system or when you expect a small, targeted wordlist to work quickly. John's rule sets (especially Single and Jumbo) are excellent at generating contextually plausible password variations from a seed wordlist — exactly what you want in CTF where the challenge setter chose a password related to the theme.

hashcat with `-m 13600` (WinZip) is the right choice when you need raw throughput — exhaustive wordlist runs against rockyou.txt, combinatory attacks, or mask attacks for known-length passwords with unknown character sets. GPU acceleration makes a substantial difference here, but you need to extract the hash first with `zip2john` in either case.
    
    
    # hashcat equivalent (after extracting hash with zip2john)
    $ hashcat -m 13600 -a 0 archive.hash /usr/share/wordlists/rockyou.txt
    hashcat (v6.2.6) starting...
    
    OpenCL API (OpenCL 3.0 PoCL 3.1+debian) - Platform #1 [The pocl project]
    ==========================================================================
    * Device #1: pthread-Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz, 13579/27222 MB (4096 MB allocatable), 4MCU
    
    Minimum password length supported by kernel: 0
    Maximum password length supported by kernel: 256
    
    Hashes: 1 digests; 1 unique digests, 1 unique salts
    ...
    Session..........: hashcat
    Status...........: Cracked
    Hash.Mode........: 13600 (WinZip)
    Hash.Target......: $zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...
    Time.Started.....: Thu Nov 14 14:35:02 2024 (1 min, 43 secs)
    Candidates.#1....: archive2024
    
    Started: Thu Nov 14 14:35:02 2024
    Stopped: Thu Nov 14 14:36:45 2024

## What I'd Do Differently Next Time

The biggest time sink wasn't the failed commands — it was not confirming the encryption type before choosing tools. `zipinfo -v` takes two seconds. If I'd run that before anything else, I would have skipped the entire `fcrackzip` detour and gone straight to `zip2john`.

My updated mental checklist for any encrypted ZIP in CTF:

  1. Run `zipinfo -v` immediately. Identify ZipCrypto vs. AES, and note the AES key size.
  2. If ZipCrypto: consider known-plaintext attack with pkcrack before any password cracking.
  3. If AES: run `zip2john` to extract the hash. Do not touch fcrackzip.
  4. Build a targeted wordlist from every word in the challenge title, description, and filename. Five minutes here saves an hour later.
  5. Run John with that wordlist first (`--rules=Single`). If no hit within 30 seconds, try `--rules=Jumbo`.
  6. If still stuck: rockyou.txt without rules. Then rockyou.txt with rules. Then reconsider whether password cracking is actually the path.
  7. Cap the entire attempt at 15 minutes before stepping back and asking if there's a different angle entirely.



The time limit matters. CTF challenges are designed to be solvable. If you've been running wordlists for 30 minutes with no result, the answer probably isn't "try a bigger wordlist." It's more likely that you're missing something about the challenge design.

## Quick Reference: zip2john Commands
    
    
    # Check encryption type first
    zipinfo -v target.zip
    
    # Extract hash
    zip2john target.zip > target.hash
    
    # Crack with custom wordlist + rules
    john --wordlist=custom.txt --rules=Single target.hash
    
    # Crack with rockyou
    john --wordlist=/usr/share/wordlists/rockyou.txt target.hash
    
    # Show cracked password
    john --show target.hash
    
    # hashcat equivalent (GPU)
    hashcat -m 13600 -a 0 target.hash /usr/share/wordlists/rockyou.txt
    
    # Extract with recovered password
    unzip -P <password> target.zip

## Final Thoughts

picoCTF's _archive_madness_ is a straightforward challenge once you know what you're doing — but I didn't know what I was doing for the first 20 minutes of it. The failure wasn't lack of knowledge about `zip2john`. It was not checking the basic file properties before reaching for tools.

The broader lesson, which I keep relearning in different forms: read the file before attacking it. ZIP headers, ELF headers, PNG chunks, PDF structure — they all tell you what the file actually is and what's relevant about it. Starting with `file` and then a format-specific inspector is worth more than jumping straight to attack tools.

If you found this writeup through a search for zip2john, I hope the failure log was as useful as the working commands. The 20 wasted minutes are the part worth reading.
    
    
    $ strings secret_archive.zip
    PK
    flag.txt
    

Just the filename and the PK magic bytes. Of course. The metadata gives you the filename and some structural information, but the password isn't stored there in any form a human can read. I knew this intellectually. I checked anyway. That's what CTF desperation looks like.

### Mistake #3: hexdump Without a Plan

After `strings` failed, I opened the file in hexdump. I was looking for... something. I'm not entirely sure what. A cleartext hint? A comment field? I scrolled through hex output for about five minutes before I admitted to myself I had no idea what I was looking for.
    
    
    $ hexdump -C secret_archive.zip | head -20
    00000000  50 4b 03 04 33 00 01 00  63 00 00 00 21 00 75 8a  |PK..3...c...!.u.|
    00000010  a4 6e 00 00 00 00 00 00  00 00 07 00 1c 00 66 6c  |.n............fl|
    00000020  61 67 2e 74 78 74 55 54  09 00 03 5e 2f 84 65 5f  |ag.txtUT...^/.e_|
    00000030  2f 84 65 55 78 0b 00 01  04 e8 03 00 00 04 e8 03  |/.eUx...........|
    00000040  00 00 01 99 07 00 51 4d  41 45 0c 00 19 18 c9 41  |......QMAE.....A|
    00000050  c0 28 4c 57 a2 fe 5c 9e  9a 4d a0 de b8 68 74 61  |.(LW..\..M...hta|

I can read a ZIP header. The `PK\x03\x04` signature, the version needed (`0x33` = 51, meaning AES encryption), the general purpose bit flag (`0x01` = encrypted). That version byte should have told me something important. I noticed it. I didn't act on it. That was the real mistake.

## The Rabbit Hole: fcrackzip and Why It Silently Failed

Around the 20-minute mark I remembered `fcrackzip` existed. I'd used it once before on a simpler challenge with ZipCrypto encryption. I figured it would work here too. This was the longest detour of the whole session — easily 15 minutes of confusion before I understood the problem.
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt secret_archive.zip
    

It ran. And ran. And ran. No output. No progress indicator. After five minutes I hit Ctrl+C and tried a smaller wordlist:
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    

Still nothing. I checked if the file was corrupted. I re-downloaded it. Same result. I tried the verbose flag:
    
    
    $ fcrackzip -v -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    found file 'flag.txt', (size cp/uc    173/   161, flags 1, chk 8a75)
    

It found the file. It attempted passwords. It never cracked it. This was the moment I finally went back to that `0x33` version byte from the hexdump.

Version 51 in a ZIP local file header means AES encryption — specifically, it means the file uses WinZip's AE-1 or AE-2 AES extension. `fcrackzip` doesn't support AES-encrypted ZIPs. It works fine against the older ZipCrypto (traditional PKZIP encryption), but AES is a different animal entirely. The tool was silently skipping every candidate because it couldn't even verify them correctly. I'd been pouring water into a bucket with no bottom.
    
    
    $ zipinfo -v secret_archive.zip
    Archive:  secret_archive.zip
    There is no zipfile comment.
    
    End-of-central-directory record:
    -------------------------------
    ...
    Central directory entry #1:
    ---------------------------
      flag.txt
      ...
      file security status:                 encrypted
      extended local header:                no
      file last modified on (DOS date/time):  ...
      32-bit CRC value (hex):               00000000
      compressed size:                      173 bytes
      uncompressed size:                    161 bytes
      length of filename:                   8 characters
      ...
      encryption method:                    AES Encryption
      AES key size:                         256-bit

AES-256. That explained everything. `fcrackzip` was never going to work. I needed a tool that understood this encryption format — and that's exactly what `zip2john` is built for.

## What zip2john Actually Does (And Why It Matters)

Before I show the commands that worked, it's worth understanding the mechanism. `zip2john` is part of the John the Ripper suite. Its job is not to crack passwords — it's to extract a crackable hash from the ZIP file's encryption metadata.

For AES-encrypted ZIPs (the WinZip AE format), the ZIP file stores a salt value and a verification value in the local file header. These are derived from the password using PBKDF2-HMAC-SHA1. `zip2john` reads these values and formats them into a hash string that John the Ripper (or hashcat, with the right module) can work with. The hash includes the salt, the number of iterations, and the verification bytes — everything needed to test a candidate password without actually decrypting the file contents.

This matters for a non-obvious reason: AES-256 ZIP encryption is actually strong. Unlike the old ZipCrypto (which has known plaintext attacks that can bypass brute force entirely), AES-256 ZIP has no practical cryptographic weakness. You're doing a genuine dictionary or brute force attack. That means wordlist quality is everything.

## The Commands That Actually Worked

### Step 1: Extract the Hash
    
    
    $ zip2john secret_archive.zip > archive.hash
    ver 2.0 efh 5455 efh 7875 secret_archive.zip/flag.txt PKZIP Encr: TS_chk, cmplen=173, decmplen=161, crc=0000 ts=8A75 cs=8a75 type=8
    
    
    
    $ cat archive.hash
    secret_archive.zip/flag.txt:$zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...[hash data]...*$/zip2$:flag.txt:secret_archive.zip::

That hash string is what John will work with. The `$zip2$` prefix tells John this is an AES-encrypted ZIP. The fields encode the salt, key length, verification data, and encrypted content sample.

### Step 2: Build a Targeted Wordlist

Before throwing rockyou.txt at it (which would take hours on CPU), I made a small targeted wordlist from everything I knew about the challenge:
    
    
    $ cat > custom.txt << 'EOF'
    archive
    madness
    secret
    archive_madness
    picoctf
    pico
    ctf
    flag
    secret_archive
    archivemadness
    picoCTF2024
    picoCTF2025
    zip
    password
    EOF

### Step 3: Run John with the Custom Wordlist First
    
    
    $ john --wordlist=custom.txt archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    0g 0:00:00:00 DONE (2024-11-14 14:23:10) 0g/s 933.3p/s 933.3p/s 933.3p/s ..
    Session completed.

No hit. The obvious guesses weren't it. Time for John's mangling rules:
    
    
    $ john --wordlist=custom.txt --rules=Single archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    archive2024       (secret_archive.zip/flag.txt)
    1g 0:00:00:03 DONE (2024-11-14 14:23:47) 0.3030g/s 8381p/s 8381p/s 8381p/s ..
    Use the "--show" option to display all cracked passwords reliably
    Session completed.

There it was. `archive2024`. John's Single rule set had appended "2024" to "archive" — a common CTF password construction pattern. The whole crack took about three seconds once I had the right wordlist.

### Step 4: Verify with --show and Extract the Flag
    
    
    $ john --show archive.hash
    secret_archive.zip/flag.txt:archive2024:flag.txt:secret_archive.zip::
    
    1 password hash cracked, 0 left
    
    
    $ unzip -P archive2024 secret_archive.zip
    Archive:  secret_archive.zip
      inflating: flag.txt
    
    $ cat flag.txt
    picoCTF{z1p_crypt0_1s_n0t_s3cur3_w1th_w34k_p4ssw0rds}

Flag captured. 200 points. I sat there for a moment just staring at the terminal.

## The Moment It Clicked

The real turning point wasn't finding the password. It was going back to that hexdump and finally paying attention to the version byte I'd noticed and ignored. Version `0x33` (decimal 51) in a ZIP local file header is documented in the ZIP specification as indicating AES encryption. I'd seen the number. I hadn't bothered to look up what it meant.

Once I ran `zipinfo -v` and saw "AES Encryption / 256-bit" explicitly printed, everything reorganized in my head. `fcrackzip` failing silently made sense. The need for `zip2john` specifically made sense. The whole challenge was designed around knowing that distinction.

That's the thing about CTF forensics challenges — the file format usually tells you exactly what you need to know. You just have to actually read it.

## Full Trial Process: Every Step I Took

Step| Action| Command| Result| Why it failed / succeeded  
---|---|---|---|---  
1| Identify file type| `file secret_archive.zip`| ZIP archive, v2.0+| Succeeded — confirmed it's a valid ZIP  
2| Attempt extraction| `unzip secret_archive.zip`| Password prompt, then failure| Expected — file is encrypted  
3| Manual password guessing| `unzip -P <word> secret_archive.zip`| 15 attempts, all failed| Failed — password not an obvious keyword  
4| Check metadata| `strings secret_archive.zip`| Only filename visible| Failed — AES content is opaque to strings  
5| Binary inspection| `hexdump -C secret_archive.zip`| Saw version 0x33 byte| Partial — noticed AES indicator but didn't act on it  
6| fcrackzip dictionary attack| `fcrackzip -u -D -p rockyou.txt ...`| Silent, no output| Failed — fcrackzip doesn't support AES-encrypted ZIPs  
7| Confirm encryption type| `zipinfo -v secret_archive.zip`| AES-256 confirmed| Succeeded — this was the key diagnostic step  
8| Extract hash with zip2john| `zip2john secret_archive.zip > archive.hash`| $zip2$ hash generated| Succeeded — correct tool for AES ZIP  
9| John with custom wordlist| `john --wordlist=custom.txt archive.hash`| 0 hits| Failed — exact words not the password  
10| John with Single rules| `john --wordlist=custom.txt --rules=Single archive.hash`| archive2024 cracked in 3s| Succeeded — rule mangling hit the pattern  
11| Extract flag| `unzip -P archive2024 secret_archive.zip`| flag.txt extracted| Succeeded — password confirmed  
  
## Technical Background: Why AES-256 ZIP Is Strong But Passwords Aren't

AES-256 is genuinely secure encryption. The key itself — 256 bits of random data — would take longer than the age of the universe to brute force with any foreseeable hardware. The weakness isn't the algorithm. It's the password.

ZIP's AES implementation (the WinZip AE format) derives the actual encryption key from the password using PBKDF2-HMAC-SHA1 with 1000 iterations. This is deliberately slow to make brute forcing expensive — but 1000 iterations is not especially slow by modern standards. On a modern CPU you can test roughly 50,000–100,000 passwords per second. On a GPU with hashcat and the right module (`-m 13600`), that number climbs to millions per second.

The practical consequence: an AES-256 ZIP is only as secure as its password. "archive2024" is 11 characters with a simple pattern — a dictionary word plus a year. In real-world password auditing, that's classified as weak. It would fall to any competent dictionary attack within minutes even without knowing the target context.

### ZipCrypto vs. AES: Why the Distinction Matters

The older ZipCrypto (traditional PKZIP) encryption is even weaker for a different reason. It has a known-plaintext attack: if you know any 12 bytes of the plaintext (which is trivially true for ZIP files, since the local file header structure is predictable), tools like `pkcrack` can recover the encryption keys directly without knowing the password at all. This attack works regardless of password complexity.

When `zipinfo -v` shows "ZipCrypto" instead of "AES Encryption," your first move should be checking if a plaintext attack is viable — not running john at all. That's a completely different solve path.

### Real-World Context: Where This Shows Up Outside CTF

Password-protected ZIP files appear constantly in malware delivery — phishing emails often include encrypted ZIPs specifically because the password (included in the email body) prevents automated scanning tools from inspecting the payload. Security researchers and incident responders use exactly these tools — `zip2john`, john, hashcat — to recover credentials from seized devices and investigate malware samples. The CTF skill and the professional skill are the same skill.

There are also legitimate recovery scenarios: encrypted archives from former employees, old backup files with forgotten passwords, forensic examination of devices where the user is unavailable. The legal and ethical boundaries matter here — using these tools against files you don't have authorization to access is a different matter entirely.

## zip2john vs. hashcat: When to Use Which

`zip2john` \+ john is the right default when you're on a CPU-only system or when you expect a small, targeted wordlist to work quickly. John's rule sets (especially Single and Jumbo) are excellent at generating contextually plausible password variations from a seed wordlist — exactly what you want in CTF where the challenge setter chose a password related to the theme.

hashcat with `-m 13600` (WinZip) is the right choice when you need raw throughput — exhaustive wordlist runs against rockyou.txt, combinatory attacks, or mask attacks for known-length passwords with unknown character sets. GPU acceleration makes a substantial difference here, but you need to extract the hash first with `zip2john` in either case.
    
    
    # hashcat equivalent (after extracting hash with zip2john)
    $ hashcat -m 13600 -a 0 archive.hash /usr/share/wordlists/rockyou.txt
    hashcat (v6.2.6) starting...
    
    OpenCL API (OpenCL 3.0 PoCL 3.1+debian) - Platform #1 [The pocl project]
    ==========================================================================
    * Device #1: pthread-Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz, 13579/27222 MB (4096 MB allocatable), 4MCU
    
    Minimum password length supported by kernel: 0
    Maximum password length supported by kernel: 256
    
    Hashes: 1 digests; 1 unique digests, 1 unique salts
    ...
    Session..........: hashcat
    Status...........: Cracked
    Hash.Mode........: 13600 (WinZip)
    Hash.Target......: $zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...
    Time.Started.....: Thu Nov 14 14:35:02 2024 (1 min, 43 secs)
    Candidates.#1....: archive2024
    
    Started: Thu Nov 14 14:35:02 2024
    Stopped: Thu Nov 14 14:36:45 2024

## What I'd Do Differently Next Time

The biggest time sink wasn't the failed commands — it was not confirming the encryption type before choosing tools. `zipinfo -v` takes two seconds. If I'd run that before anything else, I would have skipped the entire `fcrackzip` detour and gone straight to `zip2john`.

My updated mental checklist for any encrypted ZIP in CTF:

  1. Run `zipinfo -v` immediately. Identify ZipCrypto vs. AES, and note the AES key size.
  2. If ZipCrypto: consider known-plaintext attack with pkcrack before any password cracking.
  3. If AES: run `zip2john` to extract the hash. Do not touch fcrackzip.
  4. Build a targeted wordlist from every word in the challenge title, description, and filename. Five minutes here saves an hour later.
  5. Run John with that wordlist first (`--rules=Single`). If no hit within 30 seconds, try `--rules=Jumbo`.
  6. If still stuck: rockyou.txt without rules. Then rockyou.txt with rules. Then reconsider whether password cracking is actually the path.
  7. Cap the entire attempt at 15 minutes before stepping back and asking if there's a different angle entirely.



The time limit matters. CTF challenges are designed to be solvable. If you've been running wordlists for 30 minutes with no result, the answer probably isn't "try a bigger wordlist." It's more likely that you're missing something about the challenge design.

## Quick Reference: zip2john Commands
    
    
    # Check encryption type first
    zipinfo -v target.zip
    
    # Extract hash
    zip2john target.zip > target.hash
    
    # Crack with custom wordlist + rules
    john --wordlist=custom.txt --rules=Single target.hash
    
    # Crack with rockyou
    john --wordlist=/usr/share/wordlists/rockyou.txt target.hash
    
    # Show cracked password
    john --show target.hash
    
    # hashcat equivalent (GPU)
    hashcat -m 13600 -a 0 target.hash /usr/share/wordlists/rockyou.txt
    
    # Extract with recovered password
    unzip -P <password> target.zip

## Final Thoughts

picoCTF's _archive_madness_ is a straightforward challenge once you know what you're doing — but I didn't know what I was doing for the first 20 minutes of it. The failure wasn't lack of knowledge about `zip2john`. It was not checking the basic file properties before reaching for tools.

The broader lesson, which I keep relearning in different forms: read the file before attacking it. ZIP headers, ELF headers, PNG chunks, PDF structure — they all tell you what the file actually is and what's relevant about it. Starting with `file` and then a format-specific inspector is worth more than jumping straight to attack tools.

If you found this writeup through a search for zip2john, I hope the failure log was as useful as the working commands. The 20 wasted minutes are the part worth reading.

# zip2john in CTF: How I Finally Cracked an AES-256 Encrypted ZIP in picoCTF

I almost skipped this challenge entirely. A 200-point forensics problem in picoCTF labeled **archive_madness** — just a ZIP file, no hints, no README, nothing but `secret_archive.zip` staring back at me. I figured someone smarter would grab the points while I moved on to something I actually understood. I'm glad I didn't walk away.

This writeup covers every wrong turn I took, the 20-odd minutes I burned on approaches that had no business working, and the exact moment I understood what `zip2john` is actually doing — not just the commands, but the underlying structure it's reading. If you've stared at a password-protected ZIP and wondered where to even begin, this is for you.

## The Challenge: picoCTF archive_madness

The challenge description read: _"We found this archive in a suspicious place. Can you get what's inside? File: secret_archive.zip"_

Category: Forensics. Points: 200. That's medium difficulty in picoCTF terms — not trivial, but not the kind of thing that requires a novel exploit. The ZIP file downloaded cleanly. I ran `file` on it immediately:
    
    
    $ file secret_archive.zip
    secret_archive.zip: Zip archive data, at least v2.0 to extract, one file

Standard ZIP. Nothing exotic. I tried to unzip it:
    
    
    $ unzip secret_archive.zip
    Archive:  secret_archive.zip
    [secret_archive.zip] flag.txt password:
       skipping: flag.txt                incorrect password

Password protected. One file inside: `flag.txt`. This was going to be a password cracking challenge. Fine. I thought I knew exactly what to do.

## Why I Chose the Wrong Approach First (And Wasted 20 Minutes)

My reasoning at the time felt completely logical. The challenge was called _archive_madness_ and the file was named `secret_archive.zip`. In CTF, password hints often hide in plain sight — in the filename, the challenge title, the description. I'd seen that pattern before. So I spent the first chunk of time doing exactly the wrong things for what felt like exactly the right reasons.

### Mistake #1: Manual Password Guessing

I started typing passwords by hand using `unzip -P`. "archive", "madness", "secret", "flag", "pico", "picoctf", "password", "zip" — I tried about fifteen variations. Each one returned the same thing:
    
    
    $ unzip -P archive secret_archive.zip
    Archive:  secret_archive.zip
       skipping: flag.txt                incorrect password
    
    $ unzip -P picoctf secret_archive.zip
    Archive:  secret_archive.zip
       skipping: flag.txt                incorrect password
    
    $ unzip -P secret secret_archive.zip
    Archive:  secret_archive.zip
       skipping: flag.txt                incorrect password

Seven minutes gone. Nothing to show for it except slightly bruised confidence.

### Mistake #2: Running strings on an Encrypted File

Then I made the mistake that anyone who's done more stego than crypto tends to make: I reached for `strings`. The logic was: maybe the password is embedded somewhere unencrypted in the ZIP metadata. ZIP files have a local file header and a central directory — those aren't encrypted even when the content is. Worth checking, right?
    
    
    $ strings secret_archive.zip
    PK
    flag.txt
    

Just the filename and the PK magic bytes. Of course. The metadata gives you the filename and some structural information, but the password isn't stored there in any form a human can read. I knew this intellectually. I checked anyway. That's what CTF desperation looks like.

### Mistake #3: hexdump Without a Plan

After `strings` failed, I opened the file in hexdump. I was looking for... something. I'm not entirely sure what. A cleartext hint? A comment field? I scrolled through hex output for about five minutes before I admitted to myself I had no idea what I was looking for.
    
    
    $ hexdump -C secret_archive.zip | head -20
    00000000  50 4b 03 04 33 00 01 00  63 00 00 00 21 00 75 8a  |PK..3...c...!.u.|
    00000010  a4 6e 00 00 00 00 00 00  00 00 07 00 1c 00 66 6c  |.n............fl|
    00000020  61 67 2e 74 78 74 55 54  09 00 03 5e 2f 84 65 5f  |ag.txtUT...^/.e_|
    00000030  2f 84 65 55 78 0b 00 01  04 e8 03 00 00 04 e8 03  |/.eUx...........|
    00000040  00 00 01 99 07 00 51 4d  41 45 0c 00 19 18 c9 41  |......QMAE.....A|
    00000050  c0 28 4c 57 a2 fe 5c 9e  9a 4d a0 de b8 68 74 61  |.(LW..\..M...hta|

I can read a ZIP header. The `PK\x03\x04` signature, the version needed (`0x33` = 51, meaning AES encryption), the general purpose bit flag (`0x01` = encrypted). That version byte should have told me something important. I noticed it. I didn't act on it. That was the real mistake.

## The Rabbit Hole: fcrackzip and Why It Silently Failed

Around the 20-minute mark I remembered `fcrackzip` existed. I'd used it once before on a simpler challenge with ZipCrypto encryption. I figured it would work here too. This was the longest detour of the whole session — easily 15 minutes of confusion before I understood the problem.
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt secret_archive.zip
    

It ran. And ran. And ran. No output. No progress indicator. After five minutes I hit Ctrl+C and tried a smaller wordlist:
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    

Still nothing. I checked if the file was corrupted. I re-downloaded it. Same result. I tried the verbose flag:
    
    
    $ fcrackzip -v -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    found file 'flag.txt', (size cp/uc    173/   161, flags 1, chk 8a75)
    

It found the file. It attempted passwords. It never cracked it. This was the moment I finally went back to that `0x33` version byte from the hexdump.

Version 51 in a ZIP local file header means AES encryption — specifically, it means the file uses WinZip's AE-1 or AE-2 AES extension. `fcrackzip` doesn't support AES-encrypted ZIPs. It works fine against the older ZipCrypto (traditional PKZIP encryption), but AES is a different animal entirely. The tool was silently skipping every candidate because it couldn't even verify them correctly. I'd been pouring water into a bucket with no bottom.
    
    
    $ zipinfo -v secret_archive.zip
    Archive:  secret_archive.zip
    There is no zipfile comment.
    
    End-of-central-directory record:
    -------------------------------
    ...
    Central directory entry #1:
    ---------------------------
      flag.txt
      ...
      file security status:                 encrypted
      extended local header:                no
      file last modified on (DOS date/time):  ...
      32-bit CRC value (hex):               00000000
      compressed size:                      173 bytes
      uncompressed size:                    161 bytes
      length of filename:                   8 characters
      ...
      encryption method:                    AES Encryption
      AES key size:                         256-bit

AES-256. That explained everything. `fcrackzip` was never going to work. I needed a tool that understood this encryption format — and that's exactly what `zip2john` is built for.

## What zip2john Actually Does (And Why It Matters)

Before I show the commands that worked, it's worth understanding the mechanism. `zip2john` is part of the John the Ripper suite. Its job is not to crack passwords — it's to extract a crackable hash from the ZIP file's encryption metadata.

For AES-encrypted ZIPs (the WinZip AE format), the ZIP file stores a salt value and a verification value in the local file header. These are derived from the password using PBKDF2-HMAC-SHA1. `zip2john` reads these values and formats them into a hash string that John the Ripper (or hashcat, with the right module) can work with. The hash includes the salt, the number of iterations, and the verification bytes — everything needed to test a candidate password without actually decrypting the file contents.

This matters for a non-obvious reason: AES-256 ZIP encryption is actually strong. Unlike the old ZipCrypto (which has known plaintext attacks that can bypass brute force entirely), AES-256 ZIP has no practical cryptographic weakness. You're doing a genuine dictionary or brute force attack. That means wordlist quality is everything.

## The Commands That Actually Worked

### Step 1: Extract the Hash
    
    
    $ zip2john secret_archive.zip > archive.hash
    ver 2.0 efh 5455 efh 7875 secret_archive.zip/flag.txt PKZIP Encr: TS_chk, cmplen=173, decmplen=161, crc=0000 ts=8A75 cs=8a75 type=8
    
    
    
    $ cat archive.hash
    secret_archive.zip/flag.txt:$zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...[hash data]...*$/zip2$:flag.txt:secret_archive.zip::

That hash string is what John will work with. The `$zip2$` prefix tells John this is an AES-encrypted ZIP. The fields encode the salt, key length, verification data, and encrypted content sample.

### Step 2: Build a Targeted Wordlist

Before throwing rockyou.txt at it (which would take hours on CPU), I made a small targeted wordlist from everything I knew about the challenge:
    
    
    $ cat > custom.txt << 'EOF'
    archive
    madness
    secret
    archive_madness
    picoctf
    pico
    ctf
    flag
    secret_archive
    archivemadness
    picoCTF2024
    picoCTF2025
    zip
    password
    EOF

### Step 3: Run John with the Custom Wordlist First
    
    
    $ john --wordlist=custom.txt archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    0g 0:00:00:00 DONE (2024-11-14 14:23:10) 0g/s 933.3p/s 933.3p/s 933.3p/s ..
    Session completed.

No hit. The obvious guesses weren't it. Time for John's mangling rules:
    
    
    $ john --wordlist=custom.txt --rules=Single archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    archive2024       (secret_archive.zip/flag.txt)
    1g 0:00:00:03 DONE (2024-11-14 14:23:47) 0.3030g/s 8381p/s 8381p/s 8381p/s ..
    Use the "--show" option to display all cracked passwords reliably
    Session completed.

There it was. `archive2024`. John's Single rule set had appended "2024" to "archive" — a common CTF password construction pattern. The whole crack took about three seconds once I had the right wordlist.

### Step 4: Verify with --show and Extract the Flag
    
    
    $ john --show archive.hash
    secret_archive.zip/flag.txt:archive2024:flag.txt:secret_archive.zip::
    
    1 password hash cracked, 0 left
    
    
    $ unzip -P archive2024 secret_archive.zip
    Archive:  secret_archive.zip
      inflating: flag.txt
    
    $ cat flag.txt
    picoCTF{z1p_crypt0_1s_n0t_s3cur3_w1th_w34k_p4ssw0rds}

Flag captured. 200 points. I sat there for a moment just staring at the terminal.

## The Moment It Clicked

The real turning point wasn't finding the password. It was going back to that hexdump and finally paying attention to the version byte I'd noticed and ignored. Version `0x33` (decimal 51) in a ZIP local file header is documented in the ZIP specification as indicating AES encryption. I'd seen the number. I hadn't bothered to look up what it meant.

Once I ran `zipinfo -v` and saw "AES Encryption / 256-bit" explicitly printed, everything reorganized in my head. `fcrackzip` failing silently made sense. The need for `zip2john` specifically made sense. The whole challenge was designed around knowing that distinction.

That's the thing about CTF forensics challenges — the file format usually tells you exactly what you need to know. You just have to actually read it.

## Full Trial Process: Every Step I Took

Step| Action| Command| Result| Why it failed / succeeded  
---|---|---|---|---  
1| Identify file type| `file secret_archive.zip`| ZIP archive, v2.0+| Succeeded — confirmed it's a valid ZIP  
2| Attempt extraction| `unzip secret_archive.zip`| Password prompt, then failure| Expected — file is encrypted  
3| Manual password guessing| `unzip -P <word> secret_archive.zip`| 15 attempts, all failed| Failed — password not an obvious keyword  
4| Check metadata| `strings secret_archive.zip`| Only filename visible| Failed — AES content is opaque to strings  
5| Binary inspection| `hexdump -C secret_archive.zip`| Saw version 0x33 byte| Partial — noticed AES indicator but didn't act on it  
6| fcrackzip dictionary attack| `fcrackzip -u -D -p rockyou.txt ...`| Silent, no output| Failed — fcrackzip doesn't support AES-encrypted ZIPs  
7| Confirm encryption type| `zipinfo -v secret_archive.zip`| AES-256 confirmed| Succeeded — this was the key diagnostic step  
8| Extract hash with zip2john| `zip2john secret_archive.zip > archive.hash`| $zip2$ hash generated| Succeeded — correct tool for AES ZIP  
9| John with custom wordlist| `john --wordlist=custom.txt archive.hash`| 0 hits| Failed — exact words not the password  
10| John with Single rules| `john --wordlist=custom.txt --rules=Single archive.hash`| archive2024 cracked in 3s| Succeeded — rule mangling hit the pattern  
11| Extract flag| `unzip -P archive2024 secret_archive.zip`| flag.txt extracted| Succeeded — password confirmed  
  
## Technical Background: Why AES-256 ZIP Is Strong But Passwords Aren't

AES-256 is genuinely secure encryption. The key itself — 256 bits of random data — would take longer than the age of the universe to brute force with any foreseeable hardware. The weakness isn't the algorithm. It's the password.

ZIP's AES implementation (the WinZip AE format) derives the actual encryption key from the password using PBKDF2-HMAC-SHA1 with 1000 iterations. This is deliberately slow to make brute forcing expensive — but 1000 iterations is not especially slow by modern standards. On a modern CPU you can test roughly 50,000–100,000 passwords per second. On a GPU with hashcat and the right module (`-m 13600`), that number climbs to millions per second.

The practical consequence: an AES-256 ZIP is only as secure as its password. "archive2024" is 11 characters with a simple pattern — a dictionary word plus a year. In real-world password auditing, that's classified as weak. It would fall to any competent dictionary attack within minutes even without knowing the target context.

### ZipCrypto vs. AES: Why the Distinction Matters

The older ZipCrypto (traditional PKZIP) encryption is even weaker for a different reason. It has a known-plaintext attack: if you know any 12 bytes of the plaintext (which is trivially true for ZIP files, since the local file header structure is predictable), tools like `pkcrack` can recover the encryption keys directly without knowing the password at all. This attack works regardless of password complexity.

When `zipinfo -v` shows "ZipCrypto" instead of "AES Encryption," your first move should be checking if a plaintext attack is viable — not running john at all. That's a completely different solve path.

### Real-World Context: Where This Shows Up Outside CTF

Password-protected ZIP files appear constantly in malware delivery — phishing emails often include encrypted ZIPs specifically because the password (included in the email body) prevents automated scanning tools from inspecting the payload. Security researchers and incident responders use exactly these tools — `zip2john`, john, hashcat — to recover credentials from seized devices and investigate malware samples. The CTF skill and the professional skill are the same skill.

There are also legitimate recovery scenarios: encrypted archives from former employees, old backup files with forgotten passwords, forensic examination of devices where the user is unavailable. The legal and ethical boundaries matter here — using these tools against files you don't have authorization to access is a different matter entirely.

## zip2john vs. hashcat: When to Use Which

`zip2john` \+ john is the right default when you're on a CPU-only system or when you expect a small, targeted wordlist to work quickly. John's rule sets (especially Single and Jumbo) are excellent at generating contextually plausible password variations from a seed wordlist — exactly what you want in CTF where the challenge setter chose a password related to the theme.

hashcat with `-m 13600` (WinZip) is the right choice when you need raw throughput — exhaustive wordlist runs against rockyou.txt, combinatory attacks, or mask attacks for known-length passwords with unknown character sets. GPU acceleration makes a substantial difference here, but you need to extract the hash first with `zip2john` in either case.
    
    
    # hashcat equivalent (after extracting hash with zip2john)
    $ hashcat -m 13600 -a 0 archive.hash /usr/share/wordlists/rockyou.txt
    hashcat (v6.2.6) starting...
    
    OpenCL API (OpenCL 3.0 PoCL 3.1+debian) - Platform #1 [The pocl project]
    ==========================================================================
    * Device #1: pthread-Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz, 13579/27222 MB (4096 MB allocatable), 4MCU
    
    Minimum password length supported by kernel: 0
    Maximum password length supported by kernel: 256
    
    Hashes: 1 digests; 1 unique digests, 1 unique salts
    ...
    Session..........: hashcat
    Status...........: Cracked
    Hash.Mode........: 13600 (WinZip)
    Hash.Target......: $zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...
    Time.Started.....: Thu Nov 14 14:35:02 2024 (1 min, 43 secs)
    Candidates.#1....: archive2024
    
    Started: Thu Nov 14 14:35:02 2024
    Stopped: Thu Nov 14 14:36:45 2024

## What I'd Do Differently Next Time

The biggest time sink wasn't the failed commands — it was not confirming the encryption type before choosing tools. `zipinfo -v` takes two seconds. If I'd run that before anything else, I would have skipped the entire `fcrackzip` detour and gone straight to `zip2john`.

My updated mental checklist for any encrypted ZIP in CTF:

  1. Run `zipinfo -v` immediately. Identify ZipCrypto vs. AES, and note the AES key size.
  2. If ZipCrypto: consider known-plaintext attack with pkcrack before any password cracking.
  3. If AES: run `zip2john` to extract the hash. Do not touch fcrackzip.
  4. Build a targeted wordlist from every word in the challenge title, description, and filename. Five minutes here saves an hour later.
  5. Run John with that wordlist first (`--rules=Single`). If no hit within 30 seconds, try `--rules=Jumbo`.
  6. If still stuck: rockyou.txt without rules. Then rockyou.txt with rules. Then reconsider whether password cracking is actually the path.
  7. Cap the entire attempt at 15 minutes before stepping back and asking if there's a different angle entirely.



The time limit matters. CTF challenges are designed to be solvable. If you've been running wordlists for 30 minutes with no result, the answer probably isn't "try a bigger wordlist." It's more likely that you're missing something about the challenge design.

## Quick Reference: zip2john Commands
    
    
    # Check encryption type first
    zipinfo -v target.zip
    
    # Extract hash
    zip2john target.zip > target.hash
    
    # Crack with custom wordlist + rules
    john --wordlist=custom.txt --rules=Single target.hash
    
    # Crack with rockyou
    john --wordlist=/usr/share/wordlists/rockyou.txt target.hash
    
    # Show cracked password
    john --show target.hash
    
    # hashcat equivalent (GPU)
    hashcat -m 13600 -a 0 target.hash /usr/share/wordlists/rockyou.txt
    
    # Extract with recovered password
    unzip -P <password> target.zip

## Final Thoughts

picoCTF's _archive_madness_ is a straightforward challenge once you know what you're doing — but I didn't know what I was doing for the first 20 minutes of it. The failure wasn't lack of knowledge about `zip2john`. It was not checking the basic file properties before reaching for tools.

The broader lesson, which I keep relearning in different forms: read the file before attacking it. ZIP headers, ELF headers, PNG chunks, PDF structure — they all tell you what the file actually is and what's relevant about it. Starting with `file` and then a format-specific inspector is worth more than jumping straight to attack tools.

If you found this writeup through a search for zip2john, I hope the failure log was as useful as the working commands. The 20 wasted minutes are the part worth reading.
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt secret_archive.zip
    

It ran. And ran. And ran. No output. No progress indicator. After five minutes I hit Ctrl+C and tried a smaller wordlist:
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    

Still nothing. I checked if the file was corrupted. I re-downloaded it. Same result. I tried the verbose flag:
    
    
    $ fcrackzip -v -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    found file 'flag.txt', (size cp/uc    173/   161, flags 1, chk 8a75)
    

It found the file. It attempted passwords. It never cracked it. This was the moment I finally went back to that `0x33` version byte from the hexdump.

Version 51 in a ZIP local file header means AES encryption — specifically, it means the file uses WinZip's AE-1 or AE-2 AES extension. `fcrackzip` doesn't support AES-encrypted ZIPs. It works fine against the older ZipCrypto (traditional PKZIP encryption), but AES is a different animal entirely. The tool was silently skipping every candidate because it couldn't even verify them correctly. I'd been pouring water into a bucket with no bottom.
    
    
    $ zipinfo -v secret_archive.zip
    Archive:  secret_archive.zip
    There is no zipfile comment.
    
    End-of-central-directory record:
    -------------------------------
    ...
    Central directory entry #1:
    ---------------------------
      flag.txt
      ...
      file security status:                 encrypted
      extended local header:                no
      file last modified on (DOS date/time):  ...
      32-bit CRC value (hex):               00000000
      compressed size:                      173 bytes
      uncompressed size:                    161 bytes
      length of filename:                   8 characters
      ...
      encryption method:                    AES Encryption
      AES key size:                         256-bit

AES-256. That explained everything. `fcrackzip` was never going to work. I needed a tool that understood this encryption format — and that's exactly what `zip2john` is built for.

## What zip2john Actually Does (And Why It Matters)

Before I show the commands that worked, it's worth understanding the mechanism. `zip2john` is part of the John the Ripper suite. Its job is not to crack passwords — it's to extract a crackable hash from the ZIP file's encryption metadata.

For AES-encrypted ZIPs (the WinZip AE format), the ZIP file stores a salt value and a verification value in the local file header. These are derived from the password using PBKDF2-HMAC-SHA1. `zip2john` reads these values and formats them into a hash string that John the Ripper (or hashcat, with the right module) can work with. The hash includes the salt, the number of iterations, and the verification bytes — everything needed to test a candidate password without actually decrypting the file contents.

This matters for a non-obvious reason: AES-256 ZIP encryption is actually strong. Unlike the old ZipCrypto (which has known plaintext attacks that can bypass brute force entirely), AES-256 ZIP has no practical cryptographic weakness. You're doing a genuine dictionary or brute force attack. That means wordlist quality is everything.

## The Commands That Actually Worked

### Step 1: Extract the Hash
    
    
    $ zip2john secret_archive.zip > archive.hash
    ver 2.0 efh 5455 efh 7875 secret_archive.zip/flag.txt PKZIP Encr: TS_chk, cmplen=173, decmplen=161, crc=0000 ts=8A75 cs=8a75 type=8
    
    
    
    $ cat archive.hash
    secret_archive.zip/flag.txt:$zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...[hash data]...*$/zip2$:flag.txt:secret_archive.zip::

That hash string is what John will work with. The `$zip2$` prefix tells John this is an AES-encrypted ZIP. The fields encode the salt, key length, verification data, and encrypted content sample.

### Step 2: Build a Targeted Wordlist

Before throwing rockyou.txt at it (which would take hours on CPU), I made a small targeted wordlist from everything I knew about the challenge:
    
    
    $ cat > custom.txt << 'EOF'
    archive
    madness
    secret
    archive_madness
    picoctf
    pico
    ctf
    flag
    secret_archive
    archivemadness
    picoCTF2024
    picoCTF2025
    zip
    password
    EOF

### Step 3: Run John with the Custom Wordlist First
    
    
    $ john --wordlist=custom.txt archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    0g 0:00:00:00 DONE (2024-11-14 14:23:10) 0g/s 933.3p/s 933.3p/s 933.3p/s ..
    Session completed.

No hit. The obvious guesses weren't it. Time for John's mangling rules:
    
    
    $ john --wordlist=custom.txt --rules=Single archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    archive2024       (secret_archive.zip/flag.txt)
    1g 0:00:00:03 DONE (2024-11-14 14:23:47) 0.3030g/s 8381p/s 8381p/s 8381p/s ..
    Use the "--show" option to display all cracked passwords reliably
    Session completed.

There it was. `archive2024`. John's Single rule set had appended "2024" to "archive" — a common CTF password construction pattern. The whole crack took about three seconds once I had the right wordlist.

### Step 4: Verify with --show and Extract the Flag
    
    
    $ john --show archive.hash
    secret_archive.zip/flag.txt:archive2024:flag.txt:secret_archive.zip::
    
    1 password hash cracked, 0 left
    
    
    $ unzip -P archive2024 secret_archive.zip
    Archive:  secret_archive.zip
      inflating: flag.txt
    
    $ cat flag.txt
    picoCTF{z1p_crypt0_1s_n0t_s3cur3_w1th_w34k_p4ssw0rds}

Flag captured. 200 points. I sat there for a moment just staring at the terminal.

## The Moment It Clicked

The real turning point wasn't finding the password. It was going back to that hexdump and finally paying attention to the version byte I'd noticed and ignored. Version `0x33` (decimal 51) in a ZIP local file header is documented in the ZIP specification as indicating AES encryption. I'd seen the number. I hadn't bothered to look up what it meant.

Once I ran `zipinfo -v` and saw "AES Encryption / 256-bit" explicitly printed, everything reorganized in my head. `fcrackzip` failing silently made sense. The need for `zip2john` specifically made sense. The whole challenge was designed around knowing that distinction.

That's the thing about CTF forensics challenges — the file format usually tells you exactly what you need to know. You just have to actually read it.

## Full Trial Process: Every Step I Took

Step| Action| Command| Result| Why it failed / succeeded  
---|---|---|---|---  
1| Identify file type| `file secret_archive.zip`| ZIP archive, v2.0+| Succeeded — confirmed it's a valid ZIP  
2| Attempt extraction| `unzip secret_archive.zip`| Password prompt, then failure| Expected — file is encrypted  
3| Manual password guessing| `unzip -P <word> secret_archive.zip`| 15 attempts, all failed| Failed — password not an obvious keyword  
4| Check metadata| `strings secret_archive.zip`| Only filename visible| Failed — AES content is opaque to strings  
5| Binary inspection| `hexdump -C secret_archive.zip`| Saw version 0x33 byte| Partial — noticed AES indicator but didn't act on it  
6| fcrackzip dictionary attack| `fcrackzip -u -D -p rockyou.txt ...`| Silent, no output| Failed — fcrackzip doesn't support AES-encrypted ZIPs  
7| Confirm encryption type| `zipinfo -v secret_archive.zip`| AES-256 confirmed| Succeeded — this was the key diagnostic step  
8| Extract hash with zip2john| `zip2john secret_archive.zip > archive.hash`| $zip2$ hash generated| Succeeded — correct tool for AES ZIP  
9| John with custom wordlist| `john --wordlist=custom.txt archive.hash`| 0 hits| Failed — exact words not the password  
10| John with Single rules| `john --wordlist=custom.txt --rules=Single archive.hash`| archive2024 cracked in 3s| Succeeded — rule mangling hit the pattern  
11| Extract flag| `unzip -P archive2024 secret_archive.zip`| flag.txt extracted| Succeeded — password confirmed  
  
## Technical Background: Why AES-256 ZIP Is Strong But Passwords Aren't

AES-256 is genuinely secure encryption. The key itself — 256 bits of random data — would take longer than the age of the universe to brute force with any foreseeable hardware. The weakness isn't the algorithm. It's the password.

ZIP's AES implementation (the WinZip AE format) derives the actual encryption key from the password using PBKDF2-HMAC-SHA1 with 1000 iterations. This is deliberately slow to make brute forcing expensive — but 1000 iterations is not especially slow by modern standards. On a modern CPU you can test roughly 50,000–100,000 passwords per second. On a GPU with hashcat and the right module (`-m 13600`), that number climbs to millions per second.

The practical consequence: an AES-256 ZIP is only as secure as its password. "archive2024" is 11 characters with a simple pattern — a dictionary word plus a year. In real-world password auditing, that's classified as weak. It would fall to any competent dictionary attack within minutes even without knowing the target context.

### ZipCrypto vs. AES: Why the Distinction Matters

The older ZipCrypto (traditional PKZIP) encryption is even weaker for a different reason. It has a known-plaintext attack: if you know any 12 bytes of the plaintext (which is trivially true for ZIP files, since the local file header structure is predictable), tools like `pkcrack` can recover the encryption keys directly without knowing the password at all. This attack works regardless of password complexity.

When `zipinfo -v` shows "ZipCrypto" instead of "AES Encryption," your first move should be checking if a plaintext attack is viable — not running john at all. That's a completely different solve path.

### Real-World Context: Where This Shows Up Outside CTF

Password-protected ZIP files appear constantly in malware delivery — phishing emails often include encrypted ZIPs specifically because the password (included in the email body) prevents automated scanning tools from inspecting the payload. Security researchers and incident responders use exactly these tools — `zip2john`, john, hashcat — to recover credentials from seized devices and investigate malware samples. The CTF skill and the professional skill are the same skill.

There are also legitimate recovery scenarios: encrypted archives from former employees, old backup files with forgotten passwords, forensic examination of devices where the user is unavailable. The legal and ethical boundaries matter here — using these tools against files you don't have authorization to access is a different matter entirely.

## zip2john vs. hashcat: When to Use Which

`zip2john` \+ john is the right default when you're on a CPU-only system or when you expect a small, targeted wordlist to work quickly. John's rule sets (especially Single and Jumbo) are excellent at generating contextually plausible password variations from a seed wordlist — exactly what you want in CTF where the challenge setter chose a password related to the theme.

hashcat with `-m 13600` (WinZip) is the right choice when you need raw throughput — exhaustive wordlist runs against rockyou.txt, combinatory attacks, or mask attacks for known-length passwords with unknown character sets. GPU acceleration makes a substantial difference here, but you need to extract the hash first with `zip2john` in either case.
    
    
    # hashcat equivalent (after extracting hash with zip2john)
    $ hashcat -m 13600 -a 0 archive.hash /usr/share/wordlists/rockyou.txt
    hashcat (v6.2.6) starting...
    
    OpenCL API (OpenCL 3.0 PoCL 3.1+debian) - Platform #1 [The pocl project]
    ==========================================================================
    * Device #1: pthread-Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz, 13579/27222 MB (4096 MB allocatable), 4MCU
    
    Minimum password length supported by kernel: 0
    Maximum password length supported by kernel: 256
    
    Hashes: 1 digests; 1 unique digests, 1 unique salts
    ...
    Session..........: hashcat
    Status...........: Cracked
    Hash.Mode........: 13600 (WinZip)
    Hash.Target......: $zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...
    Time.Started.....: Thu Nov 14 14:35:02 2024 (1 min, 43 secs)
    Candidates.#1....: archive2024
    
    Started: Thu Nov 14 14:35:02 2024
    Stopped: Thu Nov 14 14:36:45 2024

## What I'd Do Differently Next Time

The biggest time sink wasn't the failed commands — it was not confirming the encryption type before choosing tools. `zipinfo -v` takes two seconds. If I'd run that before anything else, I would have skipped the entire `fcrackzip` detour and gone straight to `zip2john`.

My updated mental checklist for any encrypted ZIP in CTF:

  1. Run `zipinfo -v` immediately. Identify ZipCrypto vs. AES, and note the AES key size.
  2. If ZipCrypto: consider known-plaintext attack with pkcrack before any password cracking.
  3. If AES: run `zip2john` to extract the hash. Do not touch fcrackzip.
  4. Build a targeted wordlist from every word in the challenge title, description, and filename. Five minutes here saves an hour later.
  5. Run John with that wordlist first (`--rules=Single`). If no hit within 30 seconds, try `--rules=Jumbo`.
  6. If still stuck: rockyou.txt without rules. Then rockyou.txt with rules. Then reconsider whether password cracking is actually the path.
  7. Cap the entire attempt at 15 minutes before stepping back and asking if there's a different angle entirely.



The time limit matters. CTF challenges are designed to be solvable. If you've been running wordlists for 30 minutes with no result, the answer probably isn't "try a bigger wordlist." It's more likely that you're missing something about the challenge design.

## Quick Reference: zip2john Commands
    
    
    # Check encryption type first
    zipinfo -v target.zip
    
    # Extract hash
    zip2john target.zip > target.hash
    
    # Crack with custom wordlist + rules
    john --wordlist=custom.txt --rules=Single target.hash
    
    # Crack with rockyou
    john --wordlist=/usr/share/wordlists/rockyou.txt target.hash
    
    # Show cracked password
    john --show target.hash
    
    # hashcat equivalent (GPU)
    hashcat -m 13600 -a 0 target.hash /usr/share/wordlists/rockyou.txt
    
    # Extract with recovered password
    unzip -P <password> target.zip

## Final Thoughts

picoCTF's _archive_madness_ is a straightforward challenge once you know what you're doing — but I didn't know what I was doing for the first 20 minutes of it. The failure wasn't lack of knowledge about `zip2john`. It was not checking the basic file properties before reaching for tools.

The broader lesson, which I keep relearning in different forms: read the file before attacking it. ZIP headers, ELF headers, PNG chunks, PDF structure — they all tell you what the file actually is and what's relevant about it. Starting with `file` and then a format-specific inspector is worth more than jumping straight to attack tools.

If you found this writeup through a search for zip2john, I hope the failure log was as useful as the working commands. The 20 wasted minutes are the part worth reading.

# zip2john in CTF: How I Finally Cracked an AES-256 Encrypted ZIP in picoCTF

I almost skipped this challenge entirely. A 200-point forensics problem in picoCTF labeled **archive_madness** — just a ZIP file, no hints, no README, nothing but `secret_archive.zip` staring back at me. I figured someone smarter would grab the points while I moved on to something I actually understood. I'm glad I didn't walk away.

This writeup covers every wrong turn I took, the 20-odd minutes I burned on approaches that had no business working, and the exact moment I understood what `zip2john` is actually doing — not just the commands, but the underlying structure it's reading. If you've stared at a password-protected ZIP and wondered where to even begin, this is for you.

## The Challenge: picoCTF archive_madness

The challenge description read: _"We found this archive in a suspicious place. Can you get what's inside? File: secret_archive.zip"_

Category: Forensics. Points: 200. That's medium difficulty in picoCTF terms — not trivial, but not the kind of thing that requires a novel exploit. The ZIP file downloaded cleanly. I ran `file` on it immediately:
    
    
    $ file secret_archive.zip
    secret_archive.zip: Zip archive data, at least v2.0 to extract, one file

Standard ZIP. Nothing exotic. I tried to unzip it:
    
    
    $ unzip secret_archive.zip
    Archive:  secret_archive.zip
    [secret_archive.zip] flag.txt password:
       skipping: flag.txt                incorrect password

Password protected. One file inside: `flag.txt`. This was going to be a password cracking challenge. Fine. I thought I knew exactly what to do.

## Why I Chose the Wrong Approach First (And Wasted 20 Minutes)

My reasoning at the time felt completely logical. The challenge was called _archive_madness_ and the file was named `secret_archive.zip`. In CTF, password hints often hide in plain sight — in the filename, the challenge title, the description. I'd seen that pattern before. So I spent the first chunk of time doing exactly the wrong things for what felt like exactly the right reasons.

### Mistake #1: Manual Password Guessing

I started typing passwords by hand using `unzip -P`. "archive", "madness", "secret", "flag", "pico", "picoctf", "password", "zip" — I tried about fifteen variations. Each one returned the same thing:
    
    
    $ unzip -P archive secret_archive.zip
    Archive:  secret_archive.zip
       skipping: flag.txt                incorrect password
    
    $ unzip -P picoctf secret_archive.zip
    Archive:  secret_archive.zip
       skipping: flag.txt                incorrect password
    
    $ unzip -P secret secret_archive.zip
    Archive:  secret_archive.zip
       skipping: flag.txt                incorrect password

Seven minutes gone. Nothing to show for it except slightly bruised confidence.

### Mistake #2: Running strings on an Encrypted File

Then I made the mistake that anyone who's done more stego than crypto tends to make: I reached for `strings`. The logic was: maybe the password is embedded somewhere unencrypted in the ZIP metadata. ZIP files have a local file header and a central directory — those aren't encrypted even when the content is. Worth checking, right?
    
    
    $ strings secret_archive.zip
    PK
    flag.txt
    

Just the filename and the PK magic bytes. Of course. The metadata gives you the filename and some structural information, but the password isn't stored there in any form a human can read. I knew this intellectually. I checked anyway. That's what CTF desperation looks like.

### Mistake #3: hexdump Without a Plan

After `strings` failed, I opened the file in hexdump. I was looking for... something. I'm not entirely sure what. A cleartext hint? A comment field? I scrolled through hex output for about five minutes before I admitted to myself I had no idea what I was looking for.
    
    
    $ hexdump -C secret_archive.zip | head -20
    00000000  50 4b 03 04 33 00 01 00  63 00 00 00 21 00 75 8a  |PK..3...c...!.u.|
    00000010  a4 6e 00 00 00 00 00 00  00 00 07 00 1c 00 66 6c  |.n............fl|
    00000020  61 67 2e 74 78 74 55 54  09 00 03 5e 2f 84 65 5f  |ag.txtUT...^/.e_|
    00000030  2f 84 65 55 78 0b 00 01  04 e8 03 00 00 04 e8 03  |/.eUx...........|
    00000040  00 00 01 99 07 00 51 4d  41 45 0c 00 19 18 c9 41  |......QMAE.....A|
    00000050  c0 28 4c 57 a2 fe 5c 9e  9a 4d a0 de b8 68 74 61  |.(LW..\..M...hta|

I can read a ZIP header. The `PK\x03\x04` signature, the version needed (`0x33` = 51, meaning AES encryption), the general purpose bit flag (`0x01` = encrypted). That version byte should have told me something important. I noticed it. I didn't act on it. That was the real mistake.

## The Rabbit Hole: fcrackzip and Why It Silently Failed

Around the 20-minute mark I remembered `fcrackzip` existed. I'd used it once before on a simpler challenge with ZipCrypto encryption. I figured it would work here too. This was the longest detour of the whole session — easily 15 minutes of confusion before I understood the problem.
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt secret_archive.zip
    

It ran. And ran. And ran. No output. No progress indicator. After five minutes I hit Ctrl+C and tried a smaller wordlist:
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    

Still nothing. I checked if the file was corrupted. I re-downloaded it. Same result. I tried the verbose flag:
    
    
    $ fcrackzip -v -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    found file 'flag.txt', (size cp/uc    173/   161, flags 1, chk 8a75)
    

It found the file. It attempted passwords. It never cracked it. This was the moment I finally went back to that `0x33` version byte from the hexdump.

Version 51 in a ZIP local file header means AES encryption — specifically, it means the file uses WinZip's AE-1 or AE-2 AES extension. `fcrackzip` doesn't support AES-encrypted ZIPs. It works fine against the older ZipCrypto (traditional PKZIP encryption), but AES is a different animal entirely. The tool was silently skipping every candidate because it couldn't even verify them correctly. I'd been pouring water into a bucket with no bottom.
    
    
    $ zipinfo -v secret_archive.zip
    Archive:  secret_archive.zip
    There is no zipfile comment.
    
    End-of-central-directory record:
    -------------------------------
    ...
    Central directory entry #1:
    ---------------------------
      flag.txt
      ...
      file security status:                 encrypted
      extended local header:                no
      file last modified on (DOS date/time):  ...
      32-bit CRC value (hex):               00000000
      compressed size:                      173 bytes
      uncompressed size:                    161 bytes
      length of filename:                   8 characters
      ...
      encryption method:                    AES Encryption
      AES key size:                         256-bit

AES-256. That explained everything. `fcrackzip` was never going to work. I needed a tool that understood this encryption format — and that's exactly what `zip2john` is built for.

## What zip2john Actually Does (And Why It Matters)

Before I show the commands that worked, it's worth understanding the mechanism. `zip2john` is part of the John the Ripper suite. Its job is not to crack passwords — it's to extract a crackable hash from the ZIP file's encryption metadata.

For AES-encrypted ZIPs (the WinZip AE format), the ZIP file stores a salt value and a verification value in the local file header. These are derived from the password using PBKDF2-HMAC-SHA1. `zip2john` reads these values and formats them into a hash string that John the Ripper (or hashcat, with the right module) can work with. The hash includes the salt, the number of iterations, and the verification bytes — everything needed to test a candidate password without actually decrypting the file contents.

This matters for a non-obvious reason: AES-256 ZIP encryption is actually strong. Unlike the old ZipCrypto (which has known plaintext attacks that can bypass brute force entirely), AES-256 ZIP has no practical cryptographic weakness. You're doing a genuine dictionary or brute force attack. That means wordlist quality is everything.

## The Commands That Actually Worked

### Step 1: Extract the Hash
    
    
    $ zip2john secret_archive.zip > archive.hash
    ver 2.0 efh 5455 efh 7875 secret_archive.zip/flag.txt PKZIP Encr: TS_chk, cmplen=173, decmplen=161, crc=0000 ts=8A75 cs=8a75 type=8
    
    
    
    $ cat archive.hash
    secret_archive.zip/flag.txt:$zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...[hash data]...*$/zip2$:flag.txt:secret_archive.zip::

That hash string is what John will work with. The `$zip2$` prefix tells John this is an AES-encrypted ZIP. The fields encode the salt, key length, verification data, and encrypted content sample.

### Step 2: Build a Targeted Wordlist

Before throwing rockyou.txt at it (which would take hours on CPU), I made a small targeted wordlist from everything I knew about the challenge:
    
    
    $ cat > custom.txt << 'EOF'
    archive
    madness
    secret
    archive_madness
    picoctf
    pico
    ctf
    flag
    secret_archive
    archivemadness
    picoCTF2024
    picoCTF2025
    zip
    password
    EOF

### Step 3: Run John with the Custom Wordlist First
    
    
    $ john --wordlist=custom.txt archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    0g 0:00:00:00 DONE (2024-11-14 14:23:10) 0g/s 933.3p/s 933.3p/s 933.3p/s ..
    Session completed.

No hit. The obvious guesses weren't it. Time for John's mangling rules:
    
    
    $ john --wordlist=custom.txt --rules=Single archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    archive2024       (secret_archive.zip/flag.txt)
    1g 0:00:00:03 DONE (2024-11-14 14:23:47) 0.3030g/s 8381p/s 8381p/s 8381p/s ..
    Use the "--show" option to display all cracked passwords reliably
    Session completed.

There it was. `archive2024`. John's Single rule set had appended "2024" to "archive" — a common CTF password construction pattern. The whole crack took about three seconds once I had the right wordlist.

### Step 4: Verify with --show and Extract the Flag
    
    
    $ john --show archive.hash
    secret_archive.zip/flag.txt:archive2024:flag.txt:secret_archive.zip::
    
    1 password hash cracked, 0 left
    
    
    $ unzip -P archive2024 secret_archive.zip
    Archive:  secret_archive.zip
      inflating: flag.txt
    
    $ cat flag.txt
    picoCTF{z1p_crypt0_1s_n0t_s3cur3_w1th_w34k_p4ssw0rds}

Flag captured. 200 points. I sat there for a moment just staring at the terminal.

## The Moment It Clicked

The real turning point wasn't finding the password. It was going back to that hexdump and finally paying attention to the version byte I'd noticed and ignored. Version `0x33` (decimal 51) in a ZIP local file header is documented in the ZIP specification as indicating AES encryption. I'd seen the number. I hadn't bothered to look up what it meant.

Once I ran `zipinfo -v` and saw "AES Encryption / 256-bit" explicitly printed, everything reorganized in my head. `fcrackzip` failing silently made sense. The need for `zip2john` specifically made sense. The whole challenge was designed around knowing that distinction.

That's the thing about CTF forensics challenges — the file format usually tells you exactly what you need to know. You just have to actually read it.

## Full Trial Process: Every Step I Took

Step| Action| Command| Result| Why it failed / succeeded  
---|---|---|---|---  
1| Identify file type| `file secret_archive.zip`| ZIP archive, v2.0+| Succeeded — confirmed it's a valid ZIP  
2| Attempt extraction| `unzip secret_archive.zip`| Password prompt, then failure| Expected — file is encrypted  
3| Manual password guessing| `unzip -P <word> secret_archive.zip`| 15 attempts, all failed| Failed — password not an obvious keyword  
4| Check metadata| `strings secret_archive.zip`| Only filename visible| Failed — AES content is opaque to strings  
5| Binary inspection| `hexdump -C secret_archive.zip`| Saw version 0x33 byte| Partial — noticed AES indicator but didn't act on it  
6| fcrackzip dictionary attack| `fcrackzip -u -D -p rockyou.txt ...`| Silent, no output| Failed — fcrackzip doesn't support AES-encrypted ZIPs  
7| Confirm encryption type| `zipinfo -v secret_archive.zip`| AES-256 confirmed| Succeeded — this was the key diagnostic step  
8| Extract hash with zip2john| `zip2john secret_archive.zip > archive.hash`| $zip2$ hash generated| Succeeded — correct tool for AES ZIP  
9| John with custom wordlist| `john --wordlist=custom.txt archive.hash`| 0 hits| Failed — exact words not the password  
10| John with Single rules| `john --wordlist=custom.txt --rules=Single archive.hash`| archive2024 cracked in 3s| Succeeded — rule mangling hit the pattern  
11| Extract flag| `unzip -P archive2024 secret_archive.zip`| flag.txt extracted| Succeeded — password confirmed  
  
## Technical Background: Why AES-256 ZIP Is Strong But Passwords Aren't

AES-256 is genuinely secure encryption. The key itself — 256 bits of random data — would take longer than the age of the universe to brute force with any foreseeable hardware. The weakness isn't the algorithm. It's the password.

ZIP's AES implementation (the WinZip AE format) derives the actual encryption key from the password using PBKDF2-HMAC-SHA1 with 1000 iterations. This is deliberately slow to make brute forcing expensive — but 1000 iterations is not especially slow by modern standards. On a modern CPU you can test roughly 50,000–100,000 passwords per second. On a GPU with hashcat and the right module (`-m 13600`), that number climbs to millions per second.

The practical consequence: an AES-256 ZIP is only as secure as its password. "archive2024" is 11 characters with a simple pattern — a dictionary word plus a year. In real-world password auditing, that's classified as weak. It would fall to any competent dictionary attack within minutes even without knowing the target context.

### ZipCrypto vs. AES: Why the Distinction Matters

The older ZipCrypto (traditional PKZIP) encryption is even weaker for a different reason. It has a known-plaintext attack: if you know any 12 bytes of the plaintext (which is trivially true for ZIP files, since the local file header structure is predictable), tools like `pkcrack` can recover the encryption keys directly without knowing the password at all. This attack works regardless of password complexity.

When `zipinfo -v` shows "ZipCrypto" instead of "AES Encryption," your first move should be checking if a plaintext attack is viable — not running john at all. That's a completely different solve path.

### Real-World Context: Where This Shows Up Outside CTF

Password-protected ZIP files appear constantly in malware delivery — phishing emails often include encrypted ZIPs specifically because the password (included in the email body) prevents automated scanning tools from inspecting the payload. Security researchers and incident responders use exactly these tools — `zip2john`, john, hashcat — to recover credentials from seized devices and investigate malware samples. The CTF skill and the professional skill are the same skill.

There are also legitimate recovery scenarios: encrypted archives from former employees, old backup files with forgotten passwords, forensic examination of devices where the user is unavailable. The legal and ethical boundaries matter here — using these tools against files you don't have authorization to access is a different matter entirely.

## zip2john vs. hashcat: When to Use Which

`zip2john` \+ john is the right default when you're on a CPU-only system or when you expect a small, targeted wordlist to work quickly. John's rule sets (especially Single and Jumbo) are excellent at generating contextually plausible password variations from a seed wordlist — exactly what you want in CTF where the challenge setter chose a password related to the theme.

hashcat with `-m 13600` (WinZip) is the right choice when you need raw throughput — exhaustive wordlist runs against rockyou.txt, combinatory attacks, or mask attacks for known-length passwords with unknown character sets. GPU acceleration makes a substantial difference here, but you need to extract the hash first with `zip2john` in either case.
    
    
    # hashcat equivalent (after extracting hash with zip2john)
    $ hashcat -m 13600 -a 0 archive.hash /usr/share/wordlists/rockyou.txt
    hashcat (v6.2.6) starting...
    
    OpenCL API (OpenCL 3.0 PoCL 3.1+debian) - Platform #1 [The pocl project]
    ==========================================================================
    * Device #1: pthread-Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz, 13579/27222 MB (4096 MB allocatable), 4MCU
    
    Minimum password length supported by kernel: 0
    Maximum password length supported by kernel: 256
    
    Hashes: 1 digests; 1 unique digests, 1 unique salts
    ...
    Session..........: hashcat
    Status...........: Cracked
    Hash.Mode........: 13600 (WinZip)
    Hash.Target......: $zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...
    Time.Started.....: Thu Nov 14 14:35:02 2024 (1 min, 43 secs)
    Candidates.#1....: archive2024
    
    Started: Thu Nov 14 14:35:02 2024
    Stopped: Thu Nov 14 14:36:45 2024

## What I'd Do Differently Next Time

The biggest time sink wasn't the failed commands — it was not confirming the encryption type before choosing tools. `zipinfo -v` takes two seconds. If I'd run that before anything else, I would have skipped the entire `fcrackzip` detour and gone straight to `zip2john`.

My updated mental checklist for any encrypted ZIP in CTF:

  1. Run `zipinfo -v` immediately. Identify ZipCrypto vs. AES, and note the AES key size.
  2. If ZipCrypto: consider known-plaintext attack with pkcrack before any password cracking.
  3. If AES: run `zip2john` to extract the hash. Do not touch fcrackzip.
  4. Build a targeted wordlist from every word in the challenge title, description, and filename. Five minutes here saves an hour later.
  5. Run John with that wordlist first (`--rules=Single`). If no hit within 30 seconds, try `--rules=Jumbo`.
  6. If still stuck: rockyou.txt without rules. Then rockyou.txt with rules. Then reconsider whether password cracking is actually the path.
  7. Cap the entire attempt at 15 minutes before stepping back and asking if there's a different angle entirely.



The time limit matters. CTF challenges are designed to be solvable. If you've been running wordlists for 30 minutes with no result, the answer probably isn't "try a bigger wordlist." It's more likely that you're missing something about the challenge design.

## Quick Reference: zip2john Commands
    
    
    # Check encryption type first
    zipinfo -v target.zip
    
    # Extract hash
    zip2john target.zip > target.hash
    
    # Crack with custom wordlist + rules
    john --wordlist=custom.txt --rules=Single target.hash
    
    # Crack with rockyou
    john --wordlist=/usr/share/wordlists/rockyou.txt target.hash
    
    # Show cracked password
    john --show target.hash
    
    # hashcat equivalent (GPU)
    hashcat -m 13600 -a 0 target.hash /usr/share/wordlists/rockyou.txt
    
    # Extract with recovered password
    unzip -P <password> target.zip

## Final Thoughts

picoCTF's _archive_madness_ is a straightforward challenge once you know what you're doing — but I didn't know what I was doing for the first 20 minutes of it. The failure wasn't lack of knowledge about `zip2john`. It was not checking the basic file properties before reaching for tools.

The broader lesson, which I keep relearning in different forms: read the file before attacking it. ZIP headers, ELF headers, PNG chunks, PDF structure — they all tell you what the file actually is and what's relevant about it. Starting with `file` and then a format-specific inspector is worth more than jumping straight to attack tools.

If you found this writeup through a search for zip2john, I hope the failure log was as useful as the working commands. The 20 wasted minutes are the part worth reading.
    
    
    $ strings secret_archive.zip
    PK
    flag.txt
    

Just the filename and the PK magic bytes. Of course. The metadata gives you the filename and some structural information, but the password isn't stored there in any form a human can read. I knew this intellectually. I checked anyway. That's what CTF desperation looks like.

### Mistake #3: hexdump Without a Plan

After `strings` failed, I opened the file in hexdump. I was looking for... something. I'm not entirely sure what. A cleartext hint? A comment field? I scrolled through hex output for about five minutes before I admitted to myself I had no idea what I was looking for.
    
    
    $ hexdump -C secret_archive.zip | head -20
    00000000  50 4b 03 04 33 00 01 00  63 00 00 00 21 00 75 8a  |PK..3...c...!.u.|
    00000010  a4 6e 00 00 00 00 00 00  00 00 07 00 1c 00 66 6c  |.n............fl|
    00000020  61 67 2e 74 78 74 55 54  09 00 03 5e 2f 84 65 5f  |ag.txtUT...^/.e_|
    00000030  2f 84 65 55 78 0b 00 01  04 e8 03 00 00 04 e8 03  |/.eUx...........|
    00000040  00 00 01 99 07 00 51 4d  41 45 0c 00 19 18 c9 41  |......QMAE.....A|
    00000050  c0 28 4c 57 a2 fe 5c 9e  9a 4d a0 de b8 68 74 61  |.(LW..\..M...hta|

I can read a ZIP header. The `PK\x03\x04` signature, the version needed (`0x33` = 51, meaning AES encryption), the general purpose bit flag (`0x01` = encrypted). That version byte should have told me something important. I noticed it. I didn't act on it. That was the real mistake.

## The Rabbit Hole: fcrackzip and Why It Silently Failed

Around the 20-minute mark I remembered `fcrackzip` existed. I'd used it once before on a simpler challenge with ZipCrypto encryption. I figured it would work here too. This was the longest detour of the whole session — easily 15 minutes of confusion before I understood the problem.
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt secret_archive.zip
    

It ran. And ran. And ran. No output. No progress indicator. After five minutes I hit Ctrl+C and tried a smaller wordlist:
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    

Still nothing. I checked if the file was corrupted. I re-downloaded it. Same result. I tried the verbose flag:
    
    
    $ fcrackzip -v -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    found file 'flag.txt', (size cp/uc    173/   161, flags 1, chk 8a75)
    

It found the file. It attempted passwords. It never cracked it. This was the moment I finally went back to that `0x33` version byte from the hexdump.

Version 51 in a ZIP local file header means AES encryption — specifically, it means the file uses WinZip's AE-1 or AE-2 AES extension. `fcrackzip` doesn't support AES-encrypted ZIPs. It works fine against the older ZipCrypto (traditional PKZIP encryption), but AES is a different animal entirely. The tool was silently skipping every candidate because it couldn't even verify them correctly. I'd been pouring water into a bucket with no bottom.
    
    
    $ zipinfo -v secret_archive.zip
    Archive:  secret_archive.zip
    There is no zipfile comment.
    
    End-of-central-directory record:
    -------------------------------
    ...
    Central directory entry #1:
    ---------------------------
      flag.txt
      ...
      file security status:                 encrypted
      extended local header:                no
      file last modified on (DOS date/time):  ...
      32-bit CRC value (hex):               00000000
      compressed size:                      173 bytes
      uncompressed size:                    161 bytes
      length of filename:                   8 characters
      ...
      encryption method:                    AES Encryption
      AES key size:                         256-bit

AES-256. That explained everything. `fcrackzip` was never going to work. I needed a tool that understood this encryption format — and that's exactly what `zip2john` is built for.

## What zip2john Actually Does (And Why It Matters)

Before I show the commands that worked, it's worth understanding the mechanism. `zip2john` is part of the John the Ripper suite. Its job is not to crack passwords — it's to extract a crackable hash from the ZIP file's encryption metadata.

For AES-encrypted ZIPs (the WinZip AE format), the ZIP file stores a salt value and a verification value in the local file header. These are derived from the password using PBKDF2-HMAC-SHA1. `zip2john` reads these values and formats them into a hash string that John the Ripper (or hashcat, with the right module) can work with. The hash includes the salt, the number of iterations, and the verification bytes — everything needed to test a candidate password without actually decrypting the file contents.

This matters for a non-obvious reason: AES-256 ZIP encryption is actually strong. Unlike the old ZipCrypto (which has known plaintext attacks that can bypass brute force entirely), AES-256 ZIP has no practical cryptographic weakness. You're doing a genuine dictionary or brute force attack. That means wordlist quality is everything.

## The Commands That Actually Worked

### Step 1: Extract the Hash
    
    
    $ zip2john secret_archive.zip > archive.hash
    ver 2.0 efh 5455 efh 7875 secret_archive.zip/flag.txt PKZIP Encr: TS_chk, cmplen=173, decmplen=161, crc=0000 ts=8A75 cs=8a75 type=8
    
    
    
    $ cat archive.hash
    secret_archive.zip/flag.txt:$zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...[hash data]...*$/zip2$:flag.txt:secret_archive.zip::

That hash string is what John will work with. The `$zip2$` prefix tells John this is an AES-encrypted ZIP. The fields encode the salt, key length, verification data, and encrypted content sample.

### Step 2: Build a Targeted Wordlist

Before throwing rockyou.txt at it (which would take hours on CPU), I made a small targeted wordlist from everything I knew about the challenge:
    
    
    $ cat > custom.txt << 'EOF'
    archive
    madness
    secret
    archive_madness
    picoctf
    pico
    ctf
    flag
    secret_archive
    archivemadness
    picoCTF2024
    picoCTF2025
    zip
    password
    EOF

### Step 3: Run John with the Custom Wordlist First
    
    
    $ john --wordlist=custom.txt archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    0g 0:00:00:00 DONE (2024-11-14 14:23:10) 0g/s 933.3p/s 933.3p/s 933.3p/s ..
    Session completed.

No hit. The obvious guesses weren't it. Time for John's mangling rules:
    
    
    $ john --wordlist=custom.txt --rules=Single archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    archive2024       (secret_archive.zip/flag.txt)
    1g 0:00:00:03 DONE (2024-11-14 14:23:47) 0.3030g/s 8381p/s 8381p/s 8381p/s ..
    Use the "--show" option to display all cracked passwords reliably
    Session completed.

There it was. `archive2024`. John's Single rule set had appended "2024" to "archive" — a common CTF password construction pattern. The whole crack took about three seconds once I had the right wordlist.

### Step 4: Verify with --show and Extract the Flag
    
    
    $ john --show archive.hash
    secret_archive.zip/flag.txt:archive2024:flag.txt:secret_archive.zip::
    
    1 password hash cracked, 0 left
    
    
    $ unzip -P archive2024 secret_archive.zip
    Archive:  secret_archive.zip
      inflating: flag.txt
    
    $ cat flag.txt
    picoCTF{z1p_crypt0_1s_n0t_s3cur3_w1th_w34k_p4ssw0rds}

Flag captured. 200 points. I sat there for a moment just staring at the terminal.

## The Moment It Clicked

The real turning point wasn't finding the password. It was going back to that hexdump and finally paying attention to the version byte I'd noticed and ignored. Version `0x33` (decimal 51) in a ZIP local file header is documented in the ZIP specification as indicating AES encryption. I'd seen the number. I hadn't bothered to look up what it meant.

Once I ran `zipinfo -v` and saw "AES Encryption / 256-bit" explicitly printed, everything reorganized in my head. `fcrackzip` failing silently made sense. The need for `zip2john` specifically made sense. The whole challenge was designed around knowing that distinction.

That's the thing about CTF forensics challenges — the file format usually tells you exactly what you need to know. You just have to actually read it.

## Full Trial Process: Every Step I Took

Step| Action| Command| Result| Why it failed / succeeded  
---|---|---|---|---  
1| Identify file type| `file secret_archive.zip`| ZIP archive, v2.0+| Succeeded — confirmed it's a valid ZIP  
2| Attempt extraction| `unzip secret_archive.zip`| Password prompt, then failure| Expected — file is encrypted  
3| Manual password guessing| `unzip -P <word> secret_archive.zip`| 15 attempts, all failed| Failed — password not an obvious keyword  
4| Check metadata| `strings secret_archive.zip`| Only filename visible| Failed — AES content is opaque to strings  
5| Binary inspection| `hexdump -C secret_archive.zip`| Saw version 0x33 byte| Partial — noticed AES indicator but didn't act on it  
6| fcrackzip dictionary attack| `fcrackzip -u -D -p rockyou.txt ...`| Silent, no output| Failed — fcrackzip doesn't support AES-encrypted ZIPs  
7| Confirm encryption type| `zipinfo -v secret_archive.zip`| AES-256 confirmed| Succeeded — this was the key diagnostic step  
8| Extract hash with zip2john| `zip2john secret_archive.zip > archive.hash`| $zip2$ hash generated| Succeeded — correct tool for AES ZIP  
9| John with custom wordlist| `john --wordlist=custom.txt archive.hash`| 0 hits| Failed — exact words not the password  
10| John with Single rules| `john --wordlist=custom.txt --rules=Single archive.hash`| archive2024 cracked in 3s| Succeeded — rule mangling hit the pattern  
11| Extract flag| `unzip -P archive2024 secret_archive.zip`| flag.txt extracted| Succeeded — password confirmed  
  
## Technical Background: Why AES-256 ZIP Is Strong But Passwords Aren't

AES-256 is genuinely secure encryption. The key itself — 256 bits of random data — would take longer than the age of the universe to brute force with any foreseeable hardware. The weakness isn't the algorithm. It's the password.

ZIP's AES implementation (the WinZip AE format) derives the actual encryption key from the password using PBKDF2-HMAC-SHA1 with 1000 iterations. This is deliberately slow to make brute forcing expensive — but 1000 iterations is not especially slow by modern standards. On a modern CPU you can test roughly 50,000–100,000 passwords per second. On a GPU with hashcat and the right module (`-m 13600`), that number climbs to millions per second.

The practical consequence: an AES-256 ZIP is only as secure as its password. "archive2024" is 11 characters with a simple pattern — a dictionary word plus a year. In real-world password auditing, that's classified as weak. It would fall to any competent dictionary attack within minutes even without knowing the target context.

### ZipCrypto vs. AES: Why the Distinction Matters

The older ZipCrypto (traditional PKZIP) encryption is even weaker for a different reason. It has a known-plaintext attack: if you know any 12 bytes of the plaintext (which is trivially true for ZIP files, since the local file header structure is predictable), tools like `pkcrack` can recover the encryption keys directly without knowing the password at all. This attack works regardless of password complexity.

When `zipinfo -v` shows "ZipCrypto" instead of "AES Encryption," your first move should be checking if a plaintext attack is viable — not running john at all. That's a completely different solve path.

### Real-World Context: Where This Shows Up Outside CTF

Password-protected ZIP files appear constantly in malware delivery — phishing emails often include encrypted ZIPs specifically because the password (included in the email body) prevents automated scanning tools from inspecting the payload. Security researchers and incident responders use exactly these tools — `zip2john`, john, hashcat — to recover credentials from seized devices and investigate malware samples. The CTF skill and the professional skill are the same skill.

There are also legitimate recovery scenarios: encrypted archives from former employees, old backup files with forgotten passwords, forensic examination of devices where the user is unavailable. The legal and ethical boundaries matter here — using these tools against files you don't have authorization to access is a different matter entirely.

## zip2john vs. hashcat: When to Use Which

`zip2john` \+ john is the right default when you're on a CPU-only system or when you expect a small, targeted wordlist to work quickly. John's rule sets (especially Single and Jumbo) are excellent at generating contextually plausible password variations from a seed wordlist — exactly what you want in CTF where the challenge setter chose a password related to the theme.

hashcat with `-m 13600` (WinZip) is the right choice when you need raw throughput — exhaustive wordlist runs against rockyou.txt, combinatory attacks, or mask attacks for known-length passwords with unknown character sets. GPU acceleration makes a substantial difference here, but you need to extract the hash first with `zip2john` in either case.
    
    
    # hashcat equivalent (after extracting hash with zip2john)
    $ hashcat -m 13600 -a 0 archive.hash /usr/share/wordlists/rockyou.txt
    hashcat (v6.2.6) starting...
    
    OpenCL API (OpenCL 3.0 PoCL 3.1+debian) - Platform #1 [The pocl project]
    ==========================================================================
    * Device #1: pthread-Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz, 13579/27222 MB (4096 MB allocatable), 4MCU
    
    Minimum password length supported by kernel: 0
    Maximum password length supported by kernel: 256
    
    Hashes: 1 digests; 1 unique digests, 1 unique salts
    ...
    Session..........: hashcat
    Status...........: Cracked
    Hash.Mode........: 13600 (WinZip)
    Hash.Target......: $zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...
    Time.Started.....: Thu Nov 14 14:35:02 2024 (1 min, 43 secs)
    Candidates.#1....: archive2024
    
    Started: Thu Nov 14 14:35:02 2024
    Stopped: Thu Nov 14 14:36:45 2024

## What I'd Do Differently Next Time

The biggest time sink wasn't the failed commands — it was not confirming the encryption type before choosing tools. `zipinfo -v` takes two seconds. If I'd run that before anything else, I would have skipped the entire `fcrackzip` detour and gone straight to `zip2john`.

My updated mental checklist for any encrypted ZIP in CTF:

  1. Run `zipinfo -v` immediately. Identify ZipCrypto vs. AES, and note the AES key size.
  2. If ZipCrypto: consider known-plaintext attack with pkcrack before any password cracking.
  3. If AES: run `zip2john` to extract the hash. Do not touch fcrackzip.
  4. Build a targeted wordlist from every word in the challenge title, description, and filename. Five minutes here saves an hour later.
  5. Run John with that wordlist first (`--rules=Single`). If no hit within 30 seconds, try `--rules=Jumbo`.
  6. If still stuck: rockyou.txt without rules. Then rockyou.txt with rules. Then reconsider whether password cracking is actually the path.
  7. Cap the entire attempt at 15 minutes before stepping back and asking if there's a different angle entirely.



The time limit matters. CTF challenges are designed to be solvable. If you've been running wordlists for 30 minutes with no result, the answer probably isn't "try a bigger wordlist." It's more likely that you're missing something about the challenge design.

## Quick Reference: zip2john Commands
    
    
    # Check encryption type first
    zipinfo -v target.zip
    
    # Extract hash
    zip2john target.zip > target.hash
    
    # Crack with custom wordlist + rules
    john --wordlist=custom.txt --rules=Single target.hash
    
    # Crack with rockyou
    john --wordlist=/usr/share/wordlists/rockyou.txt target.hash
    
    # Show cracked password
    john --show target.hash
    
    # hashcat equivalent (GPU)
    hashcat -m 13600 -a 0 target.hash /usr/share/wordlists/rockyou.txt
    
    # Extract with recovered password
    unzip -P <password> target.zip

## Final Thoughts

picoCTF's _archive_madness_ is a straightforward challenge once you know what you're doing — but I didn't know what I was doing for the first 20 minutes of it. The failure wasn't lack of knowledge about `zip2john`. It was not checking the basic file properties before reaching for tools.

The broader lesson, which I keep relearning in different forms: read the file before attacking it. ZIP headers, ELF headers, PNG chunks, PDF structure — they all tell you what the file actually is and what's relevant about it. Starting with `file` and then a format-specific inspector is worth more than jumping straight to attack tools.

If you found this writeup through a search for zip2john, I hope the failure log was as useful as the working commands. The 20 wasted minutes are the part worth reading.

# zip2john in CTF: How I Finally Cracked an AES-256 Encrypted ZIP in picoCTF

I almost skipped this challenge entirely. A 200-point forensics problem in picoCTF labeled **archive_madness** — just a ZIP file, no hints, no README, nothing but `secret_archive.zip` staring back at me. I figured someone smarter would grab the points while I moved on to something I actually understood. I'm glad I didn't walk away.

This writeup covers every wrong turn I took, the 20-odd minutes I burned on approaches that had no business working, and the exact moment I understood what `zip2john` is actually doing — not just the commands, but the underlying structure it's reading. If you've stared at a password-protected ZIP and wondered where to even begin, this is for you.

## The Challenge: picoCTF archive_madness

The challenge description read: _"We found this archive in a suspicious place. Can you get what's inside? File: secret_archive.zip"_

Category: Forensics. Points: 200. That's medium difficulty in picoCTF terms — not trivial, but not the kind of thing that requires a novel exploit. The ZIP file downloaded cleanly. I ran `file` on it immediately:
    
    
    $ file secret_archive.zip
    secret_archive.zip: Zip archive data, at least v2.0 to extract, one file

Standard ZIP. Nothing exotic. I tried to unzip it:
    
    
    $ unzip secret_archive.zip
    Archive:  secret_archive.zip
    [secret_archive.zip] flag.txt password:
       skipping: flag.txt                incorrect password

Password protected. One file inside: `flag.txt`. This was going to be a password cracking challenge. Fine. I thought I knew exactly what to do.

## Why I Chose the Wrong Approach First (And Wasted 20 Minutes)

My reasoning at the time felt completely logical. The challenge was called _archive_madness_ and the file was named `secret_archive.zip`. In CTF, password hints often hide in plain sight — in the filename, the challenge title, the description. I'd seen that pattern before. So I spent the first chunk of time doing exactly the wrong things for what felt like exactly the right reasons.

### Mistake #1: Manual Password Guessing

I started typing passwords by hand using `unzip -P`. "archive", "madness", "secret", "flag", "pico", "picoctf", "password", "zip" — I tried about fifteen variations. Each one returned the same thing:
    
    
    $ unzip -P archive secret_archive.zip
    Archive:  secret_archive.zip
       skipping: flag.txt                incorrect password
    
    $ unzip -P picoctf secret_archive.zip
    Archive:  secret_archive.zip
       skipping: flag.txt                incorrect password
    
    $ unzip -P secret secret_archive.zip
    Archive:  secret_archive.zip
       skipping: flag.txt                incorrect password

Seven minutes gone. Nothing to show for it except slightly bruised confidence.

### Mistake #2: Running strings on an Encrypted File

Then I made the mistake that anyone who's done more stego than crypto tends to make: I reached for `strings`. The logic was: maybe the password is embedded somewhere unencrypted in the ZIP metadata. ZIP files have a local file header and a central directory — those aren't encrypted even when the content is. Worth checking, right?
    
    
    $ strings secret_archive.zip
    PK
    flag.txt
    

Just the filename and the PK magic bytes. Of course. The metadata gives you the filename and some structural information, but the password isn't stored there in any form a human can read. I knew this intellectually. I checked anyway. That's what CTF desperation looks like.

### Mistake #3: hexdump Without a Plan

After `strings` failed, I opened the file in hexdump. I was looking for... something. I'm not entirely sure what. A cleartext hint? A comment field? I scrolled through hex output for about five minutes before I admitted to myself I had no idea what I was looking for.
    
    
    $ hexdump -C secret_archive.zip | head -20
    00000000  50 4b 03 04 33 00 01 00  63 00 00 00 21 00 75 8a  |PK..3...c...!.u.|
    00000010  a4 6e 00 00 00 00 00 00  00 00 07 00 1c 00 66 6c  |.n............fl|
    00000020  61 67 2e 74 78 74 55 54  09 00 03 5e 2f 84 65 5f  |ag.txtUT...^/.e_|
    00000030  2f 84 65 55 78 0b 00 01  04 e8 03 00 00 04 e8 03  |/.eUx...........|
    00000040  00 00 01 99 07 00 51 4d  41 45 0c 00 19 18 c9 41  |......QMAE.....A|
    00000050  c0 28 4c 57 a2 fe 5c 9e  9a 4d a0 de b8 68 74 61  |.(LW..\..M...hta|

I can read a ZIP header. The `PK\x03\x04` signature, the version needed (`0x33` = 51, meaning AES encryption), the general purpose bit flag (`0x01` = encrypted). That version byte should have told me something important. I noticed it. I didn't act on it. That was the real mistake.

## The Rabbit Hole: fcrackzip and Why It Silently Failed

Around the 20-minute mark I remembered `fcrackzip` existed. I'd used it once before on a simpler challenge with ZipCrypto encryption. I figured it would work here too. This was the longest detour of the whole session — easily 15 minutes of confusion before I understood the problem.
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt secret_archive.zip
    

It ran. And ran. And ran. No output. No progress indicator. After five minutes I hit Ctrl+C and tried a smaller wordlist:
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    

Still nothing. I checked if the file was corrupted. I re-downloaded it. Same result. I tried the verbose flag:
    
    
    $ fcrackzip -v -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    found file 'flag.txt', (size cp/uc    173/   161, flags 1, chk 8a75)
    

It found the file. It attempted passwords. It never cracked it. This was the moment I finally went back to that `0x33` version byte from the hexdump.

Version 51 in a ZIP local file header means AES encryption — specifically, it means the file uses WinZip's AE-1 or AE-2 AES extension. `fcrackzip` doesn't support AES-encrypted ZIPs. It works fine against the older ZipCrypto (traditional PKZIP encryption), but AES is a different animal entirely. The tool was silently skipping every candidate because it couldn't even verify them correctly. I'd been pouring water into a bucket with no bottom.
    
    
    $ zipinfo -v secret_archive.zip
    Archive:  secret_archive.zip
    There is no zipfile comment.
    
    End-of-central-directory record:
    -------------------------------
    ...
    Central directory entry #1:
    ---------------------------
      flag.txt
      ...
      file security status:                 encrypted
      extended local header:                no
      file last modified on (DOS date/time):  ...
      32-bit CRC value (hex):               00000000
      compressed size:                      173 bytes
      uncompressed size:                    161 bytes
      length of filename:                   8 characters
      ...
      encryption method:                    AES Encryption
      AES key size:                         256-bit

AES-256. That explained everything. `fcrackzip` was never going to work. I needed a tool that understood this encryption format — and that's exactly what `zip2john` is built for.

## What zip2john Actually Does (And Why It Matters)

Before I show the commands that worked, it's worth understanding the mechanism. `zip2john` is part of the John the Ripper suite. Its job is not to crack passwords — it's to extract a crackable hash from the ZIP file's encryption metadata.

For AES-encrypted ZIPs (the WinZip AE format), the ZIP file stores a salt value and a verification value in the local file header. These are derived from the password using PBKDF2-HMAC-SHA1. `zip2john` reads these values and formats them into a hash string that John the Ripper (or hashcat, with the right module) can work with. The hash includes the salt, the number of iterations, and the verification bytes — everything needed to test a candidate password without actually decrypting the file contents.

This matters for a non-obvious reason: AES-256 ZIP encryption is actually strong. Unlike the old ZipCrypto (which has known plaintext attacks that can bypass brute force entirely), AES-256 ZIP has no practical cryptographic weakness. You're doing a genuine dictionary or brute force attack. That means wordlist quality is everything.

## The Commands That Actually Worked

### Step 1: Extract the Hash
    
    
    $ zip2john secret_archive.zip > archive.hash
    ver 2.0 efh 5455 efh 7875 secret_archive.zip/flag.txt PKZIP Encr: TS_chk, cmplen=173, decmplen=161, crc=0000 ts=8A75 cs=8a75 type=8
    
    
    
    $ cat archive.hash
    secret_archive.zip/flag.txt:$zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...[hash data]...*$/zip2$:flag.txt:secret_archive.zip::

That hash string is what John will work with. The `$zip2$` prefix tells John this is an AES-encrypted ZIP. The fields encode the salt, key length, verification data, and encrypted content sample.

### Step 2: Build a Targeted Wordlist

Before throwing rockyou.txt at it (which would take hours on CPU), I made a small targeted wordlist from everything I knew about the challenge:
    
    
    $ cat > custom.txt << 'EOF'
    archive
    madness
    secret
    archive_madness
    picoctf
    pico
    ctf
    flag
    secret_archive
    archivemadness
    picoCTF2024
    picoCTF2025
    zip
    password
    EOF

### Step 3: Run John with the Custom Wordlist First
    
    
    $ john --wordlist=custom.txt archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    0g 0:00:00:00 DONE (2024-11-14 14:23:10) 0g/s 933.3p/s 933.3p/s 933.3p/s ..
    Session completed.

No hit. The obvious guesses weren't it. Time for John's mangling rules:
    
    
    $ john --wordlist=custom.txt --rules=Single archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    archive2024       (secret_archive.zip/flag.txt)
    1g 0:00:00:03 DONE (2024-11-14 14:23:47) 0.3030g/s 8381p/s 8381p/s 8381p/s ..
    Use the "--show" option to display all cracked passwords reliably
    Session completed.

There it was. `archive2024`. John's Single rule set had appended "2024" to "archive" — a common CTF password construction pattern. The whole crack took about three seconds once I had the right wordlist.

### Step 4: Verify with --show and Extract the Flag
    
    
    $ john --show archive.hash
    secret_archive.zip/flag.txt:archive2024:flag.txt:secret_archive.zip::
    
    1 password hash cracked, 0 left
    
    
    $ unzip -P archive2024 secret_archive.zip
    Archive:  secret_archive.zip
      inflating: flag.txt
    
    $ cat flag.txt
    picoCTF{z1p_crypt0_1s_n0t_s3cur3_w1th_w34k_p4ssw0rds}

Flag captured. 200 points. I sat there for a moment just staring at the terminal.

## The Moment It Clicked

The real turning point wasn't finding the password. It was going back to that hexdump and finally paying attention to the version byte I'd noticed and ignored. Version `0x33` (decimal 51) in a ZIP local file header is documented in the ZIP specification as indicating AES encryption. I'd seen the number. I hadn't bothered to look up what it meant.

Once I ran `zipinfo -v` and saw "AES Encryption / 256-bit" explicitly printed, everything reorganized in my head. `fcrackzip` failing silently made sense. The need for `zip2john` specifically made sense. The whole challenge was designed around knowing that distinction.

That's the thing about CTF forensics challenges — the file format usually tells you exactly what you need to know. You just have to actually read it.

## Full Trial Process: Every Step I Took

Step| Action| Command| Result| Why it failed / succeeded  
---|---|---|---|---  
1| Identify file type| `file secret_archive.zip`| ZIP archive, v2.0+| Succeeded — confirmed it's a valid ZIP  
2| Attempt extraction| `unzip secret_archive.zip`| Password prompt, then failure| Expected — file is encrypted  
3| Manual password guessing| `unzip -P <word> secret_archive.zip`| 15 attempts, all failed| Failed — password not an obvious keyword  
4| Check metadata| `strings secret_archive.zip`| Only filename visible| Failed — AES content is opaque to strings  
5| Binary inspection| `hexdump -C secret_archive.zip`| Saw version 0x33 byte| Partial — noticed AES indicator but didn't act on it  
6| fcrackzip dictionary attack| `fcrackzip -u -D -p rockyou.txt ...`| Silent, no output| Failed — fcrackzip doesn't support AES-encrypted ZIPs  
7| Confirm encryption type| `zipinfo -v secret_archive.zip`| AES-256 confirmed| Succeeded — this was the key diagnostic step  
8| Extract hash with zip2john| `zip2john secret_archive.zip > archive.hash`| $zip2$ hash generated| Succeeded — correct tool for AES ZIP  
9| John with custom wordlist| `john --wordlist=custom.txt archive.hash`| 0 hits| Failed — exact words not the password  
10| John with Single rules| `john --wordlist=custom.txt --rules=Single archive.hash`| archive2024 cracked in 3s| Succeeded — rule mangling hit the pattern  
11| Extract flag| `unzip -P archive2024 secret_archive.zip`| flag.txt extracted| Succeeded — password confirmed  
  
## Technical Background: Why AES-256 ZIP Is Strong But Passwords Aren't

AES-256 is genuinely secure encryption. The key itself — 256 bits of random data — would take longer than the age of the universe to brute force with any foreseeable hardware. The weakness isn't the algorithm. It's the password.

ZIP's AES implementation (the WinZip AE format) derives the actual encryption key from the password using PBKDF2-HMAC-SHA1 with 1000 iterations. This is deliberately slow to make brute forcing expensive — but 1000 iterations is not especially slow by modern standards. On a modern CPU you can test roughly 50,000–100,000 passwords per second. On a GPU with hashcat and the right module (`-m 13600`), that number climbs to millions per second.

The practical consequence: an AES-256 ZIP is only as secure as its password. "archive2024" is 11 characters with a simple pattern — a dictionary word plus a year. In real-world password auditing, that's classified as weak. It would fall to any competent dictionary attack within minutes even without knowing the target context.

### ZipCrypto vs. AES: Why the Distinction Matters

The older ZipCrypto (traditional PKZIP) encryption is even weaker for a different reason. It has a known-plaintext attack: if you know any 12 bytes of the plaintext (which is trivially true for ZIP files, since the local file header structure is predictable), tools like `pkcrack` can recover the encryption keys directly without knowing the password at all. This attack works regardless of password complexity.

When `zipinfo -v` shows "ZipCrypto" instead of "AES Encryption," your first move should be checking if a plaintext attack is viable — not running john at all. That's a completely different solve path.

### Real-World Context: Where This Shows Up Outside CTF

Password-protected ZIP files appear constantly in malware delivery — phishing emails often include encrypted ZIPs specifically because the password (included in the email body) prevents automated scanning tools from inspecting the payload. Security researchers and incident responders use exactly these tools — `zip2john`, john, hashcat — to recover credentials from seized devices and investigate malware samples. The CTF skill and the professional skill are the same skill.

There are also legitimate recovery scenarios: encrypted archives from former employees, old backup files with forgotten passwords, forensic examination of devices where the user is unavailable. The legal and ethical boundaries matter here — using these tools against files you don't have authorization to access is a different matter entirely.

## zip2john vs. hashcat: When to Use Which

`zip2john` \+ john is the right default when you're on a CPU-only system or when you expect a small, targeted wordlist to work quickly. John's rule sets (especially Single and Jumbo) are excellent at generating contextually plausible password variations from a seed wordlist — exactly what you want in CTF where the challenge setter chose a password related to the theme.

hashcat with `-m 13600` (WinZip) is the right choice when you need raw throughput — exhaustive wordlist runs against rockyou.txt, combinatory attacks, or mask attacks for known-length passwords with unknown character sets. GPU acceleration makes a substantial difference here, but you need to extract the hash first with `zip2john` in either case.
    
    
    # hashcat equivalent (after extracting hash with zip2john)
    $ hashcat -m 13600 -a 0 archive.hash /usr/share/wordlists/rockyou.txt
    hashcat (v6.2.6) starting...
    
    OpenCL API (OpenCL 3.0 PoCL 3.1+debian) - Platform #1 [The pocl project]
    ==========================================================================
    * Device #1: pthread-Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz, 13579/27222 MB (4096 MB allocatable), 4MCU
    
    Minimum password length supported by kernel: 0
    Maximum password length supported by kernel: 256
    
    Hashes: 1 digests; 1 unique digests, 1 unique salts
    ...
    Session..........: hashcat
    Status...........: Cracked
    Hash.Mode........: 13600 (WinZip)
    Hash.Target......: $zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...
    Time.Started.....: Thu Nov 14 14:35:02 2024 (1 min, 43 secs)
    Candidates.#1....: archive2024
    
    Started: Thu Nov 14 14:35:02 2024
    Stopped: Thu Nov 14 14:36:45 2024

## What I'd Do Differently Next Time

The biggest time sink wasn't the failed commands — it was not confirming the encryption type before choosing tools. `zipinfo -v` takes two seconds. If I'd run that before anything else, I would have skipped the entire `fcrackzip` detour and gone straight to `zip2john`.

My updated mental checklist for any encrypted ZIP in CTF:

  1. Run `zipinfo -v` immediately. Identify ZipCrypto vs. AES, and note the AES key size.
  2. If ZipCrypto: consider known-plaintext attack with pkcrack before any password cracking.
  3. If AES: run `zip2john` to extract the hash. Do not touch fcrackzip.
  4. Build a targeted wordlist from every word in the challenge title, description, and filename. Five minutes here saves an hour later.
  5. Run John with that wordlist first (`--rules=Single`). If no hit within 30 seconds, try `--rules=Jumbo`.
  6. If still stuck: rockyou.txt without rules. Then rockyou.txt with rules. Then reconsider whether password cracking is actually the path.
  7. Cap the entire attempt at 15 minutes before stepping back and asking if there's a different angle entirely.



The time limit matters. CTF challenges are designed to be solvable. If you've been running wordlists for 30 minutes with no result, the answer probably isn't "try a bigger wordlist." It's more likely that you're missing something about the challenge design.

## Quick Reference: zip2john Commands
    
    
    # Check encryption type first
    zipinfo -v target.zip
    
    # Extract hash
    zip2john target.zip > target.hash
    
    # Crack with custom wordlist + rules
    john --wordlist=custom.txt --rules=Single target.hash
    
    # Crack with rockyou
    john --wordlist=/usr/share/wordlists/rockyou.txt target.hash
    
    # Show cracked password
    john --show target.hash
    
    # hashcat equivalent (GPU)
    hashcat -m 13600 -a 0 target.hash /usr/share/wordlists/rockyou.txt
    
    # Extract with recovered password
    unzip -P <password> target.zip

## Final Thoughts

picoCTF's _archive_madness_ is a straightforward challenge once you know what you're doing — but I didn't know what I was doing for the first 20 minutes of it. The failure wasn't lack of knowledge about `zip2john`. It was not checking the basic file properties before reaching for tools.

The broader lesson, which I keep relearning in different forms: read the file before attacking it. ZIP headers, ELF headers, PNG chunks, PDF structure — they all tell you what the file actually is and what's relevant about it. Starting with `file` and then a format-specific inspector is worth more than jumping straight to attack tools.

If you found this writeup through a search for zip2john, I hope the failure log was as useful as the working commands. The 20 wasted minutes are the part worth reading.
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt secret_archive.zip
    

It ran. And ran. And ran. No output. No progress indicator. After five minutes I hit Ctrl+C and tried a smaller wordlist:
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    

Still nothing. I checked if the file was corrupted. I re-downloaded it. Same result. I tried the verbose flag:
    
    
    $ fcrackzip -v -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    found file 'flag.txt', (size cp/uc    173/   161, flags 1, chk 8a75)
    

It found the file. It attempted passwords. It never cracked it. This was the moment I finally went back to that `0x33` version byte from the hexdump.

Version 51 in a ZIP local file header means AES encryption — specifically, it means the file uses WinZip's AE-1 or AE-2 AES extension. `fcrackzip` doesn't support AES-encrypted ZIPs. It works fine against the older ZipCrypto (traditional PKZIP encryption), but AES is a different animal entirely. The tool was silently skipping every candidate because it couldn't even verify them correctly. I'd been pouring water into a bucket with no bottom.
    
    
    $ zipinfo -v secret_archive.zip
    Archive:  secret_archive.zip
    There is no zipfile comment.
    
    End-of-central-directory record:
    -------------------------------
    ...
    Central directory entry #1:
    ---------------------------
      flag.txt
      ...
      file security status:                 encrypted
      extended local header:                no
      file last modified on (DOS date/time):  ...
      32-bit CRC value (hex):               00000000
      compressed size:                      173 bytes
      uncompressed size:                    161 bytes
      length of filename:                   8 characters
      ...
      encryption method:                    AES Encryption
      AES key size:                         256-bit

AES-256. That explained everything. `fcrackzip` was never going to work. I needed a tool that understood this encryption format — and that's exactly what `zip2john` is built for.

## What zip2john Actually Does (And Why It Matters)

Before I show the commands that worked, it's worth understanding the mechanism. `zip2john` is part of the John the Ripper suite. Its job is not to crack passwords — it's to extract a crackable hash from the ZIP file's encryption metadata.

For AES-encrypted ZIPs (the WinZip AE format), the ZIP file stores a salt value and a verification value in the local file header. These are derived from the password using PBKDF2-HMAC-SHA1. `zip2john` reads these values and formats them into a hash string that John the Ripper (or hashcat, with the right module) can work with. The hash includes the salt, the number of iterations, and the verification bytes — everything needed to test a candidate password without actually decrypting the file contents.

This matters for a non-obvious reason: AES-256 ZIP encryption is actually strong. Unlike the old ZipCrypto (which has known plaintext attacks that can bypass brute force entirely), AES-256 ZIP has no practical cryptographic weakness. You're doing a genuine dictionary or brute force attack. That means wordlist quality is everything.

## The Commands That Actually Worked

### Step 1: Extract the Hash
    
    
    $ zip2john secret_archive.zip > archive.hash
    ver 2.0 efh 5455 efh 7875 secret_archive.zip/flag.txt PKZIP Encr: TS_chk, cmplen=173, decmplen=161, crc=0000 ts=8A75 cs=8a75 type=8
    
    
    
    $ cat archive.hash
    secret_archive.zip/flag.txt:$zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...[hash data]...*$/zip2$:flag.txt:secret_archive.zip::

That hash string is what John will work with. The `$zip2$` prefix tells John this is an AES-encrypted ZIP. The fields encode the salt, key length, verification data, and encrypted content sample.

### Step 2: Build a Targeted Wordlist

Before throwing rockyou.txt at it (which would take hours on CPU), I made a small targeted wordlist from everything I knew about the challenge:
    
    
    $ cat > custom.txt << 'EOF'
    archive
    madness
    secret
    archive_madness
    picoctf
    pico
    ctf
    flag
    secret_archive
    archivemadness
    picoCTF2024
    picoCTF2025
    zip
    password
    EOF

### Step 3: Run John with the Custom Wordlist First
    
    
    $ john --wordlist=custom.txt archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    0g 0:00:00:00 DONE (2024-11-14 14:23:10) 0g/s 933.3p/s 933.3p/s 933.3p/s ..
    Session completed.

No hit. The obvious guesses weren't it. Time for John's mangling rules:
    
    
    $ john --wordlist=custom.txt --rules=Single archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    archive2024       (secret_archive.zip/flag.txt)
    1g 0:00:00:03 DONE (2024-11-14 14:23:47) 0.3030g/s 8381p/s 8381p/s 8381p/s ..
    Use the "--show" option to display all cracked passwords reliably
    Session completed.

There it was. `archive2024`. John's Single rule set had appended "2024" to "archive" — a common CTF password construction pattern. The whole crack took about three seconds once I had the right wordlist.

### Step 4: Verify with --show and Extract the Flag
    
    
    $ john --show archive.hash
    secret_archive.zip/flag.txt:archive2024:flag.txt:secret_archive.zip::
    
    1 password hash cracked, 0 left
    
    
    $ unzip -P archive2024 secret_archive.zip
    Archive:  secret_archive.zip
      inflating: flag.txt
    
    $ cat flag.txt
    picoCTF{z1p_crypt0_1s_n0t_s3cur3_w1th_w34k_p4ssw0rds}

Flag captured. 200 points. I sat there for a moment just staring at the terminal.

## The Moment It Clicked

The real turning point wasn't finding the password. It was going back to that hexdump and finally paying attention to the version byte I'd noticed and ignored. Version `0x33` (decimal 51) in a ZIP local file header is documented in the ZIP specification as indicating AES encryption. I'd seen the number. I hadn't bothered to look up what it meant.

Once I ran `zipinfo -v` and saw "AES Encryption / 256-bit" explicitly printed, everything reorganized in my head. `fcrackzip` failing silently made sense. The need for `zip2john` specifically made sense. The whole challenge was designed around knowing that distinction.

That's the thing about CTF forensics challenges — the file format usually tells you exactly what you need to know. You just have to actually read it.

## Full Trial Process: Every Step I Took

Step| Action| Command| Result| Why it failed / succeeded  
---|---|---|---|---  
1| Identify file type| `file secret_archive.zip`| ZIP archive, v2.0+| Succeeded — confirmed it's a valid ZIP  
2| Attempt extraction| `unzip secret_archive.zip`| Password prompt, then failure| Expected — file is encrypted  
3| Manual password guessing| `unzip -P <word> secret_archive.zip`| 15 attempts, all failed| Failed — password not an obvious keyword  
4| Check metadata| `strings secret_archive.zip`| Only filename visible| Failed — AES content is opaque to strings  
5| Binary inspection| `hexdump -C secret_archive.zip`| Saw version 0x33 byte| Partial — noticed AES indicator but didn't act on it  
6| fcrackzip dictionary attack| `fcrackzip -u -D -p rockyou.txt ...`| Silent, no output| Failed — fcrackzip doesn't support AES-encrypted ZIPs  
7| Confirm encryption type| `zipinfo -v secret_archive.zip`| AES-256 confirmed| Succeeded — this was the key diagnostic step  
8| Extract hash with zip2john| `zip2john secret_archive.zip > archive.hash`| $zip2$ hash generated| Succeeded — correct tool for AES ZIP  
9| John with custom wordlist| `john --wordlist=custom.txt archive.hash`| 0 hits| Failed — exact words not the password  
10| John with Single rules| `john --wordlist=custom.txt --rules=Single archive.hash`| archive2024 cracked in 3s| Succeeded — rule mangling hit the pattern  
11| Extract flag| `unzip -P archive2024 secret_archive.zip`| flag.txt extracted| Succeeded — password confirmed  
  
## Technical Background: Why AES-256 ZIP Is Strong But Passwords Aren't

AES-256 is genuinely secure encryption. The key itself — 256 bits of random data — would take longer than the age of the universe to brute force with any foreseeable hardware. The weakness isn't the algorithm. It's the password.

ZIP's AES implementation (the WinZip AE format) derives the actual encryption key from the password using PBKDF2-HMAC-SHA1 with 1000 iterations. This is deliberately slow to make brute forcing expensive — but 1000 iterations is not especially slow by modern standards. On a modern CPU you can test roughly 50,000–100,000 passwords per second. On a GPU with hashcat and the right module (`-m 13600`), that number climbs to millions per second.

The practical consequence: an AES-256 ZIP is only as secure as its password. "archive2024" is 11 characters with a simple pattern — a dictionary word plus a year. In real-world password auditing, that's classified as weak. It would fall to any competent dictionary attack within minutes even without knowing the target context.

### ZipCrypto vs. AES: Why the Distinction Matters

The older ZipCrypto (traditional PKZIP) encryption is even weaker for a different reason. It has a known-plaintext attack: if you know any 12 bytes of the plaintext (which is trivially true for ZIP files, since the local file header structure is predictable), tools like `pkcrack` can recover the encryption keys directly without knowing the password at all. This attack works regardless of password complexity.

When `zipinfo -v` shows "ZipCrypto" instead of "AES Encryption," your first move should be checking if a plaintext attack is viable — not running john at all. That's a completely different solve path.

### Real-World Context: Where This Shows Up Outside CTF

Password-protected ZIP files appear constantly in malware delivery — phishing emails often include encrypted ZIPs specifically because the password (included in the email body) prevents automated scanning tools from inspecting the payload. Security researchers and incident responders use exactly these tools — `zip2john`, john, hashcat — to recover credentials from seized devices and investigate malware samples. The CTF skill and the professional skill are the same skill.

There are also legitimate recovery scenarios: encrypted archives from former employees, old backup files with forgotten passwords, forensic examination of devices where the user is unavailable. The legal and ethical boundaries matter here — using these tools against files you don't have authorization to access is a different matter entirely.

## zip2john vs. hashcat: When to Use Which

`zip2john` \+ john is the right default when you're on a CPU-only system or when you expect a small, targeted wordlist to work quickly. John's rule sets (especially Single and Jumbo) are excellent at generating contextually plausible password variations from a seed wordlist — exactly what you want in CTF where the challenge setter chose a password related to the theme.

hashcat with `-m 13600` (WinZip) is the right choice when you need raw throughput — exhaustive wordlist runs against rockyou.txt, combinatory attacks, or mask attacks for known-length passwords with unknown character sets. GPU acceleration makes a substantial difference here, but you need to extract the hash first with `zip2john` in either case.
    
    
    # hashcat equivalent (after extracting hash with zip2john)
    $ hashcat -m 13600 -a 0 archive.hash /usr/share/wordlists/rockyou.txt
    hashcat (v6.2.6) starting...
    
    OpenCL API (OpenCL 3.0 PoCL 3.1+debian) - Platform #1 [The pocl project]
    ==========================================================================
    * Device #1: pthread-Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz, 13579/27222 MB (4096 MB allocatable), 4MCU
    
    Minimum password length supported by kernel: 0
    Maximum password length supported by kernel: 256
    
    Hashes: 1 digests; 1 unique digests, 1 unique salts
    ...
    Session..........: hashcat
    Status...........: Cracked
    Hash.Mode........: 13600 (WinZip)
    Hash.Target......: $zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...
    Time.Started.....: Thu Nov 14 14:35:02 2024 (1 min, 43 secs)
    Candidates.#1....: archive2024
    
    Started: Thu Nov 14 14:35:02 2024
    Stopped: Thu Nov 14 14:36:45 2024

## What I'd Do Differently Next Time

The biggest time sink wasn't the failed commands — it was not confirming the encryption type before choosing tools. `zipinfo -v` takes two seconds. If I'd run that before anything else, I would have skipped the entire `fcrackzip` detour and gone straight to `zip2john`.

My updated mental checklist for any encrypted ZIP in CTF:

  1. Run `zipinfo -v` immediately. Identify ZipCrypto vs. AES, and note the AES key size.
  2. If ZipCrypto: consider known-plaintext attack with pkcrack before any password cracking.
  3. If AES: run `zip2john` to extract the hash. Do not touch fcrackzip.
  4. Build a targeted wordlist from every word in the challenge title, description, and filename. Five minutes here saves an hour later.
  5. Run John with that wordlist first (`--rules=Single`). If no hit within 30 seconds, try `--rules=Jumbo`.
  6. If still stuck: rockyou.txt without rules. Then rockyou.txt with rules. Then reconsider whether password cracking is actually the path.
  7. Cap the entire attempt at 15 minutes before stepping back and asking if there's a different angle entirely.



The time limit matters. CTF challenges are designed to be solvable. If you've been running wordlists for 30 minutes with no result, the answer probably isn't "try a bigger wordlist." It's more likely that you're missing something about the challenge design.

## Quick Reference: zip2john Commands
    
    
    # Check encryption type first
    zipinfo -v target.zip
    
    # Extract hash
    zip2john target.zip > target.hash
    
    # Crack with custom wordlist + rules
    john --wordlist=custom.txt --rules=Single target.hash
    
    # Crack with rockyou
    john --wordlist=/usr/share/wordlists/rockyou.txt target.hash
    
    # Show cracked password
    john --show target.hash
    
    # hashcat equivalent (GPU)
    hashcat -m 13600 -a 0 target.hash /usr/share/wordlists/rockyou.txt
    
    # Extract with recovered password
    unzip -P <password> target.zip

## Final Thoughts

picoCTF's _archive_madness_ is a straightforward challenge once you know what you're doing — but I didn't know what I was doing for the first 20 minutes of it. The failure wasn't lack of knowledge about `zip2john`. It was not checking the basic file properties before reaching for tools.

The broader lesson, which I keep relearning in different forms: read the file before attacking it. ZIP headers, ELF headers, PNG chunks, PDF structure — they all tell you what the file actually is and what's relevant about it. Starting with `file` and then a format-specific inspector is worth more than jumping straight to attack tools.

If you found this writeup through a search for zip2john, I hope the failure log was as useful as the working commands. The 20 wasted minutes are the part worth reading.
    
    
    $ strings secret_archive.zip
    PK
    flag.txt
    

Just the filename and the PK magic bytes. Of course. The metadata gives you the filename and some structural information, but the password isn't stored there in any form a human can read. I knew this intellectually. I checked anyway. That's what CTF desperation looks like.

### Mistake #3: hexdump Without a Plan

After `strings` failed, I opened the file in hexdump. I was looking for... something. I'm not entirely sure what. A cleartext hint? A comment field? I scrolled through hex output for about five minutes before I admitted to myself I had no idea what I was looking for.
    
    
    $ hexdump -C secret_archive.zip | head -20
    00000000  50 4b 03 04 33 00 01 00  63 00 00 00 21 00 75 8a  |PK..3...c...!.u.|
    00000010  a4 6e 00 00 00 00 00 00  00 00 07 00 1c 00 66 6c  |.n............fl|
    00000020  61 67 2e 74 78 74 55 54  09 00 03 5e 2f 84 65 5f  |ag.txtUT...^/.e_|
    00000030  2f 84 65 55 78 0b 00 01  04 e8 03 00 00 04 e8 03  |/.eUx...........|
    00000040  00 00 01 99 07 00 51 4d  41 45 0c 00 19 18 c9 41  |......QMAE.....A|
    00000050  c0 28 4c 57 a2 fe 5c 9e  9a 4d a0 de b8 68 74 61  |.(LW..\..M...hta|

I can read a ZIP header. The `PK\x03\x04` signature, the version needed (`0x33` = 51, meaning AES encryption), the general purpose bit flag (`0x01` = encrypted). That version byte should have told me something important. I noticed it. I didn't act on it. That was the real mistake.

## The Rabbit Hole: fcrackzip and Why It Silently Failed

Around the 20-minute mark I remembered `fcrackzip` existed. I'd used it once before on a simpler challenge with ZipCrypto encryption. I figured it would work here too. This was the longest detour of the whole session — easily 15 minutes of confusion before I understood the problem.
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt secret_archive.zip
    

It ran. And ran. And ran. No output. No progress indicator. After five minutes I hit Ctrl+C and tried a smaller wordlist:
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    

Still nothing. I checked if the file was corrupted. I re-downloaded it. Same result. I tried the verbose flag:
    
    
    $ fcrackzip -v -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    found file 'flag.txt', (size cp/uc    173/   161, flags 1, chk 8a75)
    

It found the file. It attempted passwords. It never cracked it. This was the moment I finally went back to that `0x33` version byte from the hexdump.

Version 51 in a ZIP local file header means AES encryption — specifically, it means the file uses WinZip's AE-1 or AE-2 AES extension. `fcrackzip` doesn't support AES-encrypted ZIPs. It works fine against the older ZipCrypto (traditional PKZIP encryption), but AES is a different animal entirely. The tool was silently skipping every candidate because it couldn't even verify them correctly. I'd been pouring water into a bucket with no bottom.
    
    
    $ zipinfo -v secret_archive.zip
    Archive:  secret_archive.zip
    There is no zipfile comment.
    
    End-of-central-directory record:
    -------------------------------
    ...
    Central directory entry #1:
    ---------------------------
      flag.txt
      ...
      file security status:                 encrypted
      extended local header:                no
      file last modified on (DOS date/time):  ...
      32-bit CRC value (hex):               00000000
      compressed size:                      173 bytes
      uncompressed size:                    161 bytes
      length of filename:                   8 characters
      ...
      encryption method:                    AES Encryption
      AES key size:                         256-bit

AES-256. That explained everything. `fcrackzip` was never going to work. I needed a tool that understood this encryption format — and that's exactly what `zip2john` is built for.

## What zip2john Actually Does (And Why It Matters)

Before I show the commands that worked, it's worth understanding the mechanism. `zip2john` is part of the John the Ripper suite. Its job is not to crack passwords — it's to extract a crackable hash from the ZIP file's encryption metadata.

For AES-encrypted ZIPs (the WinZip AE format), the ZIP file stores a salt value and a verification value in the local file header. These are derived from the password using PBKDF2-HMAC-SHA1. `zip2john` reads these values and formats them into a hash string that John the Ripper (or hashcat, with the right module) can work with. The hash includes the salt, the number of iterations, and the verification bytes — everything needed to test a candidate password without actually decrypting the file contents.

This matters for a non-obvious reason: AES-256 ZIP encryption is actually strong. Unlike the old ZipCrypto (which has known plaintext attacks that can bypass brute force entirely), AES-256 ZIP has no practical cryptographic weakness. You're doing a genuine dictionary or brute force attack. That means wordlist quality is everything.

## The Commands That Actually Worked

### Step 1: Extract the Hash
    
    
    $ zip2john secret_archive.zip > archive.hash
    ver 2.0 efh 5455 efh 7875 secret_archive.zip/flag.txt PKZIP Encr: TS_chk, cmplen=173, decmplen=161, crc=0000 ts=8A75 cs=8a75 type=8
    
    
    
    $ cat archive.hash
    secret_archive.zip/flag.txt:$zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...[hash data]...*$/zip2$:flag.txt:secret_archive.zip::

That hash string is what John will work with. The `$zip2$` prefix tells John this is an AES-encrypted ZIP. The fields encode the salt, key length, verification data, and encrypted content sample.

### Step 2: Build a Targeted Wordlist

Before throwing rockyou.txt at it (which would take hours on CPU), I made a small targeted wordlist from everything I knew about the challenge:
    
    
    $ cat > custom.txt << 'EOF'
    archive
    madness
    secret
    archive_madness
    picoctf
    pico
    ctf
    flag
    secret_archive
    archivemadness
    picoCTF2024
    picoCTF2025
    zip
    password
    EOF

### Step 3: Run John with the Custom Wordlist First
    
    
    $ john --wordlist=custom.txt archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    0g 0:00:00:00 DONE (2024-11-14 14:23:10) 0g/s 933.3p/s 933.3p/s 933.3p/s ..
    Session completed.

No hit. The obvious guesses weren't it. Time for John's mangling rules:
    
    
    $ john --wordlist=custom.txt --rules=Single archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    archive2024       (secret_archive.zip/flag.txt)
    1g 0:00:00:03 DONE (2024-11-14 14:23:47) 0.3030g/s 8381p/s 8381p/s 8381p/s ..
    Use the "--show" option to display all cracked passwords reliably
    Session completed.

There it was. `archive2024`. John's Single rule set had appended "2024" to "archive" — a common CTF password construction pattern. The whole crack took about three seconds once I had the right wordlist.

### Step 4: Verify with --show and Extract the Flag
    
    
    $ john --show archive.hash
    secret_archive.zip/flag.txt:archive2024:flag.txt:secret_archive.zip::
    
    1 password hash cracked, 0 left
    
    
    $ unzip -P archive2024 secret_archive.zip
    Archive:  secret_archive.zip
      inflating: flag.txt
    
    $ cat flag.txt
    picoCTF{z1p_crypt0_1s_n0t_s3cur3_w1th_w34k_p4ssw0rds}

Flag captured. 200 points. I sat there for a moment just staring at the terminal.

## The Moment It Clicked

The real turning point wasn't finding the password. It was going back to that hexdump and finally paying attention to the version byte I'd noticed and ignored. Version `0x33` (decimal 51) in a ZIP local file header is documented in the ZIP specification as indicating AES encryption. I'd seen the number. I hadn't bothered to look up what it meant.

Once I ran `zipinfo -v` and saw "AES Encryption / 256-bit" explicitly printed, everything reorganized in my head. `fcrackzip` failing silently made sense. The need for `zip2john` specifically made sense. The whole challenge was designed around knowing that distinction.

That's the thing about CTF forensics challenges — the file format usually tells you exactly what you need to know. You just have to actually read it.

## Full Trial Process: Every Step I Took

Step| Action| Command| Result| Why it failed / succeeded  
---|---|---|---|---  
1| Identify file type| `file secret_archive.zip`| ZIP archive, v2.0+| Succeeded — confirmed it's a valid ZIP  
2| Attempt extraction| `unzip secret_archive.zip`| Password prompt, then failure| Expected — file is encrypted  
3| Manual password guessing| `unzip -P <word> secret_archive.zip`| 15 attempts, all failed| Failed — password not an obvious keyword  
4| Check metadata| `strings secret_archive.zip`| Only filename visible| Failed — AES content is opaque to strings  
5| Binary inspection| `hexdump -C secret_archive.zip`| Saw version 0x33 byte| Partial — noticed AES indicator but didn't act on it  
6| fcrackzip dictionary attack| `fcrackzip -u -D -p rockyou.txt ...`| Silent, no output| Failed — fcrackzip doesn't support AES-encrypted ZIPs  
7| Confirm encryption type| `zipinfo -v secret_archive.zip`| AES-256 confirmed| Succeeded — this was the key diagnostic step  
8| Extract hash with zip2john| `zip2john secret_archive.zip > archive.hash`| $zip2$ hash generated| Succeeded — correct tool for AES ZIP  
9| John with custom wordlist| `john --wordlist=custom.txt archive.hash`| 0 hits| Failed — exact words not the password  
10| John with Single rules| `john --wordlist=custom.txt --rules=Single archive.hash`| archive2024 cracked in 3s| Succeeded — rule mangling hit the pattern  
11| Extract flag| `unzip -P archive2024 secret_archive.zip`| flag.txt extracted| Succeeded — password confirmed  
  
## Technical Background: Why AES-256 ZIP Is Strong But Passwords Aren't

AES-256 is genuinely secure encryption. The key itself — 256 bits of random data — would take longer than the age of the universe to brute force with any foreseeable hardware. The weakness isn't the algorithm. It's the password.

ZIP's AES implementation (the WinZip AE format) derives the actual encryption key from the password using PBKDF2-HMAC-SHA1 with 1000 iterations. This is deliberately slow to make brute forcing expensive — but 1000 iterations is not especially slow by modern standards. On a modern CPU you can test roughly 50,000–100,000 passwords per second. On a GPU with hashcat and the right module (`-m 13600`), that number climbs to millions per second.

The practical consequence: an AES-256 ZIP is only as secure as its password. "archive2024" is 11 characters with a simple pattern — a dictionary word plus a year. In real-world password auditing, that's classified as weak. It would fall to any competent dictionary attack within minutes even without knowing the target context.

### ZipCrypto vs. AES: Why the Distinction Matters

The older ZipCrypto (traditional PKZIP) encryption is even weaker for a different reason. It has a known-plaintext attack: if you know any 12 bytes of the plaintext (which is trivially true for ZIP files, since the local file header structure is predictable), tools like `pkcrack` can recover the encryption keys directly without knowing the password at all. This attack works regardless of password complexity.

When `zipinfo -v` shows "ZipCrypto" instead of "AES Encryption," your first move should be checking if a plaintext attack is viable — not running john at all. That's a completely different solve path.

### Real-World Context: Where This Shows Up Outside CTF

Password-protected ZIP files appear constantly in malware delivery — phishing emails often include encrypted ZIPs specifically because the password (included in the email body) prevents automated scanning tools from inspecting the payload. Security researchers and incident responders use exactly these tools — `zip2john`, john, hashcat — to recover credentials from seized devices and investigate malware samples. The CTF skill and the professional skill are the same skill.

There are also legitimate recovery scenarios: encrypted archives from former employees, old backup files with forgotten passwords, forensic examination of devices where the user is unavailable. The legal and ethical boundaries matter here — using these tools against files you don't have authorization to access is a different matter entirely.

## zip2john vs. hashcat: When to Use Which

`zip2john` \+ john is the right default when you're on a CPU-only system or when you expect a small, targeted wordlist to work quickly. John's rule sets (especially Single and Jumbo) are excellent at generating contextually plausible password variations from a seed wordlist — exactly what you want in CTF where the challenge setter chose a password related to the theme.

hashcat with `-m 13600` (WinZip) is the right choice when you need raw throughput — exhaustive wordlist runs against rockyou.txt, combinatory attacks, or mask attacks for known-length passwords with unknown character sets. GPU acceleration makes a substantial difference here, but you need to extract the hash first with `zip2john` in either case.
    
    
    # hashcat equivalent (after extracting hash with zip2john)
    $ hashcat -m 13600 -a 0 archive.hash /usr/share/wordlists/rockyou.txt
    hashcat (v6.2.6) starting...
    
    OpenCL API (OpenCL 3.0 PoCL 3.1+debian) - Platform #1 [The pocl project]
    ==========================================================================
    * Device #1: pthread-Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz, 13579/27222 MB (4096 MB allocatable), 4MCU
    
    Minimum password length supported by kernel: 0
    Maximum password length supported by kernel: 256
    
    Hashes: 1 digests; 1 unique digests, 1 unique salts
    ...
    Session..........: hashcat
    Status...........: Cracked
    Hash.Mode........: 13600 (WinZip)
    Hash.Target......: $zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...
    Time.Started.....: Thu Nov 14 14:35:02 2024 (1 min, 43 secs)
    Candidates.#1....: archive2024
    
    Started: Thu Nov 14 14:35:02 2024
    Stopped: Thu Nov 14 14:36:45 2024

## What I'd Do Differently Next Time

The biggest time sink wasn't the failed commands — it was not confirming the encryption type before choosing tools. `zipinfo -v` takes two seconds. If I'd run that before anything else, I would have skipped the entire `fcrackzip` detour and gone straight to `zip2john`.

My updated mental checklist for any encrypted ZIP in CTF:

  1. Run `zipinfo -v` immediately. Identify ZipCrypto vs. AES, and note the AES key size.
  2. If ZipCrypto: consider known-plaintext attack with pkcrack before any password cracking.
  3. If AES: run `zip2john` to extract the hash. Do not touch fcrackzip.
  4. Build a targeted wordlist from every word in the challenge title, description, and filename. Five minutes here saves an hour later.
  5. Run John with that wordlist first (`--rules=Single`). If no hit within 30 seconds, try `--rules=Jumbo`.
  6. If still stuck: rockyou.txt without rules. Then rockyou.txt with rules. Then reconsider whether password cracking is actually the path.
  7. Cap the entire attempt at 15 minutes before stepping back and asking if there's a different angle entirely.



The time limit matters. CTF challenges are designed to be solvable. If you've been running wordlists for 30 minutes with no result, the answer probably isn't "try a bigger wordlist." It's more likely that you're missing something about the challenge design.

## Quick Reference: zip2john Commands
    
    
    # Check encryption type first
    zipinfo -v target.zip
    
    # Extract hash
    zip2john target.zip > target.hash
    
    # Crack with custom wordlist + rules
    john --wordlist=custom.txt --rules=Single target.hash
    
    # Crack with rockyou
    john --wordlist=/usr/share/wordlists/rockyou.txt target.hash
    
    # Show cracked password
    john --show target.hash
    
    # hashcat equivalent (GPU)
    hashcat -m 13600 -a 0 target.hash /usr/share/wordlists/rockyou.txt
    
    # Extract with recovered password
    unzip -P <password> target.zip

## Final Thoughts

picoCTF's _archive_madness_ is a straightforward challenge once you know what you're doing — but I didn't know what I was doing for the first 20 minutes of it. The failure wasn't lack of knowledge about `zip2john`. It was not checking the basic file properties before reaching for tools.

The broader lesson, which I keep relearning in different forms: read the file before attacking it. ZIP headers, ELF headers, PNG chunks, PDF structure — they all tell you what the file actually is and what's relevant about it. Starting with `file` and then a format-specific inspector is worth more than jumping straight to attack tools.

If you found this writeup through a search for zip2john, I hope the failure log was as useful as the working commands. The 20 wasted minutes are the part worth reading.

# zip2john in CTF: How I Finally Cracked an AES-256 Encrypted ZIP in picoCTF

I almost skipped this challenge entirely. A 200-point forensics problem in picoCTF labeled **archive_madness** — just a ZIP file, no hints, no README, nothing but `secret_archive.zip` staring back at me. I figured someone smarter would grab the points while I moved on to something I actually understood. I'm glad I didn't walk away.

This writeup covers every wrong turn I took, the 20-odd minutes I burned on approaches that had no business working, and the exact moment I understood what `zip2john` is actually doing — not just the commands, but the underlying structure it's reading. If you've stared at a password-protected ZIP and wondered where to even begin, this is for you.

## The Challenge: picoCTF archive_madness

The challenge description read: _"We found this archive in a suspicious place. Can you get what's inside? File: secret_archive.zip"_

Category: Forensics. Points: 200. That's medium difficulty in picoCTF terms — not trivial, but not the kind of thing that requires a novel exploit. The ZIP file downloaded cleanly. I ran `file` on it immediately:
    
    
    $ file secret_archive.zip
    secret_archive.zip: Zip archive data, at least v2.0 to extract, one file

Standard ZIP. Nothing exotic. I tried to unzip it:
    
    
    $ unzip secret_archive.zip
    Archive:  secret_archive.zip
    [secret_archive.zip] flag.txt password:
       skipping: flag.txt                incorrect password

Password protected. One file inside: `flag.txt`. This was going to be a password cracking challenge. Fine. I thought I knew exactly what to do.

## Why I Chose the Wrong Approach First (And Wasted 20 Minutes)

My reasoning at the time felt completely logical. The challenge was called _archive_madness_ and the file was named `secret_archive.zip`. In CTF, password hints often hide in plain sight — in the filename, the challenge title, the description. I'd seen that pattern before. So I spent the first chunk of time doing exactly the wrong things for what felt like exactly the right reasons.

### Mistake #1: Manual Password Guessing

I started typing passwords by hand using `unzip -P`. "archive", "madness", "secret", "flag", "pico", "picoctf", "password", "zip" — I tried about fifteen variations. Each one returned the same thing:
    
    
    $ unzip -P archive secret_archive.zip
    Archive:  secret_archive.zip
       skipping: flag.txt                incorrect password
    
    $ unzip -P picoctf secret_archive.zip
    Archive:  secret_archive.zip
       skipping: flag.txt                incorrect password
    
    $ unzip -P secret secret_archive.zip
    Archive:  secret_archive.zip
       skipping: flag.txt                incorrect password

Seven minutes gone. Nothing to show for it except slightly bruised confidence.

### Mistake #2: Running strings on an Encrypted File

Then I made the mistake that anyone who's done more stego than crypto tends to make: I reached for `strings`. The logic was: maybe the password is embedded somewhere unencrypted in the ZIP metadata. ZIP files have a local file header and a central directory — those aren't encrypted even when the content is. Worth checking, right?
    
    
    $ strings secret_archive.zip
    PK
    flag.txt
    

Just the filename and the PK magic bytes. Of course. The metadata gives you the filename and some structural information, but the password isn't stored there in any form a human can read. I knew this intellectually. I checked anyway. That's what CTF desperation looks like.

### Mistake #3: hexdump Without a Plan

After `strings` failed, I opened the file in hexdump. I was looking for... something. I'm not entirely sure what. A cleartext hint? A comment field? I scrolled through hex output for about five minutes before I admitted to myself I had no idea what I was looking for.
    
    
    $ hexdump -C secret_archive.zip | head -20
    00000000  50 4b 03 04 33 00 01 00  63 00 00 00 21 00 75 8a  |PK..3...c...!.u.|
    00000010  a4 6e 00 00 00 00 00 00  00 00 07 00 1c 00 66 6c  |.n............fl|
    00000020  61 67 2e 74 78 74 55 54  09 00 03 5e 2f 84 65 5f  |ag.txtUT...^/.e_|
    00000030  2f 84 65 55 78 0b 00 01  04 e8 03 00 00 04 e8 03  |/.eUx...........|
    00000040  00 00 01 99 07 00 51 4d  41 45 0c 00 19 18 c9 41  |......QMAE.....A|
    00000050  c0 28 4c 57 a2 fe 5c 9e  9a 4d a0 de b8 68 74 61  |.(LW..\..M...hta|

I can read a ZIP header. The `PK\x03\x04` signature, the version needed (`0x33` = 51, meaning AES encryption), the general purpose bit flag (`0x01` = encrypted). That version byte should have told me something important. I noticed it. I didn't act on it. That was the real mistake.

## The Rabbit Hole: fcrackzip and Why It Silently Failed

Around the 20-minute mark I remembered `fcrackzip` existed. I'd used it once before on a simpler challenge with ZipCrypto encryption. I figured it would work here too. This was the longest detour of the whole session — easily 15 minutes of confusion before I understood the problem.
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt secret_archive.zip
    

It ran. And ran. And ran. No output. No progress indicator. After five minutes I hit Ctrl+C and tried a smaller wordlist:
    
    
    $ fcrackzip -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    

Still nothing. I checked if the file was corrupted. I re-downloaded it. Same result. I tried the verbose flag:
    
    
    $ fcrackzip -v -u -D -p /usr/share/wordlists/fasttrack.txt secret_archive.zip
    found file 'flag.txt', (size cp/uc    173/   161, flags 1, chk 8a75)
    

It found the file. It attempted passwords. It never cracked it. This was the moment I finally went back to that `0x33` version byte from the hexdump.

Version 51 in a ZIP local file header means AES encryption — specifically, it means the file uses WinZip's AE-1 or AE-2 AES extension. `fcrackzip` doesn't support AES-encrypted ZIPs. It works fine against the older ZipCrypto (traditional PKZIP encryption), but AES is a different animal entirely. The tool was silently skipping every candidate because it couldn't even verify them correctly. I'd been pouring water into a bucket with no bottom.
    
    
    $ zipinfo -v secret_archive.zip
    Archive:  secret_archive.zip
    There is no zipfile comment.
    
    End-of-central-directory record:
    -------------------------------
    ...
    Central directory entry #1:
    ---------------------------
      flag.txt
      ...
      file security status:                 encrypted
      extended local header:                no
      file last modified on (DOS date/time):  ...
      32-bit CRC value (hex):               00000000
      compressed size:                      173 bytes
      uncompressed size:                    161 bytes
      length of filename:                   8 characters
      ...
      encryption method:                    AES Encryption
      AES key size:                         256-bit

AES-256. That explained everything. `fcrackzip` was never going to work. I needed a tool that understood this encryption format — and that's exactly what `zip2john` is built for.

## What zip2john Actually Does (And Why It Matters)

Before I show the commands that worked, it's worth understanding the mechanism. `zip2john` is part of the John the Ripper suite. Its job is not to crack passwords — it's to extract a crackable hash from the ZIP file's encryption metadata.

For AES-encrypted ZIPs (the WinZip AE format), the ZIP file stores a salt value and a verification value in the local file header. These are derived from the password using PBKDF2-HMAC-SHA1. `zip2john` reads these values and formats them into a hash string that John the Ripper (or hashcat, with the right module) can work with. The hash includes the salt, the number of iterations, and the verification bytes — everything needed to test a candidate password without actually decrypting the file contents.

This matters for a non-obvious reason: AES-256 ZIP encryption is actually strong. Unlike the old ZipCrypto (which has known plaintext attacks that can bypass brute force entirely), AES-256 ZIP has no practical cryptographic weakness. You're doing a genuine dictionary or brute force attack. That means wordlist quality is everything.

## The Commands That Actually Worked

### Step 1: Extract the Hash
    
    
    $ zip2john secret_archive.zip > archive.hash
    ver 2.0 efh 5455 efh 7875 secret_archive.zip/flag.txt PKZIP Encr: TS_chk, cmplen=173, decmplen=161, crc=0000 ts=8A75 cs=8a75 type=8
    
    
    
    $ cat archive.hash
    secret_archive.zip/flag.txt:$zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...[hash data]...*$/zip2$:flag.txt:secret_archive.zip::

That hash string is what John will work with. The `$zip2$` prefix tells John this is an AES-encrypted ZIP. The fields encode the salt, key length, verification data, and encrypted content sample.

### Step 2: Build a Targeted Wordlist

Before throwing rockyou.txt at it (which would take hours on CPU), I made a small targeted wordlist from everything I knew about the challenge:
    
    
    $ cat > custom.txt << 'EOF'
    archive
    madness
    secret
    archive_madness
    picoctf
    pico
    ctf
    flag
    secret_archive
    archivemadness
    picoCTF2024
    picoCTF2025
    zip
    password
    EOF

### Step 3: Run John with the Custom Wordlist First
    
    
    $ john --wordlist=custom.txt archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    0g 0:00:00:00 DONE (2024-11-14 14:23:10) 0g/s 933.3p/s 933.3p/s 933.3p/s ..
    Session completed.

No hit. The obvious guesses weren't it. Time for John's mangling rules:
    
    
    $ john --wordlist=custom.txt --rules=Single archive.hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
    Cost 1 (HMAC size) is 173 for all loaded hashes
    Will run 4 OpenMP threads
    Press 'q' or Ctrl-C to abort, almost any other key for status
    archive2024       (secret_archive.zip/flag.txt)
    1g 0:00:00:03 DONE (2024-11-14 14:23:47) 0.3030g/s 8381p/s 8381p/s 8381p/s ..
    Use the "--show" option to display all cracked passwords reliably
    Session completed.

There it was. `archive2024`. John's Single rule set had appended "2024" to "archive" — a common CTF password construction pattern. The whole crack took about three seconds once I had the right wordlist.

### Step 4: Verify with --show and Extract the Flag
    
    
    $ john --show archive.hash
    secret_archive.zip/flag.txt:archive2024:flag.txt:secret_archive.zip::
    
    1 password hash cracked, 0 left
    
    
    $ unzip -P archive2024 secret_archive.zip
    Archive:  secret_archive.zip
      inflating: flag.txt
    
    $ cat flag.txt
    picoCTF{z1p_crypt0_1s_n0t_s3cur3_w1th_w34k_p4ssw0rds}

Flag captured. 200 points. I sat there for a moment just staring at the terminal.

## The Moment It Clicked

The real turning point wasn't finding the password. It was going back to that hexdump and finally paying attention to the version byte I'd noticed and ignored. Version `0x33` (decimal 51) in a ZIP local file header is documented in the ZIP specification as indicating AES encryption. I'd seen the number. I hadn't bothered to look up what it meant.

Once I ran `zipinfo -v` and saw "AES Encryption / 256-bit" explicitly printed, everything reorganized in my head. `fcrackzip` failing silently made sense. The need for `zip2john` specifically made sense. The whole challenge was designed around knowing that distinction.

That's the thing about CTF forensics challenges — the file format usually tells you exactly what you need to know. You just have to actually read it.

## Full Trial Process: Every Step I Took

Step| Action| Command| Result| Why it failed / succeeded  
---|---|---|---|---  
1| Identify file type| `file secret_archive.zip`| ZIP archive, v2.0+| Succeeded — confirmed it's a valid ZIP  
2| Attempt extraction| `unzip secret_archive.zip`| Password prompt, then failure| Expected — file is encrypted  
3| Manual password guessing| `unzip -P <word> secret_archive.zip`| 15 attempts, all failed| Failed — password not an obvious keyword  
4| Check metadata| `strings secret_archive.zip`| Only filename visible| Failed — AES content is opaque to strings  
5| Binary inspection| `hexdump -C secret_archive.zip`| Saw version 0x33 byte| Partial — noticed AES indicator but didn't act on it  
6| fcrackzip dictionary attack| `fcrackzip -u -D -p rockyou.txt ...`| Silent, no output| Failed — fcrackzip doesn't support AES-encrypted ZIPs  
7| Confirm encryption type| `zipinfo -v secret_archive.zip`| AES-256 confirmed| Succeeded — this was the key diagnostic step  
8| Extract hash with zip2john| `zip2john secret_archive.zip > archive.hash`| $zip2$ hash generated| Succeeded — correct tool for AES ZIP  
9| John with custom wordlist| `john --wordlist=custom.txt archive.hash`| 0 hits| Failed — exact words not the password  
10| John with Single rules| `john --wordlist=custom.txt --rules=Single archive.hash`| archive2024 cracked in 3s| Succeeded — rule mangling hit the pattern  
11| Extract flag| `unzip -P archive2024 secret_archive.zip`| flag.txt extracted| Succeeded — password confirmed  
  
## Technical Background: Why AES-256 ZIP Is Strong But Passwords Aren't

AES-256 is genuinely secure encryption. The key itself — 256 bits of random data — would take longer than the age of the universe to brute force with any foreseeable hardware. The weakness isn't the algorithm. It's the password.

ZIP's AES implementation (the WinZip AE format) derives the actual encryption key from the password using PBKDF2-HMAC-SHA1 with 1000 iterations. This is deliberately slow to make brute forcing expensive — but 1000 iterations is not especially slow by modern standards. On a modern CPU you can test roughly 50,000–100,000 passwords per second. On a GPU with hashcat and the right module (`-m 13600`), that number climbs to millions per second.

The practical consequence: an AES-256 ZIP is only as secure as its password. "archive2024" is 11 characters with a simple pattern — a dictionary word plus a year. In real-world password auditing, that's classified as weak. It would fall to any competent dictionary attack within minutes even without knowing the target context.

### ZipCrypto vs. AES: Why the Distinction Matters

The older ZipCrypto (traditional PKZIP) encryption is even weaker for a different reason. It has a known-plaintext attack: if you know any 12 bytes of the plaintext (which is trivially true for ZIP files, since the local file header structure is predictable), tools like `pkcrack` can recover the encryption keys directly without knowing the password at all. This attack works regardless of password complexity.

When `zipinfo -v` shows "ZipCrypto" instead of "AES Encryption," your first move should be checking if a plaintext attack is viable — not running john at all. That's a completely different solve path.

### Real-World Context: Where This Shows Up Outside CTF

Password-protected ZIP files appear constantly in malware delivery — phishing emails often include encrypted ZIPs specifically because the password (included in the email body) prevents automated scanning tools from inspecting the payload. Security researchers and incident responders use exactly these tools — `zip2john`, john, hashcat — to recover credentials from seized devices and investigate malware samples. The CTF skill and the professional skill are the same skill.

There are also legitimate recovery scenarios: encrypted archives from former employees, old backup files with forgotten passwords, forensic examination of devices where the user is unavailable. The legal and ethical boundaries matter here — using these tools against files you don't have authorization to access is a different matter entirely.

## zip2john vs. hashcat: When to Use Which

`zip2john` \+ john is the right default when you're on a CPU-only system or when you expect a small, targeted wordlist to work quickly. John's rule sets (especially Single and Jumbo) are excellent at generating contextually plausible password variations from a seed wordlist — exactly what you want in CTF where the challenge setter chose a password related to the theme.

hashcat with `-m 13600` (WinZip) is the right choice when you need raw throughput — exhaustive wordlist runs against rockyou.txt, combinatory attacks, or mask attacks for known-length passwords with unknown character sets. GPU acceleration makes a substantial difference here, but you need to extract the hash first with `zip2john` in either case.
    
    
    # hashcat equivalent (after extracting hash with zip2john)
    $ hashcat -m 13600 -a 0 archive.hash /usr/share/wordlists/rockyou.txt
    hashcat (v6.2.6) starting...
    
    OpenCL API (OpenCL 3.0 PoCL 3.1+debian) - Platform #1 [The pocl project]
    ==========================================================================
    * Device #1: pthread-Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz, 13579/27222 MB (4096 MB allocatable), 4MCU
    
    Minimum password length supported by kernel: 0
    Maximum password length supported by kernel: 256
    
    Hashes: 1 digests; 1 unique digests, 1 unique salts
    ...
    Session..........: hashcat
    Status...........: Cracked
    Hash.Mode........: 13600 (WinZip)
    Hash.Target......: $zip2$*0*3*0*7a3d9e2f1b8c4a56*8a75*28*...
    Time.Started.....: Thu Nov 14 14:35:02 2024 (1 min, 43 secs)
    Candidates.#1....: archive2024
    
    Started: Thu Nov 14 14:35:02 2024
    Stopped: Thu Nov 14 14:36:45 2024

## What I'd Do Differently Next Time

The biggest time sink wasn't the failed commands — it was not confirming the encryption type before choosing tools. `zipinfo -v` takes two seconds. If I'd run that before anything else, I would have skipped the entire `fcrackzip` detour and gone straight to `zip2john`.

My updated mental checklist for any encrypted ZIP in CTF:

  1. Run `zipinfo -v` immediately. Identify ZipCrypto vs. AES, and note the AES key size.
  2. If ZipCrypto: consider known-plaintext attack with pkcrack before any password cracking.
  3. If AES: run `zip2john` to extract the hash. Do not touch fcrackzip.
  4. Build a targeted wordlist from every word in the challenge title, description, and filename. Five minutes here saves an hour later.
  5. Run John with that wordlist first (`--rules=Single`). If no hit within 30 seconds, try `--rules=Jumbo`.
  6. If still stuck: rockyou.txt without rules. Then rockyou.txt with rules. Then reconsider whether password cracking is actually the path.
  7. Cap the entire attempt at 15 minutes before stepping back and asking if there's a different angle entirely.



The time limit matters. CTF challenges are designed to be solvable. If you've been running wordlists for 30 minutes with no result, the answer probably isn't "try a bigger wordlist." It's more likely that you're missing something about the challenge design.

## Quick Reference: zip2john Commands
    
    
    # Check encryption type first
    zipinfo -v target.zip
    
    # Extract hash
    zip2john target.zip > target.hash
    
    # Crack with custom wordlist + rules
    john --wordlist=custom.txt --rules=Single target.hash
    
    # Crack with rockyou
    john --wordlist=/usr/share/wordlists/rockyou.txt target.hash
    
    # Show cracked password
    john --show target.hash
    
    # hashcat equivalent (GPU)
    hashcat -m 13600 -a 0 target.hash /usr/share/wordlists/rockyou.txt
    
    # Extract with recovered password
    unzip -P <password> target.zip

## Final Thoughts

picoCTF's _archive_madness_ is a straightforward challenge once you know what you're doing — but I didn't know what I was doing for the first 20 minutes of it. The failure wasn't lack of knowledge about `zip2john`. It was not checking the basic file properties before reaching for tools.

The broader lesson, which I keep relearning in different forms: read the file before attacking it. ZIP headers, ELF headers, PNG chunks, PDF structure — they all tell you what the file actually is and what's relevant about it. Starting with `file` and then a format-specific inspector is worth more than jumping straight to attack tools.

If you found this writeup through a search for zip2john, I hope the failure log was as useful as the working commands. The 20 wasted minutes are the part worth reading.
