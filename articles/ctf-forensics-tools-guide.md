---
title: "CTFのForensicsで使うツール選びの判断基準【picoCTF対応】"
emoji: "🔍"
type: "tech"
topics: ["ctf", "forensics", "picoctf", "security", "linux"]
published: true
---

CTFのForensicsカテゴリで一番つまずくのは「どのツールを使えばいいか」という判断です。ツールはインストールしてある。でも、ファイルを渡された瞬間に何を開けばいいか分からない。

20+のpicoCTFフォレンジック問題を解いて気づいたのは、「間違いのパターンが毎回同じ」だということです。この記事はその失敗から作った判断フローです。

## まずやること：ファイルの正体を確認する（拡張子は信用しない）

```bash
file challenge.bin   # マジックバイトから本当の形式を判定
xxd challenge.bin | head -20  # 先頭バイトを直接確認
```

`.png` なのに `file` が `Zip archive` と返してきた経験が複数回あります。拡張子は嘘をつく。

## 次に必ず2つ走らせる：strings と binwalk

```bash
strings challenge.* | grep -i "picoctf\|flag{"
binwalk challenge.*
```

`strings` は地味に見えますが、フラグが平文で埋め込まれているケースが実際にあります。専用ツールを起動する前に必ず確認。

`binwalk` はZIPや圧縮データが埋め込まれていないか確認します。ただし**PNG内でTIFF偽陽性が出ることがある**（ICCプロファイルのmagic bytesがTIFFと一致するため）。全レイヤーで同じオフセットに出る場合は構造的なアーティファクト。

## ファイル種別ごとの最初の一手

### ディスクイメージ（.img / .dd）

```bash
fdisk -l disk.img    # まずパーティション構造を確認
# → オフセットを読んでからdd→mount
```

`mount` から始めるのが典型的な失敗です。「disk disk sleuth II」では2時間マウントを試行して見つからず、fdiskでオフセット確認→dd→mountで35秒で解決しました。

### 音声ファイル（.wav / .mp3）

**最初にAudacityでスペクトログラム表示。** 波形を見ていても何も分からないことが多い。周波数ドメインに文字が隠れているパターンが多いです。

何も見えなければ1–4kHz帯域にズームイン。デフォルトスケールでは見えないことがある。

### 画像ファイル（.png / .jpg）

**PNGはexiftoolを先に**。「RED」という問題では、exiftoolで見つかった`Poem`メタデータフィールドに「CHECKLSB」というアクロスティックが隠されていました。steghideとbinwalkを先に試して5分無駄にしました。

```bash
exiftool challenge.png   # まずメタデータ確認
pngcheck -v challenge.png  # チャンク整合性確認
zsteg challenge.png      # LSBステガノグラフィ解析
```

JPEGはsteghide（PNG非対応）、PNGはzsteg。この区別は重要です。

### ZIPアーカイブ

暗号化方式を先に確認：

```bash
7z l -slt challenge.zip | grep Method
# ZipCrypto → zip2john + hashcat で解読可能
# AES-256  → 辞書攻撃不可・パスワードは問題文のどこかに必ずある
```

AES-256に1時間wordlist攻撃をかけた経験があります。方式確認が先です。

### パケットキャプチャ（.pcap）

Wiresharkの「Follow TCP Stream」はパケットを**到着順**に結合します。意図的に順序を入れ替えて送信されたフラグフラグメントは、到着順で結合すると文字化けします。**タイムスタンプ順**にソートしてから結合するのが正解。

## ツール選択早見表

| ツール | 対象 | 使うタイミング |
|---|---|---|
| file / xxd | すべて | 最初に必ず |
| strings | すべてのバイナリ | 専用ツールの前に |
| binwalk | すべてのバイナリ | 埋め込みファイル検出 |
| exiftool | 画像・動画 | ステガノグラフィ前に |
| fdisk | ディスクイメージ | mountの前に必ず |
| Audacity | 音声 | スペクトログラム表示で最初に |
| pngcheck | PNG | チャンク検証（他ツールの前） |
| zsteg | PNG/BMP | LSBステガノグラフィ |
| steghide | JPEG/BMP | パスフレーズ付き埋め込み（PNGは非対応） |
| zbarimg | QR/バーコード | CLIで最速デコード |
| zip2john | ZIP | ZipCryptoのみ（AES-256は不可） |

## 詳細記事

各ツールの詳細・実際のフラグ値・エラー出力・失敗談は英語記事に：

→ [CTF Forensics Tools: The Ultimate Guide for Beginners](https://alsavaudomila.com/ctf-forensics-tools/)
