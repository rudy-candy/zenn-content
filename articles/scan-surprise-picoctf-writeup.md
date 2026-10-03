---
title: "【picoCTF】Scan Surprise — CLIでQRコードを読む方法を知っているか"
emoji: "📷"
type: "tech"
topics: ["ctf", "picoctf", "forensics", "linux", "security"]
published: true
---

:::message
2026年10月追記：picoCTF は CyLab Security Academy（learn.cylabacademy.org）に名前が変わり、フラグの形式も `picoCTF{...}` から `academy{...}` に変わりました。この記事のフラグとコマンドは当時のものです。今の配布ファイルで解くときは、`picoCTF{` を `academy{` に読み替えてください（当時のフラグは今は通りません）。
:::

picoCTFのGeneral Skillsカテゴリにある「Scan Surprise」は、解法自体は単純です。でもこの問題が問いかけているのは「ツールの存在を知っているか」という一点だけです。

## Pythonで解こうとして25分溶かした話

QRコードのPNGを渡されたとき、最初に思いついたのは `pyzbar` でPythonスクリプトを書くことでした。インストールも一瞬で終わり、コードも数行。……のはずでした。

実際には `ImportError: Unable to find zbar shared library` というエラーに10分近く詰まりました。`pyzbar` はPythonラッパーで、ネイティブのCライブラリ（`libzbar0`）を別途インストールしなければ動きません。それを知らなかった。

最終的にDiscordで「なんでzbarimgを使わないの？」と言われて初めて存在を知りました。

```bash
zbarimg flag.png
# QR-Code:picoCTF{p33k_@_b00_3f7cf1ae}
```

ワンコマンド。1秒未満。

## この問題が教えてくれること

**コードを書く前に、CLIツールを探す**という習慣です。

QRコードなら `zbarimg`、音声なら `sox`、PDFなら `pdfdumper`、バイナリなら `binwalk`。フォーマットに対応するCLIツールを知っていれば、Pythonで車輪を再発明する時間は不要になります。

CTFのGeneral Skillsカテゴリはこういう問題が多い。「ツールを知っているか」だけを問うている。

## QRコードが読めないときの対処

`zbarimg` は失敗してもエラーを出しません。ただ「0 barcodes detected」と返します。その場合のチェック順：

1. 色が反転していないか（白背景に黒QRが正常）
2. 解像度が低すぎないか（100px以下は失敗しやすい）
3. 大きな画像の一部にQRが含まれているか

詳細な対処コマンドは英語記事にまとめています。

## セキュリティ的な意味

QRコードを使ったフィッシング（「クイッシング」）は、メール本文のURLフィルタを迂回するために使われます。CLIで解析できると、怪しいQRコード画像を自動処理したりバッチスキャンしたりする場面で役立ちます。

## 詳細記事

手順とコマンド出力を全部書いた日本語の完全版は、noteに置いています（有料）。

→ [picoCTF「Scan Surprise」writeup 日本語解説（note・¥200）](https://note.com/rudy_candy/n/n30bfc712d72d)
→ [zbarimg in CTF（ツール詳細・4つの失敗モード）](https://alsavaudomila.com/zbarimg-in-ctf-qr-barcode-decoding-techniques-and-common-challenge-patterns/)
