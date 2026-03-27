---
title: "CTFで使うsteghide — JPEGやBMPにデータが埋め込まれているか確認する"
emoji: "🔏"
type: "tech"
topics: ["ctf", "forensics", "steganography", "linux", "security"]
published: true
---

steghideはJPEGやBMP画像にデータを埋め込む・取り出すツールです。CTFではパスフレーズ付きで埋め込まれたデータを取り出す問題によく登場します。

## steghideを使うタイミング

JPEG/BMP画像が渡されて、他の解析で何も見つからないとき。特に問題文や別ファイルにパスフレーズのヒントがある場合はsteghideが使われていることが多いです。

PNGには対応していないため、PNG画像にはzstegを使います。この使い分けは覚えておく価値があります。

## パスフレーズなしでも試せる

パスフレーズが分からなくても `steghide extract -sf flag.jpg -p ""` で空のパスフレーズを試すことができます。問題によってはパスフレーズなしで埋め込んであるケースもあります。

## セキュリティ的な意味

steghideはマルウェアがC2通信を隠すために使った事例があります。一見普通の画像ファイルに命令を埋め込んで、URLフィルタリングをすり抜ける手法です。インシデント対応でJPEGファイルがネットワークで大量に送受信されていた場合、steghideで中身を確認することがあります。

## 詳細記事

インストール方法・パスフレーズ付き抽出・PNG/BMP/JPEGの使い分けを含む英語記事はこちら：

→ [steghide in CTF](https://alsavaudomila.com/steghide-in-ctf-how-to-hide-and-extract-data-from-files/)
