---
title: "Crack the Power picoCTF Writeup"
published: true
description: "
picoCTF Crack the Power is an RSA cryptography challenge where the key insight isn’t just knowing the attack — it’s reading the numbers closely enough to "
tags: ctf, picoctf, cryptography, security
canonical_url: https://alsavaudomila.com/crack-the-power/
---

picoCTF Crack the Power is an RSA cryptography challenge where the key insight isn't just knowing the attack — it's reading the numbers closely enough to know which version of the attack you're dealing with before you write a single line of code. I didn't do that at first, and it cost me about 40 minutes.

## The Challenge

The challenge provides a `message.txt` file with three values:
    
    
    n = 557838419289674163475248835907844438905982230212026010035453573...
    e = 20
    c = 640637430810406857500566702096274079797813783756285040245597839...369140625

RSA with `e = 20`. The hint says the primes are large, and factorization is not the intended approach.

## What I Tried First (and Why It Was Wrong)

Seeing `e = 20` and "large primes," I immediately thought: low-exponent RSA attack. I knew two variants exist — the direct root case and the k-iteration case — but instead of figuring out which one applied here, I just reached for the general solution that handles both:
    
    
    # olddecript.py — my first attempt, works but misses the point
    from Crypto.Util.number import *
    import gmpy2
    
    i = 1
    while True:
        m, ok = gmpy2.iroot(c + i * N, e)
        if ok:
            break
        i += 1
    print(long_to_bytes(m))

This ran and produced the flag at `i = 1`, meaning the loop exited on the first iteration. Which meant… `i = 1` wasn't even needed. The loop was unnecessary. I had written the mini-rsa solution for a problem that didn't require it.

That bothered me enough to go back and figure out why `i = 0` would have worked just as well — and that's where the actual learning happened.

## Reading the Numbers Before Writing Code

The two cases of RSA low-exponent attack differ by one condition:

  * **No wrapping** (`m^e < N`): the modular reduction never fires, so `c = m^e` exactly. Direct iroot works.
  * **Wrapping** (`m^e ≥ N`): `c = m^e mod N`, meaning `m^e = c + k·N` for some small k. You need to iterate.



Looking at the ciphertext again:
    
    
    c = ...369140625

The last 9 digits are `369140625`. Now check what happens when you compute `5^20`:
    
    
    >>> 5**20
    95367431640625

The last 9 digits of `5^20` are `431640625`. And `369140625` ends with the same suffix pattern — multiples of powers of 5 propagate their trailing digit structure predictably. This is a hint that `c` is a raw integer power of a plaintext that ends in 5, not a reduced modular value. If the modular reduction had fired, those trailing digits would look arbitrary.

The cleaner check: if `m^e < N`, then `c < N` should hold. Comparing the number of digits in c versus n in this challenge confirms c is smaller — no wrapping occurred.

## The Actual Solution
    
    
    import gmpy2
    from Crypto.Util.number import long_to_bytes
    
    e = 20
    c = 640637430810406857500566702096274079797813783756285040245597839...369140625
    
    m, exact = gmpy2.iroot(c, e)
    
    print("Exact:", exact)
    print(long_to_bytes(m))
    
    
    $ python3 decript.py
    Exact: True
    b'picoCTF{t1ny_e_2fe2da79}'

`exact = True` confirms it: `c` was a perfect 20th power, no wrapping, no iteration needed.

Flag: **picoCTF{t1ny_e_2fe2da79}**

## How to Distinguish the Two Cases Quickly

Check| No wrapping (this challenge)| Wrapping (mini-rsa pattern)  
---|---|---  
Compare digit count of c vs n| c has fewer digits than n| c has similar digit count as n  
gmpy2.iroot(c, e) exact flag| True| False  
Required technique| Direct iroot| k-iteration: iroot(c + k·N, e)  
Loop exits at| k=0 (no loop needed)| k=1 or small value  
  
The fastest approach in practice: try `gmpy2.iroot(c, e)` first and check the exact flag. If it's `True`, you're done. If it's `False`, you need the k-iteration loop. No need to analyze trailing digits beforehand.

## Why This Matters Beyond CTF

The "no wrapping" case happens in practice when RSA is implemented without proper padding and the message is short relative to the modulus. A 20-byte flag encrypted with `e = 20` and a 2048-bit modulus will almost always satisfy `m^e < N`, making the encryption trivially reversible. This is why modern RSA uses OAEP or PKCS#1 v1.5 padding — padding inflates the effective message size to make `m^e > N` guaranteed, forcing modular reduction and making direct root extraction impossible.

## Further Reading

Here are related articles from [alsavaudomila.com](<https://alsavaudomila.com/>) that complement this challenge:

For the wrapping case — where `m^e ≥ N` and direct iroot fails — the [Mini RSA writeup](<https://alsavaudomila.com/mini-rsa-picoctf-writeup/>) covers the k-iteration technique in detail, including why floating-point cube root methods silently fail and how gmpy2 handles it correctly.

For a broader overview of RSA attack categories in CTF, [CTF Forensics Tools: The Ultimate Guide for Beginners](<https://alsavaudomila.com/ctf-forensics-tools/>) includes the cryptography tool decision flow alongside forensics tools.
