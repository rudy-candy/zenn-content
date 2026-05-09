---
title: "【picoCTF】Riddle Registry — 黒塗りPDFの答えは黒塗りの下にはなかった"
emoji: "📄"
type: "tech"
topics: ["ctf", "picoctf", "forensics", "pdf", "security"]
published: true
---

黒塗りされた機密文書PDF。直感的に「黒塗りを外せばフラグが出てくる」と思いました。その直感は間違いでした。

## やった失敗

`confidential.pdf` を開くと、所々黒いバーで覆われた文書が出てきます。

「PDFの黒塗りはオーバーレイ（文字の上に黒い四角を重ねただけ）の場合がある」というCTF知識を思い出して、テキスト選択→コピーを試みました。結果：何も取れない。

`strings confidential.pdf | grep -i flag` も試しました。生のPDF構造データが大量に出て、フラグパターンは見つからない。

## 答えはメタデータにあった

`pdfinfo confidential.pdf` を実行すると：

```
Author:          cGljb0NURntwdXp6bDNkX20zdGFkYXRhX2YwdW5kIV9lZTQ1NDk1MH0=
Producer:        PyPDF2
```

`Author` フィールドに末尾が `=` で終わる文字列。**これはBase64のパディング文字です。** 本物の著者名が `=` で終わることはありません。

`Producer: PyPDF2` というのも興味深い。Pythonライブラリで自動生成されたPDFということが分かります。

## デコード

```python
import base64
cipher = "cGljb0NURntwdXp6bDNkX20zdGFkYXRhX2YwdW5kIV9lZTQ1NDk1MH0="
print(base64.b64decode(cipher).decode())
# → picoCTF{puzzl3d_m3tadata_f0und!_ee454950}
```

フラグは文書の中ではなく、PDFのメタデータフィールドに隠されていました。

## この問題から学んだこと

**PDFに限らず、ファイルのメタデータは「見えない場所」として機能します。**

実際のセキュリティ事故でも、外部公開PDFのAuthorフィールドに社内ユーザー名が残っていたり、作成ツール名からソフトウェアスタックが推測されたりするケースがあります。

また、`=` や `==` で終わる文字列を見たらBase64を疑う、という反射を身につけました。メタデータ値・HTTPヘッダ・ログ・設定ファイル、どこに現れても同じです。

## 詳細記事

pdfinfo実出力・overlay redactionの仕組み・実世界のPDF漏洩事例を含む英語記事：

→ [Riddle Registry picoCTF Writeup](https://alsavaudomila.com/riddle-registry-picoctf-writeup/)
