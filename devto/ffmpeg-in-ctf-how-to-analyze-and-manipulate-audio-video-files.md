---
title: "FFmpeg for CTF Forensics"
published: true
description: "
The CTF Video That Hid the Flag in a Single Frame I Almost Missed
I once spent the better part of an hour on a CTF video forensics challenge doing everyth"
tags: ctf, security, linux, forensics
canonical_url: https://alsavaudomila.com/ffmpeg-in-ctf-how-to-analyze-and-manipulate-audio-video-files/
---

## The CTF Video That Hid the Flag in a Single Frame I Almost Missed

I once spent the better part of an hour on a CTF video forensics challenge doing everything right — inspecting metadata, analyzing the extracted audio, checking every frame visually — and still couldn't find the flag. What I didn't realize was that the flag appeared in exactly one frame out of several thousand, displayed for less than a single frame's duration at the native playback speed. VLC showed nothing. Stepping through manually was hopeless. It was only after I dumped every frame with `ffmpeg` and then wrote a twelve-line Python script to scan the output folder that I found it: frame 2847, a white rectangle with a QR code that existed nowhere else in the video.

That challenge taught me two things: first, that video forensics challenges routinely hide information at timescales and in layers that human perception simply can't catch; and second, that `ffmpeg` is the only tool that can systematically decompose a media file down to every individual frame, stream, and metadata field. This article is about building the same instincts — when to reach for ffmpeg, what to extract first, and the patterns that catch most beginners off guard.

## The FFmpeg Commands That Cover 90% of CTF Work

FFmpeg has hundreds of options. In CTF forensics, you use maybe eight commands repeatedly. Here's what each one reveals and when to reach for it:

### Inspect Everything First
    
    
    # Run this before anything else — shows codec, resolution, duration, metadata, errors
    $ ffmpeg -i challenge.mp4
    
    # More detailed output including all metadata fields
    $ ffprobe -v quiet -show_format -show_streams challenge.mp4

The `ffmpeg -i` output tells you how many streams the file contains, what codecs are used, whether there's embedded metadata, and — crucially — whether ffmpeg detects any errors or corruption. I've had challenges where the error message itself contained a hint, and challenges where the metadata showed a flag directly. Always read the full output before doing anything else.

### The Four Core Extraction Commands
    
    
    # Extract ALL frames (most important command in video forensics)
    $ mkdir frames && ffmpeg -i challenge.mp4 frames/frame_%05d.png
    
    # Extract audio track as WAV (for Audacity/spectrogram analysis)
    $ ffmpeg -i challenge.mp4 audio.wav
    
    # Extract a single frame at a specific timestamp
    $ ffmpeg -ss 00:01:23 -i challenge.mp4 -vframes 1 frame_at_83s.png
    
    # Extract metadata to a file for searching
    $ ffprobe -v quiet -show_format -show_streams challenge.mp4 > metadata.txt
    $ grep -i "flag\|ctf\|pico\|comment\|title" metadata.txt

### Speed and Repair Commands
    
    
    # Slow down video 2x (useful for catching fast-flashing frames visually)
    $ ffmpeg -i challenge.mp4 -filter:v "setpts=2.0*PTS" slow_video.mp4
    
    # Slow down audio 2x (for sped-up speech or tones)
    $ ffmpeg -i audio.wav -filter:a "atempo=0.5" slow_audio.wav
    
    # Attempt to repair a corrupted/broken media file
    $ ffmpeg -i broken.mp4 -c copy fixed.mp4
    # If that fails, try re-encoding:
    $ ffmpeg -i broken.mp4 fixed.mp4
    
    # Extract raw pixel data in grayscale (for pixel-level steganography)
    $ ffmpeg -i challenge.mp4 -pix_fmt gray gray_output.mp4

## Rabbit Hole: The Hour I Spent Watching a Video Instead of Decomposing It

Here's what my actual workflow looked like on that single-frame challenge before I understood the right approach:
    
    
    $ file challenge.mp4
    challenge.mp4: ISO Media, MP4 Base Media v1 [ISO 14496-12:2003]
    
    $ strings challenge.mp4 | grep -i "flag\|ctf\|pico"
    (no output)
    # Flag was visual content in a frame, not ASCII bytes in the container
    
    # Opened in VLC, watched the 3-minute video
    # Saw: a person talking, some background music, a brief flicker around the 1-minute mark
    # Didn't pause at the flicker — it was too fast to catch manually
    
    $ ffmpeg -i challenge.mp4
    # Duration: 00:03:12, Stream #0: Video h264, 1920x1080 @ 30fps
    # Stream #1: Audio aac, stereo 44100Hz
    # No errors, no obvious metadata
    # I stopped reading here — missed the custom "comment" metadata field
    
    # Extracted audio and opened in Audacity
    # Checked spectrogram — nothing obvious
    # Tried reversing, slowing down — nothing
    
    # Tried ffprobe — copy-pasted output but didn't grep it
    # Scrolled past "TAG:comment=Look at frame 2847" without noticing
    
    # --- 50 minutes in ---
    # Extracted all frames (should have done this at minute 3)
    $ mkdir frames && ffmpeg -i challenge.mp4 frames/frame_%05d.png
    # Generated 5,760 PNG files
    
    # Wrote a quick script to find anomalous frames
    $ python3 -c "
    import os
    from PIL import Image
    sizes = [(os.path.getsize(f'frames/{f}'), f) for f in os.listdir('frames')]
    sizes.sort(reverse=True)
    print(sizes[:5])
    "
    # frame_02847.png was 8x larger than average — contained the QR code
    
    $ zbarimg frames/frame_02847.png
    QR-Code:picoCTF{...}

Two mistakes compounded here. First: I watched the video instead of immediately extracting all frames — three minutes of watching versus thirty seconds of frame extraction. Second: I ran `ffprobe` but didn't grep the output, and the challenge author had literally put "Look at frame 2847" in a metadata comment field. The answer was in the first tool I ran. I just didn't read the output carefully enough.

## Seven CTF Patterns Where FFmpeg Is the Right Tool

Video and audio forensics challenges cluster into recognizable patterns. Here's what each looks like and the specific ffmpeg approach that works:

### Pattern 1: Flag Hidden in a Single Frame

The flag appears in one frame — a QR code, text overlay, or image — often for a fraction of a second. Impossible to catch by watching; trivial to find once all frames are extracted.
    
    
    $ mkdir frames && ffmpeg -i challenge.mp4 frames/frame_%05d.png
    
    # Find the anomalous frame by file size (larger = more content)
    $ ls -la frames/ | sort -k5 -rn | head -10
    
    # Or use Python to find outliers:
    $ python3 -c "
    import os
    from pathlib import Path
    files = list(Path('frames').glob('*.png'))
    sizes = [(f.stat().st_size, f.name) for f in files]
    avg = sum(s for s,_ in sizes) / len(sizes)
    outliers = [(s,n) for s,n in sizes if s > avg * 3]
    print(sorted(outliers, reverse=True)[:10])
    "
    
    # Once you find the frame, decode it:
    $ zbarimg frames/frame_02847.png        # for QR/barcodes
    $ tesseract frames/frame_02847.png out  # for text (OCR)

The file-size outlier trick works because a frame containing a QR code or text is significantly larger than frames containing blurred video content. It's not perfect — highly detailed video frames can also be large — but it narrows 5000 frames down to ten candidates in seconds.

### Pattern 2: Flag in Metadata

The flag is stored directly in a metadata field — title, comment, artist, encoder, or a custom tag. This takes fifteen seconds to check and is often overlooked because people open the video in a player rather than inspecting it with ffprobe.
    
    
    $ ffprobe -v quiet -show_format -show_streams challenge.mp4 | grep -i "TAG\|title\|comment\|artist\|encoder"
    TAG:title           : nothing_here
    TAG:comment         : picoCTF{m3tadata_1s_0ften_0verl00ked}
    TAG:encoder         : Lavf58.76.100
    
    # Also check for unusual stream counts or codec names
    $ ffprobe -v quiet -show_streams challenge.mp4 | grep codec_name
    codec_name=h264
    codec_name=aac
    codec_name=mjpeg    # ← unexpected third stream — worth extracting

That last example — an unexpected `mjpeg` stream in an MP4 that should only have video and audio — is a real pattern I've seen. The third stream was a still image of the flag, attached to the video file as an "album art" stream. `ffmpeg -i` showed three streams; most people only extract the first two.

### Pattern 3: Audio Analysis (Spectrogram / Morse / DTMF)

The video's audio track contains the flag — hidden in the spectrogram, encoded as Morse code, or encoded as DTMF tones. FFmpeg's job here is extraction; analysis happens in Audacity or other tools.
    
    
    # Extract audio as WAV for Audacity analysis
    $ ffmpeg -i challenge.mp4 audio.wav
    
    # If the audio is in an unusual format, normalize it first
    $ ffmpeg -i challenge.mp4 -ar 44100 -ac 1 audio_mono.wav
    
    # For DTMF decoding without Audacity:
    $ multimon-ng -t wav -a DTMF audio.wav
    
    # If the audio track sounds wrong — try slowing it down
    $ ffmpeg -i audio.wav -filter:a "atempo=0.5" audio_half_speed.wav

### Pattern 4: Speed-Manipulated Video or Audio

Content recorded at the wrong speed — speech that sounds like chipmunks (too fast), visually incomprehensible motion (too fast or too slow), or audio tones at the wrong pitch. The ffmpeg filter approach fixes these cleanly.
    
    
    # Video too fast — slow it to half speed
    $ ffmpeg -i challenge.mp4 -filter:v "setpts=2.0*PTS" -filter:a "atempo=0.5" slowed.mp4
    
    # Audio too fast (chipmunk voice) — slow audio only, preserve video
    $ ffmpeg -i challenge.mp4 -filter:a "atempo=0.5" -c:v copy audio_fixed.mp4
    
    # For extreme speed changes (< 0.5x or > 2.0x), chain atempo filters:
    $ ffmpeg -i challenge.mp4 -filter:a "atempo=0.5,atempo=0.5" quarter_speed.wav
    # atempo only accepts 0.5–2.0 per filter; chain for larger changes
    
    # Pitch-shifted audio (content shifted up or down by semitones):
    $ ffmpeg -i audio.wav -filter:a "asetrate=44100*0.5,aresample=44100" pitch_down.wav

### Pattern 5: Corrupted or Broken Media File

The challenge file won't open in any player — wrong container, corrupted header, missing moov atom. FFmpeg often fixes these automatically when you re-encode or remux the file.
    
    
    # Try remuxing first (fast, no re-encoding)
    $ ffmpeg -i broken.mp4 -c copy fixed.mp4
    
    # If that fails, try full re-encode
    $ ffmpeg -i broken.mp4 fixed.mp4
    
    # If ffmpeg can't identify the format, try forcing one:
    $ ffmpeg -f mp4 -i broken.bin fixed.mp4
    $ ffmpeg -f avi -i broken.bin fixed.avi
    
    # Check the actual file header to identify the real format:
    $ xxd broken.mp4 | head -3
    # ftyp = MP4, RIFF = AVI, 1a45dfa3 = MKV/WebM
    # If the header doesn't match the extension, rename and re-try

### Pattern 6: Multiple Hidden Streams

The media file contains more streams than expected — a second audio track, a subtitle stream, an attached image, or a data stream. Each extra stream is a potential hiding spot.
    
    
    $ ffprobe -v quiet -show_streams challenge.mkv | grep -E "codec_name|index|codec_type"
    index=0
    codec_type=video
    codec_name=h264
    index=1
    codec_type=audio
    codec_name=aac
    index=2
    codec_type=subtitle   # ← subtitle stream — extract and read it
    codec_name=subrip
    index=3
    codec_type=data       # ← data stream — extract as binary
    codec_name=bin_data
    
    # Extract specific streams by index
    $ ffmpeg -i challenge.mkv -map 0:2 subtitles.srt
    $ ffmpeg -i challenge.mkv -map 0:3 -c copy hidden_data.bin
    
    # Then: strings hidden_data.bin | grep -i flag
    # Or: file hidden_data.bin (may reveal a zip, image, etc.)

### Pattern 7: Pixel-Level or Channel Steganography

Data hidden in specific color channels (R/G/B/Y) or in the least significant bits of pixel values. FFmpeg can extract individual color planes; deeper LSB analysis requires external tools.
    
    
    # Extract frames in different pixel formats for channel analysis
    $ ffmpeg -i challenge.mp4 -pix_fmt gray frames_gray/frame_%05d.png   # luminance only
    $ ffmpeg -i challenge.mp4 -pix_fmt yuv420p frames_yuv/frame_%05d.yuv # raw YUV
    
    # For a single suspicious frame, extract and analyze with stegsolve:
    $ ffmpeg -ss 00:01:23 -i challenge.mp4 -vframes 1 suspect_frame.png
    $ java -jar stegsolve.jar   # open suspect_frame.png and cycle through bit planes
    
    # LSB extraction from extracted frames:
    $ zsteg suspect_frame.png   # checks multiple LSB patterns automatically

## FFmpeg vs Other Tools: When to Switch

FFmpeg is a container and stream processor — it doesn't decode steganography, analyze spectrogram content visually, or identify file signatures embedded inside frames. Here's how I decide what to use:

Situation| My First Choice| Why Not FFmpeg?  
---|---|---  
Unknown media file, first look| **ffmpeg -i / ffprobe**|  —  
Extract all frames from video| **ffmpeg**|  —  
Analyze audio spectrogram visually| Audacity| FFmpeg has no GUI spectrogram viewer  
Decode DTMF tones in audio| multimon-ng| FFmpeg extracts audio; decoding is external  
Detect embedded files inside frames| binwalk (on extracted frames)| FFmpeg extracts frames; binwalk finds hidden files in them  
Analyze bit planes of a frame| stegsolve / zsteg| FFmpeg can export to raw formats but doesn't analyze LSB patterns  
Repair a corrupted file container| **ffmpeg -c copy**|  —  
Convert format for another tool| **ffmpeg**|  —  
  
FFmpeg's role in CTF is almost always as a decomposer and converter — it breaks media files apart into frames, streams, and metadata, then hands the pieces to specialized tools. The workflow is rarely "ffmpeg finds the flag." It's "ffmpeg extracts the component that contains the flag, then another tool decodes it."

## Full Trial Process Table

Step| Action| Command| Result| Why it failed / succeeded  
---|---|---|---|---  
1| File identification| file challenge.mp4| ISO Media, MP4 Base Media v1| Confirmed valid MP4 — gave me nothing about content  
2| String search| strings challenge.mp4 | grep flag| No output| Flag was in a video frame, not as raw ASCII in the container  
3| Watch the video| vlc challenge.mp4| 3-minute video with a barely visible flicker at 1:25| Wasted time — the flicker was one frame, impossible to catch by watching  
4| Audio extraction + analysis| ffmpeg -i challenge.mp4 audio.wav → Audacity| No spectrogram content, no Morse pattern| Audio was clean — not the right layer for this challenge  
5| ffprobe metadata check| ffprobe -v quiet -show_format challenge.mp4| Ran it, saw "TAG:comment" in output, didn't read it carefully| The answer was literally in the metadata — I scrolled past it  
6| Frame extraction| ffmpeg -i challenge.mp4 frames/frame_%05d.png| 5,760 PNG files generated| Should have done this at step 1  
7| Anomalous frame detection| Python file-size outlier script| frame_02847.png = 8x average size| QR code frame was significantly larger than average video frames  
8| QR decode| zbarimg frames/frame_02847.png| picoCTF{…}| Done — metadata would have led here in 15 seconds if I'd read it at step 5  
  
## Why Multimedia Decomposition Matters Beyond CTF

FFmpeg exists because multimedia files are containers — a single .mp4 file can hold multiple video tracks, multiple audio tracks, subtitle streams, chapter data, embedded thumbnails, and arbitrary metadata, all encoded in different formats. Most players show you one video and one audio track and ignore the rest. FFmpeg exposes everything.

In real forensic investigations, this matters significantly. Video evidence from security cameras may have metadata timestamps that don't match the actual recording time. A video file submitted as evidence may contain additional streams that weren't visible in the player shown to the court. Audio extracted from a video may contain background frequencies that identify the recording location. The same decomposition techniques from CTF challenges apply directly to evidence analysis.

The CTF lesson that transfers: never trust what a media player shows you. A player makes decisions about what to display. FFmpeg makes no such decisions — it shows you every stream, every metadata field, every frame. In CTF and in real forensics, the important content is often in the parts the player chose not to show.

## My First-Three-Minutes Workflow for Media Forensics
    
    
    # Step 1: Identify and inspect — read ALL of this output
    $ ffmpeg -i challenge.mp4 2>&1 | tee ffmpeg_info.txt
    $ ffprobe -v quiet -show_format -show_streams challenge.mp4 | tee ffprobe_info.txt
    $ grep -i "TAG\|comment\|title\|artist\|flag\|ctf" ffprobe_info.txt
    
    # Step 2: Check stream count — more than 2 streams = suspicious
    $ ffprobe -v quiet -show_streams challenge.mp4 | grep -c "^index="
    
    # Step 3: Extract everything in parallel
    $ mkdir frames
    $ ffmpeg -i challenge.mp4 frames/frame_%05d.png &
    $ ffmpeg -i challenge.mp4 audio.wav &
    wait
    
    # Step 4: Find anomalous frames by size
    $ ls -la frames/ | sort -k5 -rn | head -10
    
    # Step 5: Check audio in Audacity
    # Spectrogram view first — always
    
    # Step 6: If video looks corrupted
    $ ffmpeg -i broken.mp4 -c copy fixed.mp4
    
    # Step 7: If still nothing, check raw bytes of anomalous frames
    $ binwalk frames/frame_02847.png   # may contain embedded files

The change that made the biggest difference: running `ffprobe` and actually reading the full output before doing anything else. It takes thirty seconds and has revealed the answer directly — or at least the right layer to look at — on more than one challenge. Watching the video in a player is almost never the right first step.

## Further Reading

For the full picture of the forensics toolkit that FFmpeg fits into, [CTF Forensics Tools: The Ultimate Guide for Beginners](<https://alsavaudomila.com/ctf-forensics-tools/>) covers where each tool belongs — FFmpeg handles media decomposition, but the analysis layer often requires Audacity, binwalk, zsteg, or Sleuth Kit depending on what you extract.

Here are related articles from [alsavaudomila.com](<https://alsavaudomila.com/>) that pair directly with this topic:

Once FFmpeg extracts the audio track from a video challenge, [Audacity](<https://alsavaudomila.com/audacity-in-ctf-audio-forensics-techniques-and-common-challenge-patterns/>) handles the next layer — spectrogram visualization, Morse code identification, channel splitting, and speed manipulation. The two tools are a natural pair: FFmpeg pulls the audio out, Audacity shows you what's hidden inside it.

When FFmpeg extracts frames and one of them turns out to contain an embedded file (a ZIP inside a PNG, for example), [binwalk](<https://alsavaudomila.com/binwalk-in-ctf/>) is the right next tool — it scans for embedded file signatures and gives you the byte offsets needed for dd extraction. The frame-to-binwalk pipeline is a common multi-step pattern in harder video forensics challenges.

If the video file itself turns out to be a disguised disk image — a pattern that appears occasionally in mixed forensics challenges — the [fdisk guide](<https://alsavaudomila.com/fdisk-in-ctf-partition-analysis-and-common-challenge-patterns/>) covers how to inspect the partition structure and extract the relevant partition for further analysis.
