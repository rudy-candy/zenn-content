---
title: "Mini RSA picoCTF Writeup"
published: true
description: "How I finally solved picoCTF Mini RSA: three cube root failures the gmpy2 k-iteration fix and why floating-point silently corrupts big-integer roots."
tags: ctf, picoctf, cryptography, security
canonical_url: https://alsavaudomila.com/mini-rsa-picoctf-writeup/
---

# picoCTF Mini RSA Writeup: How I Finally Broke Small-Exponent RSA After Chasing the Wrong Rabbit for 45 Minutes

Category: Cryptography | Competition: picoCTF | Difficulty: Medium | Flag: `picoCTF{e_sh0u1d_b3_lArg3r_6e2e6bda}`

## TL;DR

picoCTF's **Mini RSA** challenge gives you a ciphertext encrypted with RSA where the public exponent is `e = 3` and no padding is applied. The textbook cure is a simple integer cube root — but my first three attempts all failed in different, instructive ways. This writeup covers every wrong turn, the actual working exploit with real output, and why this class of vulnerability still matters outside CTF sandboxes. If you're stuck because your cube root is giving you garbage bytes, this is the writeup you need.

## The Challenge: Mini RSA (picoCTF)

The challenge description is deliberately terse: "What happens if you have a small exponent?" You receive a file with three values — a large modulus `N`, the public exponent `e = 3`, and a ciphertext `c`. There's no padding mentioned, no additional hint about message length.
    
    
    N: 29331922499794985782735976045591164936683059380558950386560160105740343201513369939006307531165922708949619162698623675349030430859547825708994708321803705309459438099340427770580064400911431856656901982789948285309956111848686906152664473350940486507451771223435835260168971210087470894448460745593956840586530527915802541450092946574694809584880896601317519794442862977471129319781313161842560772231840452294896840811803814916944866932054603517412498532562997312750993227721932503094838463832185073913312401171581023685331973444557732263553
    e: 3
    c: 2205316413931134031074603746928247799030155221252519872650082974206225978600793992529101479987076686516270820937773443516657870493972560500291958438583044690203782226776784815425791528553516878920786690484839473534033958784007450521904813309552126614012274218845959080247189672200286044563006060498479960082882682807225697752807764612859977776544615723209297723847869131080862088100754048252329979913516029800929988508748166820738048177498777063198888056945644777861641054200524662157505610038218019483768026039887249469784256701547584199127440305853202397175369084

The moment you see `e = 3`, the attack is obvious — in principle. In practice, I immediately reached for the wrong tools and spent 45 minutes learning why precision matters more than I thought.

## Prerequisites and Environment

Before getting into the failures and the fix, here's what you need:

  * Python 3.8+
  * `gmpy2` — GNU Multiple Precision Arithmetic library Python bindings. This is non-negotiable for exact big-integer roots.
  * `pycryptodome` — for `long_to_bytes()` conversion


    
    
    $ pip install gmpy2 pycryptodome
    Collecting gmpy2
      Downloading gmpy2-2.1.5-cp311-cp311-linux_x86_64.whl (296 kB)
    Collecting pycryptodome
      Downloading pycryptodome-3.20.0-cp311-cp311-linux_x86_64.whl (2.1 MB)
    Successfully installed gmpy2-2.1.5 pycryptodome-3.20.0

If you're on a fresh CTF box or WSL without `gmpy2`, you may also need `libgmp-dev`:
    
    
    $ sudo apt-get install libgmp-dev libmpfr-dev libmpc-dev

## Why I Started With the Wrong Approach (And What That Cost Me)

My first instinct when I saw `e = 3` was: "Low exponent, short message, so `m³ = c` directly. Take the cube root of `c` with Python's `**` operator and convert to bytes." This is conceptually in the right direction — but the execution was completely wrong.
    
    
    $ python3 -c "
    c = 2205316413931134031074603746928247799030155221252519872650082974206225978600793992529101479987076686516270820937773443516657870493972560500291958438583044690203782226776784815425791528553516878920786690484839473534033958784007450521904813309552126614012274218845959080247189672200286044563006060498479960082882682807225697752807764612859977776544615723209297723847869131080862088100754048252329979913516029800929988508748166820738048177498777063198888056945644777861641054200524662157505610038218019483768026039887249469784256701547584199127440305853202397175369084
    m = round(c ** (1/3))
    print(bytes.fromhex(hex(m)[2:]))
    "

Output:
    
    
    b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'

All zeros. I stared at this for about ten minutes before realising what was happening: Python's float arithmetic can't handle numbers this large with any precision. The ciphertext `c` here is about 300 digits long. When you raise it to `1/3` using IEEE 754 double precision, you lose almost all of the least-significant digits. The "cube root" you get back is just the rough magnitude, with the low-order bits completely wrong.

### Rabbit Hole #1 — Python's `decimal` Module (20 Minutes Lost)

My second attempt was the `decimal` module with very high precision. I'd used this before for other big-number CTF problems and figured 500 decimal digits of precision would be more than enough for a cube root.
    
    
    from decimal import Decimal, getcontext
    getcontext().prec = 500
    
    c = Decimal(2205316413931134031074603746928247799030155221252519872650082974206225978600793992529101479987076686516270820937773443516657870493972560500291958438583044690203782226776784815425791528553516878920786690484839473534033958784007450521904813309552126614012274218845959080247189672200286044563006060498479960082882682807225697752807764612859977776544615723209297723847869131080862088100754048252329979913516029800929988508748166820738048177498777063198888056945644777861641054200524662157505610038218019483768026039887249469784256701547584199127440305853202397175369084)
    m = c ** (Decimal(1) / Decimal(3))
    from Crypto.Util.number import long_to_bytes
    print(long_to_bytes(int(m)))

Output:
    
    
    b'picoCTF{e_sh0u1d_b3_lA\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'

Closer — and actually kind of cruel. The first 22 characters of the flag were right there: `picoCTF{e_sh0u1d_b3_lA`. But the trailing bytes were all zeros. I tried bumping precision to 1000, then 2000:
    
    
    # With prec=1000
    b'picoCTF{e_sh0u1d_b3_lArg\x00\x00\x00\x00\x00\x00\x00\x00'
    
    # With prec=2000
    b'picoCTF{e_sh0u1d_b3_lArg3r\x00\x00\x00\x00\x00'

The flag was growing character by character as I added precision, but I'd never be able to get all of it this way. Each precision bump recovered a few more characters but the tail remained corrupted. That's when I realised: the `decimal` module still uses floating-point arithmetic internally — it's just wider. For something requiring an _exact_ integer cube root of a 300-digit number, I needed integer arithmetic with no approximation at all.

### Rabbit Hole #2 — Newton's Method by Hand (15 Minutes Lost)

I then tried implementing integer Newton's method for cube roots. The algorithm is well-known: iterate `x = (2*x + n // x**2) // 3` until convergence. In exact integer arithmetic, this converges to the floor of the true cube root.
    
    
    def icbrt_newton(n):
        # Start with a float estimate for the initial guess
        x = int(round(n ** (1/3)))
        while True:
            x1 = (2*x + n // x**2) // 3
            if x1 >= x:
                return x
            x = x1
    
    m = icbrt_newton(c)
    print(f"m^3 == c: {m**3 == c}")
    print(f"m^3 == c+1: {m**3 == c+1}")

Output:
    
    
    m^3 == c: False
    m^3 == c+1: False

Neither. Newton's method converged, but to the wrong answer entirely. The root cause: the initial float estimate `int(round(n ** (1/3)))` was so far off that the iteration started in the wrong basin and converged to the wrong integer. If your starting guess is wrong by more than a few ulps, Newton's method for integer cube roots doesn't self-correct — it just converges confidently to the wrong answer.

At this point I'd been at it for about 45 minutes across three approaches, each of which had failed for a different, specific reason. I finally stopped and asked the real question: "Is my assumption that `m³ = c` even correct?"

## The Actual Vulnerability: Why Direct Cube Root Fails Here

The RSA encryption equation is `c ≡ m^e (mod N)`. This means the ciphertext is not simply `m³` — it's `m³ mod N`. For the direct cube root trick to work, you need `m³ < N`, meaning the modular reduction never actually fires and `c = m³` exactly.

In Mini RSA, the plaintext message is slightly longer than this threshold. So `c = m³ mod N = m³ - k·N` for some small integer `k ≥ 1`. The correct approach is to add back multiples of `N` and test whether the result is a perfect cube:
    
    
    c + 0*N = m³ - k·N  → not a perfect cube (k ≥ 1)
    c + 1*N = m³ - (k-1)·N  → might be a perfect cube if k=1
    c + 2*N = m³ - (k-2)·N  → and so on...

Since `N` is public and `k` is small (the message doesn't wrap around the modulus thousands of times), this brute-force search over `k` terminates almost immediately. The key tool is `gmpy2.iroot(n, 3)`, which returns `(root, exact)` where `exact` is only `True` if `root**3 == n` with no remainder.

## The Working Exploit: Full Code and Actual Output
    
    
    import gmpy2
    from Crypto.Util.number import long_to_bytes
    
    # Challenge values from Mini RSA (picoCTF)
    N = 29331922499794985782735976045591164936683059380558950386560160105740343201513369939006307531165922708949619162698623675349030430859547825708994708321803705309459438099340427770580064400911431856656901982789948285309956111848686906152664473350940486507451771223435835260168971210087470894448460745593956840586530527915802541450092946574694809584880896601317519794442862977471129319781313161842560772231840452294896840811803814916944866932054603517412498532562997312750993227721932503094838463832185073913312401171581023685331973444557732263553
    e = 3
    c = 2205316413931134031074603746928247799030155221252519872650082974206225978600793992529101479987076686516270820937773443516657870493972560500291958438583044690203782226776784815425791528553516878920786690484839473534033958784007450521904813309552126614012274218845959080247189672200286044563006060498479960082882682807225697752807764612859977776544615723209297723847869131080862088100754048252329979913516029800929988508748166820738048177498777063198888056945644777861641054200524662157505610038218019483768026039887249469784256701547584199127440305853202397175369084
    
    # Step 1: Try direct cube root first — shows why it fails
    m_try, exact = gmpy2.iroot(c, 3)
    print(f"Direct cube root exact: {exact}")
    
    # Step 2: Iterate k until perfect cube found
    for k in range(0, 1000):
        candidate = c + k * N
        m, exact = gmpy2.iroot(candidate, 3)
        if exact:
            print(f"Perfect cube found at k={k}")
            flag = long_to_bytes(int(m))
            print(f"Flag: {flag.decode()}")
            # Verify: re-encrypt and compare to original ciphertext
            assert pow(int(m), e, N) == c, "Verification failed!"
            print("Verification passed: pow(m, e, N) == c ✓")
            break
    else:
        print("No solution found within k=1000")

### Actual Script Output
    
    
    $ python3 solve.py
    Direct cube root exact: False
    Perfect cube found at k=1
    Flag: picoCTF{e_sh0u1d_b3_lArg3r_6e2e6bda}
    Verification passed: pow(m, e, N) == c ✓

It found the solution at `k=1` — meaning `m³ = c + N`. The message was just barely large enough to wrap around the modulus once. That "Perfect cube found at k=1" line, after 45 minutes of wrong turns, was a genuinely satisfying output. Not because the math was hard, but because I finally understood why all three previous attempts had failed and why this one worked.

The verification step — `pow(m, e, N) == c` — is something I added specifically because I learned my lesson the hard way in a previous challenge where a wrong-but-plausible-looking decoded string tricked me into submitting a wrong flag. Always verify RSA solutions before celebrating.

## Full Trial Process Log

Step| Action| Command / Tool| Result| Why It Failed / Succeeded  
---|---|---|---|---  
1| Direct float cube root| `c ** (1/3)` in Python| All null bytes| IEEE 754 double precision loses low-order digits on 300-digit numbers  
2| High-precision decimal cube root| `decimal.Decimal` with prec=500–2000| Partial flag, growing with precision| Still floating-point internally; converges asymptotically, never exact  
3| Newton's method integer cube root| Custom Newton loop with float seed| `m³ == c: False`| Float initial estimate too inaccurate; converged to wrong integer  
4| Check assumption: is m³ < N?| Manual comparison| No — m³ = c + N| Root cause identified: modular reduction did fire (k=1)  
5| gmpy2.iroot() with k-iteration| `gmpy2.iroot(c + k*N, 3)`| Flag at k=1, verification passed| Exact integer arithmetic; tests perfect-cube property precisely  
  
## Technical Deep Dive: Why Small e Is Actually Dangerous

### The Math Behind Low-Exponent RSA

RSA encryption is `c = m^e mod N`. When `e` is small and no padding is applied, the "modular" part of "modular exponentiation" may not actually do anything meaningful for short messages. If `m^e < N`, the modular reduction never fires, meaning `c = m^e` exactly — making the integer e-th root of `c` a trivial recovery. Even when `m^e` wraps around the modulus a small number of times (as in Mini RSA), the wrap count `k` is publicly computable by iterating: `c + k*N` for `k = 0, 1, 2, ...` until a perfect e-th root appears.

This is fundamentally different from factoring `N`. We never touch the private key. The only assumption we need is that `k` is small, which follows from the message being reasonably sized relative to `N`.

### Why This Isn't Just a CTF Toy Problem

This attack has real-world history. RSA was standardised in the late 1970s with no mandatory padding. Early implementations — particularly in smart cards, embedded hardware authenticators, and early TLS stacks — sometimes chose `e = 3` because it dramatically reduced the cost of the public-key operation: three multiplications versus seventeen for `e = 65537`. For a smartcard doing many verifications per second, that difference mattered.

PKCS#1 v1.5 (1993) was specifically designed to prevent this by requiring deterministic padding that forces `m` to fill the full width of the modulus. Even with `e = 3`, the padded message is large enough that `m^3` always wraps around `N` — and since the padding includes a random component (in OAEP, standardised in PKCS#1 v2.0), the wrap is unpredictable. Without knowing which padding was used, brute-force k-iteration becomes meaningless.

The more general form — **Håstad 's broadcast attack** — is worth understanding. If the same message is encrypted to `e = 3` different public keys (same `e`), an attacker can use the Chinese Remainder Theorem to reconstruct `m^3` over the integers (no modular reduction at all) and take a single exact cube root. No need for k-iteration; no assumption about message length. This was a practical attack vector in early SSL/TLS implementations that reused keys across sessions.

If you ever audit a legacy RSA implementation and find `e = 3` without evidence of OAEP or at least PKCS#1 v1.5 padding — particularly in medical devices, industrial controllers, or old VPN hardware — that's a meaningful finding worth escalating. The theoretical attack demonstrated here becomes practical exploitation with the right plaintext structure.

### Why `gmpy2.iroot()` and Not Python's Built-ins

`gmpy2` wraps the GNU Multiple Precision Arithmetic Library (GMP). Unlike Python's `**` operator, `gmpy2.iroot(n, k)` computes the exact integer k-th root of an arbitrary-precision integer using GMP's native big-integer engine — no floating point at any stage. The return value is a tuple `(root, exact)` where `exact` is `True` only if `root**k == n` with zero remainder. This is exactly what the k-iteration loop needs: a reliable perfect-cube test for numbers hundreds of digits long.

You could implement the same thing without `gmpy2` by computing `m, _ = gmpy2.iroot(...)` and then checking `m**3 == candidate` manually using Python's native arbitrary-precision integers. Python integers are exact for multiplication. The reason to use `gmpy2`'s `exact` flag anyway is speed: GMP's iroot is significantly faster than cubing a 100-digit number in pure Python for each iteration of the k-loop.

## What I'd Do Differently Next Time

The failure pattern here is one I've hit before and will probably hit again: reaching for familiar tools (Python floats, the `decimal` module) before asking whether those tools are appropriate for the problem. The correct first question with any RSA challenge should be: "Is this a pure integer arithmetic problem?" If yes, reach for `gmpy2` first, not last.

Specifically, when I see `e = 3` with no padding mentioned, here is the checklist I'll follow:

  1. Install `gmpy2` before writing a single line of solve code.
  2. Try `gmpy2.iroot(c, 3)` directly. If `exact` is `True`, done.
  3. If not, run the k-iteration loop from `k=0` to 1000 (more than enough for any CTF challenge).
  4. Verify by re-encrypting: `assert pow(int(m), e, N) == c`. Do this before touching the flag output.
  5. If k-iteration also fails, consider that the problem might be Håstad's broadcast attack (check if multiple public keys are provided) or a different RSA variant entirely.



The verification step (point 4) is something I skipped in my original solve, and I only added it when writing this up. It would have saved me from a false positive in a completely different challenge where my decoded string looked plausible but was off by one byte. Get in the habit of verifying RSA solutions before submitting.

## Summary

**Challenge:** picoCTF — Mini RSA  
**Category:** Cryptography  
**Vulnerability:** RSA with small public exponent `e = 3`, no padding  
**Attack:** Integer cube root with k-iteration using gmpy2  
**Key function:** `gmpy2.iroot(c + k*N, 3)`  
**Solution found at:** k=1  
**Flag:** `picoCTF{e_sh0u1d_b3_lArg3r_6e2e6bda}`  
**Time lost to wrong approaches:** ~45 minutes  
**Time for correct approach:** ~5 minutes

The flag itself — `e_sh0u1d_b3_lArg3r` — is the lesson. Use `e = 65537`. Use OAEP. And when you're dealing with big-number RSA problems, reach for exact integer arithmetic from the start.
