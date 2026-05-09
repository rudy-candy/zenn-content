---
title: "CTFで使うzbarimg — QRコードが読めないときの診断フロー"
emoji: "📷"
type: "tech"
topics: ["ctf", "forensics", "linux", "qrcode", "security"]
published: true
---

zbarimgはターミナルからQRコード・バーコードを画像ファイルから読み取るコマンドです。CTFのForensicsやGeneral Skillsで頻出のツールですが、「0 barcodes detected」と返ってくることも多く、そこからの対処が重要です。

## pyzbarとの違いを知っておく

`pyzbar`（Pythonライブラリ）と `zbarimg`（CLIツール）は同じZBarライブラリのラッパーです。ただし：

- `pip install pyzbar` だけでは動かない——`libzbar0`（CライブラリのNative依存）を別途インストールする必要がある
- `zbarimg` は `zbar-tools` パッケージに含まれていて、インストール後すぐ使える

CTFの競技中にハマりやすい罠なので、先に `sudo apt install zbar-tools` して `zbarimg` を使えるか確認しておくことをすすめます。

## 「0 barcodes detected」の4つの原因

別のコンペで反転色QRコードに40分溶かした経験から、失敗パターンを整理しました。

**1. 色が反転している**（一番多い）
QRコードは「黒モジュール×白背景」が標準（ISO 18004）。白黒が逆だと完全に無視されます。`convert -negate` で反転してから再スキャンします。

**2. 解像度が低すぎる**
100×100ピクセル以下だとモジュールの輪郭がつぶれて認識不能になります。`convert -resize 400% -filter point` で拡大します（`-filter point` でぼかさずシャープに拡大）。

**3. QRコードが大きな画像の一部に含まれている**
トリミングしてから再スキャン。

**4. 動画のフレームに含まれている**
ffmpegでフレームを抽出してからzbarimgを使います。

## zbarimgが「ゼロでも黙っている」ことが問題

zbarimgはどの失敗ケースでも同じメッセージしか返しません：
`scanned 0 barcode symbols from 1 images in 0 seconds`

エラーメッセージなし、ヒントなし。だから診断の順序を決めておくことが重要です。まず目視で画像を確認する（色反転？）、次に解像度確認、という手順を踏むと迷わずに済みます。

## セキュリティ的な意味

QRコードを使ったフィッシング（クイッシング）の調査でも、怪しい画像をCLIで一括スキャンする場面があります。`zbarimg *.png` でバッチ処理できるため、大量ファイルの自動処理に向いています。

## 詳細記事

各失敗パターンの対処コマンド・診断フロー表・pyzbar/OpenCVとの比較表を含む英語記事はこちら：

→ [zbarimg in CTF: QR/Barcode Decoding Guide](https://alsavaudomila.com/zbarimg-in-ctf-qr-barcode-decoding-techniques-and-common-challenge-patterns/)
