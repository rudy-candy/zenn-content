---
title: "CTFで使うpdfdumper — PDFは「見えているもの」がすべてではない"
emoji: "📄"
type: "tech"
topics: ["ctf", "forensics", "pdf", "linux", "security"]
published: true
---

PDFファイルはテキストや画像を表示するだけでなく、JavaScript・埋め込みファイル・隠しストリームなど多くのオブジェクトを含めることができます。pdfdumperはそれらを一覧・抽出するためのツールです。

## PDFの構造とCTF

PDFを開いて「何も書いていない」ように見えても、内部には：

- 白い背景に白いテキストで書かれた文字
- 埋め込まれた別のファイル（ZIPやPDFなど）
- JavaScriptオブジェクト
- 圧縮されたストリームデータ

が含まれていることがあります。pdfdumperでこれらのオブジェクトを全部取り出して確認するのがCTFでのアプローチです。

## なぜテキストエディタでは不十分か

PDFはバイナリ形式で、圧縮されたストリームは人間が読める形ではありません。pdfdumperはこれを自動的に解凍・デコードして中身を見られる状態にしてくれます。

## セキュリティ的な意味

PDFはフィッシングやマルウェア配布でよく使われるファイル形式です。悪意あるPDFにはJavaScriptが埋め込まれていて、開いた瞬間にコードを実行するものがあります。pdfdumperでPDFの内部構造を確認する技術は、不審なPDFの解析に直結します。

## 詳細記事

pdfdumperのオブジェクト一覧・ストリーム抽出の手順を含む英語記事はこちら：

→ [pdfdumper in CTF](https://alsavaudomila.com/pdfdumper-in-ctf-extracting-pdf-content-and-common-challenge-patterns/)
