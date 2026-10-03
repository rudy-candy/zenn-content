---
title: "CTFで使うzip2john — パスワード付きZIPを解析する前にやること"
emoji: "🗜️"
type: "tech"
topics: ["ctf", "forensics", "zip", "linux", "security"]
published: false
---

:::message
2026年10月追記：picoCTF は CyLab Security Academy（learn.cylabacademy.org）に名前が変わり、フラグの形式も `picoCTF{...}` から `academy{...}` に変わりました。この記事のフラグとコマンドは当時のものです。今の配布ファイルで解くときは、`picoCTF{` を `academy{` に読み替えてください（当時のフラグは今は通りません）。
:::

パスワード付きZIPが渡されたとき、すぐにパスワードクラックを試みる前に確認すべきことがあります。暗号化方式によって、クラックできるかどうかが決まります。

## ZipCryptoとAES-256の違い

ZIPの暗号化には主に2種類あります：

- **ZipCrypto**（古い方式）：zip2john + john/hashcatでクラック可能
- **AES-256**（新しい方式）：現実的な時間ではクラックできない

`7z l -slt challenge.zip | grep Method` で暗号化方式を確認してから動くのが正しい手順です。AES-256だと確認せずにjohnを走らせても意味がありません。問題文にパスフレーズのヒントがあるはずです。

## zip2johnのフロー

```
zip2john → ハッシュを取り出す
→ john または hashcat でクラック
→ パスワードで解凍
```

このフローはZIPだけでなく、RARやPDFのパスワードクラックでも同様のアプローチ（それぞれ rar2john、pdf2john）が使えます。

## セキュリティ的な意味

パスワード付きZIPは社内のファイル共有でよく使われますが、ZipCryptoで暗号化されたZIPは既知平文攻撃にも脆弱です（ファイル名や内容が予測できる場合）。zip2johnはそのような脆弱なZIPの安全性評価にも使えます。

## 詳細記事

手順とコマンド出力を全部書いた日本語の完全版は、noteに置いています（有料）。

→ [ZIP暗号を解析するときの思考整理 ― zip2johnを選ぶ基準と捨てる勇気（note・¥500）](https://note.com/rudy_candy/n/nd610a9ed1f9b)
