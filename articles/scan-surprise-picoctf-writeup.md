---
title: "【picoCTF】Scan Surprise — CLIでQRコードを読む方法を知っているか"
emoji: "📷"
type: "tech"
topics: ["ctf", "picoctf", "forensics", "linux", "security"]
published: true
---

picoCTFのGeneral Skillsカテゴリにある「Scan Surprise」は、解法自体は単純です。でもこの問題が問いかけているのは「ツールの存在を知っているか」という一点だけです。

## 問題の概要

ZIPファイルを解凍すると `flag.png` が出てきます。開くとQRコードが表示されます。

最初の反応はたいてい「スマホで読めばいいか」です。実際それでも読めます。でもCTFのターミナル環境で作業しているとき、スマホを取り出すのは非効率だし、後から再現できるログも残りません。

## この問題が教えてくれること

**`zbarimg` というCLIツールが存在する**、ということです。

LinuxのターミナルからワンコマンドでQRコード・バーコードを画像から読み取れます。このツールを知っているかどうかで、この問題の所要時間が2分か20分かに分かれます。

## セキュリティ的な意味

QRコードはフィッシング攻撃でよく使われます（「クイッシング」と呼ばれる手法）。メール本文のURLはフィルタリングされても、QRコード画像は素通りするケースがある。

CLIでQRコードを解析できると、怪しいQRコード画像を自動処理したり、大量の画像をバッチスキャンしたりする場面で役立ちます。

## 詳細記事

解法の詳細・実際のコマンド出力・「zbarimgが0を返したときの対処法」まで含めた英語記事はこちら：

→ [Scan Surprise picoCTF Writeup](https://alsavaudomila.com/scan-surprise-picoctf-writeup/)
→ [zbarimg in CTF（ツール詳細）](https://alsavaudomila.com/zbarimg-in-ctf-qr-barcode-decoding-techniques-and-common-challenge-patterns/)
