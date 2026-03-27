---
title: "CTFで使うzbarimg — QRコードが読めないときの診断フロー"
emoji: "📷"
type: "tech"
topics: ["ctf", "forensics", "linux", "qrcode", "security"]
published: true
---

zbarimgはターミナルからQRコード・バーコードを画像ファイルから読み取るコマンドです。CTFのForensicsやGeneral Skillsで頻出のツールですが、「0 barcodes detected」と返ってくることも多く、そこからの対処が重要です。

## 「0 barcodes detected」の原因と対処

zbarimgが失敗する主な原因は4つあります：

**1. 色が反転している**
QRコードは「黒モジュールが白背景」が標準です。反転していると読めません。`convert -negate` で反転してから再スキャンします。

**2. 解像度が低すぎる**
50×50ピクセル以下だとデコーダが認識できないことがあります。`convert -resize 400% -filter point` で拡大します（`-filter point` で輪郭をぼかさないことが重要）。

**3. QRコードが大きな画像の一部に含まれている**
トリミングしてから再スキャンします。

**4. 動画のフレームに含まれている**
ffmpegでフレームを抽出してからzbarimgを使います。

## セキュリティ的な意味

QRコードを使ったフィッシング（クイッシング）の調査でも、怪しいQRコード画像をCLIで一括スキャンする場面がでてきます。zbarimgはbatchprocessingに対応しているため、`zbarimg *.png` で複数ファイルを一度に処理できます。

## 詳細記事

各失敗パターンの対処コマンド・診断フローチャートを含む英語記事はこちら：

→ [zbarimg in CTF](https://alsavaudomila.com/zbarimg-in-ctf-qr-barcode-decoding-techniques-and-common-challenge-patterns/)
