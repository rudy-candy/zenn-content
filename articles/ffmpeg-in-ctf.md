---
title: "CTFで使うffmpeg — 音声・動画ファイルを開けないときの最初の一手"
emoji: "🎬"
type: "tech"
topics: ["ctf", "forensics", "ffmpeg", "linux", "security"]
published: true
---

CTFで音声や動画ファイルが渡されたとき、Audacityで開けない・codecエラーが出るという状況によく遭遇します。そのときffmpegが最初の解決策になります。

## ffmpegがCTFで必要になる場面

Audacityやメディアプレイヤーは対応コーデックが限られています。ffmpegはほぼあらゆる形式を変換できるため「とりあえずffmpegで変換する」というのがよくあるフローです。

また、動画ファイルからQRコードを含むフレームを抽出したり、音声トラックだけを取り出したりする作業もffmpegでないとできません。

## なぜ他のツールではなくffmpegか

GUIツールは対話的な操作に向いていますが、大量のファイルを処理したり、スクリプトで自動化したりするにはCLIのffmpegが圧倒的に使いやすいです。変換・分割・抽出・フレーム取得がワンコマンドで完結します。

## セキュリティ的な意味

フォレンジック調査で押収した機器に動画ファイルが含まれていることは珍しくありません。証拠として動画から特定フレームを抽出したり、メタデータを確認したりする場面でffmpegは使われます。

## 詳細記事

CTFで頻出のffmpegコマンド・実際の出力・AudioとVideoを組み合わせた解析フローを含む英語記事はこちら：

→ [FFmpeg in CTF](https://alsavaudomila.com/ffmpeg-in-ctf-how-to-analyze-and-manipulate-audio-video-files/)
