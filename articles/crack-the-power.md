---
title: "【picoCTF】Crack the Power — RSA暗号文を見ただけで攻撃パターンを見抜く"
emoji: "💥"
type: "tech"
topics: ["ctf", "picoctf", "crypto", "rsa", "security"]
published: true
---

Crack the PowerもRSA小指数攻撃の問題ですが、Mini RSAとは異なるケースです。この問題の面白さは「コードを実行する前に、数値を見ただけで攻撃パターンが分かる」という点にあります。

## RSA小指数攻撃の2ケース

`e = 20` のRSAが渡されます。小指数攻撃を知っていれば「剰余演算が発生しているかどうか」を先に確認すべきです。

- **ケース1（このChallenge）**: `m^e < N` → 剰余演算が発生しない → 直接べき根を取るだけ
- **ケース2（Mini RSA等）**: `m^e ≥ N` → 剰余演算が発生 → k-iterationが必要

## ループがi=1で終了した瞬間の気づき

最初はMini RSAと同じk-iterationコードを書いて実行しました。ループカウンタを確認すると `i=1` で終了していました。

つまり、**ループ自体が不要だった**。

これはケース1の問題だったということです。剰余演算が発生していないから、直接 `gmpy2.iroot(c, e)` を呼ぶだけで解ける。

## 「剰余が発生しているか」の診断方法

最も簡単な確認は桁数比較です：

```python
print(len(str(c)))  # cの桁数
print(len(str(n)))  # nの桁数
```

cがnより大幅に少ない桁数なら、`m^e` が `n` を超えていない可能性が高い。その場合は `gmpy2.iroot(c, e)` で `exact=True` が返るはずです。

## セキュリティ的な意味

paddingなしのRSA小指数攻撃には、この直接べき根以外にもバリエーションがあります。

**Håstadのブロードキャスト攻撃**：同じ短いメッセージを同じ `e` で複数の相手に送った場合、それぞれの暗号文を中国余剰定理（CRT）でまとめると `m^e` が復元できる。OpenSSLの初期バージョンでは `e=3` がデフォルトで使われており、paddingなしの実装では実際に悪用されました。

「paddingがなぜ必要か」を体で理解できる問題です。

## 詳細記事

桁数診断・末尾桁パターン分析・Håstadブロードキャスト攻撃の詳細を含む英語記事はこちら：

→ [Crack the Power picoCTF Writeup](https://alsavaudomila.com/crack-the-power/)
