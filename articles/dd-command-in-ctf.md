---
title: "CTFで使うdd — ディスクイメージから特定パーティションを切り出す"
emoji: "💾"
type: "tech"
topics: ["ctf", "forensics", "linux", "disk", "security"]
published: true
---

`dd` はバイト単位でデータをコピーするコマンドです。ディスクイメージから特定のパーティションだけを切り出したり、別のツールで解析しやすい形に変換したりするときに使います。

## CTFでddが必要になる場面

fdiskでパーティション構造を確認した後、目的のパーティションだけを取り出す作業にddを使います。

```
fdisk -l disk.img でオフセット（開始セクタ）を確認
→ dd if=disk.img of=part.img bs=512 skip=<開始セクタ> count=<セクタ数>
→ 切り出したpart.imgをmountや他のツールで解析
```

## 注意が必要な点

`dd` はエラーが出ても止まらず黙って処理を続けることがあります。`dd` の実行中は慎重にオフセットとサイズを確認する必要があります。誤ったオフセットで切り出すと、中身が壊れたイメージができますが、エラーは出ません。

## セキュリティ的な意味

フォレンジックの現場でもddはディスクイメージ取得の標準ツールです。`dd if=/dev/sda of=evidence.img` で証拠保全のイメージを作成します。書き込みブロッカーと組み合わせて使うのが基本ですが、コマンド自体の理解はCTFで身につきます。

## 詳細記事

bs・skip・countの計算方法・実際の切り出し手順を含む英語記事はこちら：

→ [dd in CTF](https://alsavaudomila.com/dd-in-ctf-disk-imaging-extraction-and-common-challenge-patterns/)
