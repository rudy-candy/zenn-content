---
title: "【picoCTF】Flag in Flame — ログファイルと名乗った偽物の正体"
emoji: "🔥"
type: "tech"
topics: ["ctf", "picoctf", "forensics", "base64", "security"]
published: false
---

`logs.txt` というファイルを渡されました。名前の通りサーバーログかと思ってgrepし始めて、数分後に気づきました。「これログじゃない」。

## 問題の構造

`file logs.txt` を実行すると：

```
logs.txt: ASCII text, with very long lines (65536), with no line terminators
```

**65536文字、改行なし、1行のみ。** 本物のサーバーログはタイムスタンプごとに改行があります。改行なしで6万文字の1行→エンコードされたデータです。

文字の種類が `A-Z a-z 0-9 + / =` のみ → Base64と判断できます。

## 解読の流れ

Base64でデコードすると……バイナリが出てくる：

```bash
$ base64 -d logs.txt > log.dat
$ file log.dat
log.dat: PNG image data, 896 x 1152, 8-bit/color RGB, non-interlaced
```

PNGでした。ログファイルに偽装したBase64エンコードの画像。

画像を開くと、中に長い16進数の文字列が印字されていました：

```
7069636F4354467B666F72656E736963735F616E616C797369735F69735F616D617A696E675F61633165333538347D
```

`bytes.fromhex()` でデコード → フラグ。

## この問題から学んだこと

**「ファイル名を信用しない」の実践問題**でした。

CTFのセオリー「拡張子ではなくmagic bytesでファイル形式を判断する」は知っていましたが、`.txt` という拡張子と `logs` という名前の組み合わせで「ログファイルだ」という先入観が生まれた。

`file` コマンドの出力を読むのに5秒かかりました。グレップに費やした時間は無駄でした。

## エンコード識別のコツ

文字の分布でエンコードの種類がだいたい分かります：

- `A-Za-z0-9+/=` のみ → Base64
- `0-9a-fA-F` のみ → 16進数（hex）
- アルファベットのみ（数字なし）→ Caesar/ROT13系
- 点と線 → Morse

チェックの順番は `file` → `strings` → エンコード識別、です。

## 詳細記事

実際のコマンド出力・ダブルエンコードの実世界的な意味・エンコード識別の参照表を含む英語記事：

→ [Flag in Flame picoCTF Writeup](https://alsavaudomila.com/flag-in-flame-picoctf-writeup/)
