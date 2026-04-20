---
title: "Includes picoCTF Writeup"
published: true
description: "
HTML source code review is one of those skills that sounds obvious in hindsight, yet I spent a full 30+ minutes chasing the wrong rabbit before I figured "
tags: ctf, picoctf, webdev, security
canonical_url: https://alsavaudomila.com/includes-picoctf-writeup/
---

HTML source code review is one of those skills that sounds obvious in hindsight, yet I spent a full 30+ minutes chasing the wrong rabbit before I figured that out. This is my writeup for the **picoCTF "Includes"** challenge — a Web Exploitation Easy problem that taught me more about my own assumptions than it did about hacking. If you landed here hoping to find how a hidden flag lives inside CSS and JavaScript comments, you're in the right place.

## Challenge Overview: picoCTF "Includes"

**Competition:** picoCTF  
**Challenge Name:** Includes  
**Category:** Web Exploitation  
**Difficulty:** Easy  
**Points:** 100

The challenge drops you at a single-page website with a button and some decorative text. The objective: find the flag. No hints. No file downloads. Just a URL.

## My First Instinct — And Why It Was Wrong

I'll be honest: the moment I saw a web challenge labeled "Easy," my brain jumped straight to SQL injection. I don't know why — probably because SQL injection is the first thing I learned and it feels like a reliable default. I opened the page, saw a text input area, and started typing.
    
    
    ' OR 1=1 --
    " OR "1"="1
    admin'--
    ' UNION SELECT null--

Nothing. The button didn't even submit anything to a visible endpoint. I checked the Network tab in DevTools — no POST request firing at all. The form was cosmetic, or at least it wasn't wired to a backend I could see.

Then I pivoted to XSS, reasoning maybe there was a reflection point somewhere.
    
    
    <script>alert(1)</script>
    <img src=x onerror=alert(1)>
    javascript:alert(document.cookie)

Also nothing. The page just sat there looking smug. No reflected output, no pop-ups. I started to feel the slow creep of frustration — you know the feeling when you're trying three things at once and none of them are landing.

I wasted about 20 minutes here before stepping back and asking myself: _why am I jumping to SQL injection on a page that probably doesn 't even have a database?_ I had pattern-matched to "web challenge = injection" without actually reading what was in front of me.

## Rabbit Hole Log — 30+ Minutes of Going Nowhere

Here's the honest accounting of what I tried before getting to the actual solution:

Step| What I Tried| Time Spent| Result| Why It Failed  
---|---|---|---|---  
1| SQL injection via text input| ~10 min| No response change| No database backend; form is static  
2| XSS payloads in input fields| ~10 min| No reflection| Input not rendered back to DOM  
3| Brute-forcing hidden paths (/admin, /flag, /secret)| ~8 min| 404 on everything| This is a static challenge page, no hidden routes  
4| Checking cookies and local storage for encoded values| ~5 min| Empty / session cookie only| Flag isn't stored client-side in that way  
5| HTML source view — finally| 2 min| Found CSS and JS references| Should have been step 1  
  
Thirty-three minutes. That's how long it took me to do what should have been the first thing. If you're laughing right now, that's fair. I was laughing too — eventually.

## The Actual Solution: Reading the Source

### Step 1: View HTML Source

I hit `Ctrl+U` to open the raw HTML source. Right away I could see two external file references: `style.css` and `script.js`. Nothing unusual about that structure — every web page has CSS and JS. But in a CTF context, "included files" is practically a signpost.
    
    
    <link rel="stylesheet" href="style.css">
    <script src="script.js"></script>

The challenge name is literally "Includes." I cannot stress enough how much I should have caught that earlier.

### Step 2: Open style.css

I navigated directly to `/style.css` by appending it to the base URL. The file loaded. I scrolled through the usual CSS declarations — nothing unusual — until I hit a comment at the bottom of the file:
    
    
    /* You need to include picoCTF{1nclu51v17y_1of2_ */

There it was. Half a flag, wrapped in a developer comment. My first reaction was a small, embarrassed laugh — this was _right here_ the whole time, in the CSS file, which I hadn't thought to open for over half an hour.

### Step 3: Open script.js

With the first fragment in hand, I opened `/script.js`. Same approach — append to base URL, scroll through the file. Near the bottom:
    
    
    // f7w_2of2_6edef411}

The second fragment. A JavaScript single-line comment hiding in plain sight.

### Step 4: Combine the Fragments

Concatenate the two parts:
    
    
    picoCTF{1nclu51v17y_1of2_f7w_2of2_6edef411}

That's the flag. The moment I pasted both strings together I had that very specific CTF feeling — not triumph exactly, more like relief mixed with "oh come on, really?" It's humbling when a 100-point Easy challenge makes you feel genuinely foolish for 30 minutes.

## Why I Tried SQL Injection and XSS First (And Why That Was a Mistake)

I want to be specific about the reasoning failure here, because it's a pattern worth breaking.

SQL injection made sense in my head because I saw an input field and assumed there was a database behind it. That's an assumption without evidence. I should have checked whether the form even submitted a request before crafting payloads for a backend that didn't exist.

XSS followed from the same mistake: I was looking for a reflection point in a page that wasn't reflecting user input anywhere. I was testing a hypothesis I'd constructed without reading the actual page behavior first.

The correct mental model for an Easy web challenge should be: _start with what the browser already has access to, then escalate_. HTML source → linked resources → cookies/storage → requests → server behavior. I started in the middle and paid for it with half an hour of my time.

## Why This Matters in the Real World: Comment Leakage

This challenge isn't contrived. Leaving sensitive information in code comments is a genuine, documented vulnerability class. A few real cases worth knowing:

  * **API keys in JavaScript files:** Developers frequently embed API keys in client-side JS for quick prototyping, then ship to production without stripping them. Google Maps API keys, Firebase credentials, and Stripe publishable keys have all been leaked this way. Some of these allow read access to entire databases.
  * **AWS credentials in HTML comments:** AWS Access Key IDs and Secret Access Keys have been found in HTML comments of public-facing pages. Once extracted, these credentials can be used to access S3 buckets, spin up EC2 instances, or exfiltrate data.
  * **Internal path disclosure:** Comments like `/* TODO: move /admin/config.php before launch */` reveal server directory structures that help attackers map the application.
  * **Debug notes left in production:** Comments such as `// password is "admin123" for now` or `/* hardcoded until OAuth is ready */` have appeared in real production codebases during security audits.



The picoCTF "Includes" challenge distills this exact pattern: split sensitive data across multiple files, leave it in comments, and wait to see who checks. In a real engagement, an attacker running a passive reconnaissance pass over your JS and CSS files would find this in seconds. Automated tools like [LinkFinder](<https://github.com/GerbenJavado/LinkFinder>) and [subjs](<https://github.com/lc/subjs>) scrape JS files specifically looking for exposed secrets.

## OWASP Top 10: Security Misconfiguration

This vulnerability maps directly to **OWASP Top 10 A05:2021 — Security Misconfiguration**. The OWASP category covers scenarios where systems are deployed with unnecessary features enabled, default credentials unchanged, or — as here — error-prone development artifacts left in production code.

Leaving debug comments, internal notes, or fragment data in unminified CSS/JS files qualifies as misconfiguration because it exposes internal implementation details that serve no function for end users but provide genuine value to an attacker. The fix is straightforward: minify and strip comments from all production assets. A build pipeline using tools like Webpack, Rollup, or esbuild handles this automatically — there's no excuse in 2026 for shipping commented development notes to a production server.

## Beginner Web Challenge Checklist

Based on this challenge (and the 33 minutes I wasted), here's the checklist I now follow before trying anything fancy:

  * View HTML source (`Ctrl+U` or right-click → View Page Source)
  * Check all externally linked files: CSS, JS, images, fonts
  * Read every comment in every file — CSS `/* */`, JS `//` and `/* */`, HTML `<!-- -->`
  * Open browser DevTools → Network tab, reload the page, inspect every request
  * Check cookies, localStorage, and sessionStorage for encoded or suspicious values
  * Look at the page title, meta tags, and hidden form fields
  * Check robots.txt and sitemap.xml for path hints
  * Only after all of the above: start testing for injection, XSS, or authentication bypass



If you follow this order, you'll solve "Includes"-style challenges in under two minutes every time. I know this now. You know this now. Let's not lose 33 minutes again.

## Retrospective: How I'd Solve This Next Time

If I encountered a challenge like this again, my first move would be to look at the challenge name. "Includes" is not a subtle hint — it's telling you exactly what the mechanic is. Always read the problem title as a potential clue about what technique is being tested.

Second, I'd open the HTML source before even interacting with the page. No clicking buttons, no typing in forms — just `Ctrl+U`, read the markup, note every external resource, and visit each one immediately.

Third, I'd use a tool like Burp Suite to passively collect all resources the page loads, so I don't miss anything that isn't in an obvious `<link>` or `<script>` tag. Dynamic imports, fetch calls, and lazy-loaded resources can hide interesting content that raw source viewing won't catch.

The core lesson: don't let your instinct for "cool" techniques — injection, XSS, SSRF — blind you to what's already sitting in the DOM.

## Further Reading

This problem is part of the picoCTF series. You can see the other problems [here](<https://alsavaudomila.com/category/picoctf-web-exploitation-easy/>).

For more Forensics Tools, check out [CTF Forensics Tools: The Ultimate Guide for Beginners](<https://alsavaudomila.com/ctf-forensics-tools/>).

Here are related articles from [alsavaudomila.com](<https://alsavaudomila.com/>) that complement this challenge:

  * [RED picoCTF Writeup](<https://alsavaudomila.com/red-picoctf-writeup/>) — another picoCTF challenge involving hidden data
  * [Ph4nt0m 1ntrud3r picoCTF Writeup](<https://alsavaudomila.com/ph4nt0m-1ntrud3r-picoctf-writeup/>) — network forensics challenge
