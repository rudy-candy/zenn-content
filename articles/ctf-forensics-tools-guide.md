---
title: "CTFのForensicsで使うツール選びの判断基準【picoCTF対応】"
emoji: "🔍"
type: "tech"
topics: ["ctf", "forensics", "picoctf", "security", "linux"]
published: true
---

CTFのForensicsカテゴリで一番つまずくのは「どのツールを使えばいいか」という判断です。ツールはインストールしてある。でも、ファイルを渡された瞬間に何を開けばいいか分からない。

この記事では、ファイルの種類ごとに「最初に使うべきツール」と「その判断理由」をまとめます。

## まずやること：ファイルの正体を確認する

拡張子は信用しないのが基本です。

```bash
file challenge.bin
xxd challenge.bin | head -20
```

`file` はマジックバイトを読んで本当のフォーマットを教えてくれます。`xxd` で先頭のバイト列を見れば、ヘッダが壊れているかどうかも分かります。

`file` が "data" と返してきたら要注意です。意図的にヘッダを壊してあるケースが多い。

## ファイル種別ごとのツール選択

### ディスクイメージ（.img / .dd / raw）

```bash
# 1. パーティション構造を確認
fdisk -l disk.img

# 2. 必要なパーティションを切り出す
dd if=disk.img of=part.img bs=512 skip=<オフセット> count=<セクタ数>

# 3. マウントして中身を確認
mount -o loop part.img /mnt/ctf
```

**順番が重要です。** `mount` から始めると失敗することが多い。`fdisk` でオフセットを確認してから `mount -o offset=` を使うのが正解。

詳しい使い方はこちら（英語）：
- [fdisk in CTF](https://alsavaudomila.com/fdisk-in-ctf-partition-analysis-and-common-challenge-patterns/)
- [dd in CTF](https://alsavaudomila.com/dd-in-ctf-disk-imaging-extraction-and-common-challenge-patterns/)
- [mount in CTF](https://alsavaudomila.com/mount-in-ctf-disk-image-mounting-and-common-challenge-patterns/)

### 音声ファイル（.wav / .mp3 / .flac）

**最初にやること：Audacityでスペクトログラム表示。**

周波数ドメインに文字や画像が隠されているパターンが多い。波形表示のまま見ていても何も分からないので、表示を切り替えるのが先。

```bash
# CLIで処理するならffmpegかsox
ffmpeg -i challenge.mp3 output.wav
sox challenge.wav -n stat
```

詳しい使い方：
- [Audacity in CTF](https://alsavaudomila.com/audacity-in-ctf-audio-forensics-techniques-and-common-challenge-patterns/)
- [FFmpeg in CTF](https://alsavaudomila.com/ffmpeg-in-ctf-how-to-analyze-and-manipulate-audio-video-files/)
- [SoX in CTF](https://alsavaudomila.com/sox-in-ctf-how-to-analyze-and-manipulate-audio-files/)

### 画像ファイル（.png / .jpg）

```bash
# PNGならまずチャンク検証
pngcheck -v flag.png

# JPEGならsteghideでデータ埋め込みを確認
steghide extract -sf flag.jpg
```

PNGチャンクが壊れていると他のツールが正常動作しないので、`pngcheck` を先に走らせる癖をつけています。

詳しい使い方：
- [pngcheck in CTF](https://alsavaudomila.com/pngcheck-in-ctf-how-to-analyze-and-repair-png-files/)
- [steghide in CTF](https://alsavaudomila.com/steghide-in-ctf-how-to-hide-and-extract-data-from-files/)

### QRコード・バーコード

```bash
zbarimg flag.png
```

これで "0 barcodes detected" と返ってきたら、色反転か解像度不足がほぼ原因。

```bash
# 色反転して再スキャン
convert flag.png -negate inverted.png && zbarimg inverted.png

# 解像度が低い場合は拡大
convert flag.png -resize 400% -filter point upscaled.png && zbarimg upscaled.png
```

詳しい使い方：[zbarimg in CTF](https://alsavaudomila.com/zbarimg-in-ctf-qr-barcode-decoding-techniques-and-common-challenge-patterns/)

### 暗号化ZIP

```bash
# まず暗号化方式を確認（ZipCryptoかAES-256か）
7z l -slt challenge.zip | grep Method

# ZipCryptoならクラック可能
zip2john challenge.zip > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**AES-256だとjohnでは解けません。** パスワードのヒントが問題文にあるはず。

詳しい使い方：[zip2john in CTF](https://alsavaudomila.com/zip2john-in-ctf-extracting-zip-passwords-and-common-challenge-patterns/)

## 迷ったときの初動フロー

```
ファイルを受け取る
    ↓
file / xxd でフォーマット確認
    ↓
strings | grep -i flag （意外と効く）
    ↓
binwalk で埋め込みファイルを確認
    ↓
フォーマット別のツールへ
```

`strings` は地味に見えますが、フラグが平文で埋め込まれているケースが実際にあります。スキップしないようにしています。

## まとめ

各ツールの詳細な使い方・CTFでのパターン・ハマりやすいポイントは英語記事にまとめています。

→ [CTF Forensics Tools: The Ultimate Guide for Beginners](https://alsavaudomila.com/ctf-forensics-tools/)

picoCTFの具体的な問題の解き方はこちら：

→ [alsavaudomila.com](https://alsavaudomila.com/)

