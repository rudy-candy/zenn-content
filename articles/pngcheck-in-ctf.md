---
title: "CTFで使うpngcheck — PNGが開けないときにチャンク構造を検証する"
emoji: "🖼️"
type: "tech"
topics: ["ctf", "forensics", "png", "linux", "security"]
published: true
---

CTFでPNGファイルが渡されたとき、画像として開けない・表示が崩れるという状況は「意図的に壊してある」か「データが隠されている」サインです。pngcheckはPNGの内部構造を検証して、どのチャンクに問題があるかを特定します。

## PNGの構造とpngcheck

PNGファイルはチャンクと呼ばれるブロックの連結で構成されています。IHDRで始まりIENDで終わり、その間にIDAT（画像データ）などが続きます。

pngcheckはこのチャンク構造を検証して、CRCエラー・不正なチャンク順序・不明なチャンクタイプなどを報告します。

```
pngcheck -v flag.png
```

エラーが出た場所が「問題が仕掛けられている場所」です。

## steghideより先にpngcheckを使う理由

PNGチャンクが壊れた状態でsteghideや他のツールを使っても正常動作しません。pngcheckで構造を確認してから他のツールに進むのが効率的です。

## セキュリティ的な意味

PNGファイルは不正なチャンクを追加することでステガノグラフィに利用できます。ファイルサイズの割に画像が小さい・表示されない領域がある、といった異常をpngcheckが検出することがあります。不審なPNG画像の調査で使われる手法です。

## 詳細記事

チャンクエラーの読み方・修復手順・CTFでの実例を含む英語記事はこちら：

→ [pngcheck in CTF](https://alsavaudomila.com/pngcheck-in-ctf-how-to-analyze-and-repair-png-files/)
