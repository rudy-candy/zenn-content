---
title: "【picoCTF】Crack the Power — RSA暗号文を見ただけで攻撃パターンを見抜く"
emoji: "💥"
type: "tech"
topics: ["ctf", "picoctf", "crypto", "rsa", "security"]
published: true
---

Crack the PowerもRSA小指数攻撃の問題ですが、Mini RSAとは異なるケースです。この問題の面白さは「コードを実行する前に、数値を見ただけで攻撃パターンが分かる」という点にあります。

## 問題の概要

`e = 20` のRSA暗号文が渡されます。ヒントには「大きな素数で構成されている」「因数分解は推奨しない」とあります。

## この問題が教えてくれること

RSA小指数攻撃には2つのケースがあります：

- **ケース1（このChallenge）**: `m^e < N` → 剰余演算が発生しない → 直接べき根を取るだけ
- **ケース2（Mini RSA）**: `m^e ≥ N` → 剰余演算が発生 → k-iterationが必要

この問題では、暗号文 `c` の末尾の数字パターンから「剰余演算が発生していない」ことを事前に見抜けます。数値の性質を読む力がここで問われます。

また、最初はMini RSAと同じk-iterationコードで解こうとしましたが、ループが `i=1` で終了した。つまりループ自体が不要だったと気づいた瞬間が学びの核心でした。

## セキュリティ的な意味

RSAの安全性はeの大きさだけでなく、paddingの有無に依存します。padding（OAEP等）を使えば `m^e < N` になることを防げるため、この攻撃は無効化されます。

「paddingなしのRSAはなぜ危険か」を体感できる問題です。

## 詳細記事

暗号文の末尾から剰余発生を見抜く手法・2ケースの比較表を含む英語記事はこちら：

→ [Crack the Power picoCTF Writeup](https://alsavaudomila.com/crack-the-power/)
