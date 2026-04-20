---
title: "【picoCTF】DISKO 1 — stringsコマンド一発で終わる問題に1時間溶かした話"
emoji: "💾"
type: "tech"
topics: ["ctf", "picoctf", "forensics", "linux", "security"]
published: true
---

picoCTF の DISKO 1 を解くのに1時間かかりました。正解のコマンドは2行です。

```bash
gunzip disko-1.dd.gz
strings disko-1.dd | grep "pico"
```

それだけです。なのになぜ1時間もかかったかというと、「ディスクイメージ = マウントして中身を見る」という思い込みが邪魔をしたからです。

## 何をやったか（全部ハズレ）

1. `sudo mount -o loop disko-1.dd /tmp/disko` → ディレクトリが空
2. `binwalk disko-1.dd` → FAT ブートセクタしか検出されない
3. `fdisk -l disko-1.dd` → パーティション1個のみ、隠しパーティションなし
4. `fls -r disko-1.dd`（Sleuth Kit）→ 出力ゼロ

どれも間違いではないのですが、今回の問題には刺さりませんでした。

## なぜ全部ハズレだったのか

FAT16 はファイルシステムのディレクトリエントリとデータクラスタがセットで動いています。`mount` や `fls` はそのディレクトリエントリを読むツールです。

でも今回のフラグは、ファイルとして書かれていませんでした。**ディスクの生セクタに直接 ASCII テキストとして埋め込まれている**だけです。エントリがないので、ファイルシステムを意識するツールはすべて素通りします。

`strings` はそこが違います。ファイル構造を一切気にせず、バイナリ全体から印刷可能な文字列を引っこ抜きます。だからフラグが見つかった。

## フラグのテキストがヒントだった

`picoCTF{1t5_ju5t_4_5tr1n9_be6031da}`

leet speak で読むと「**it's just a string**」です。問題名もファイル名も、フラグ自体も、全部 `strings` を指差していました。気づいたのは解いた後でしたが。

## 次から使うフロー

ディスクイメージを受け取ったら、重いツールの前にまずこれ：

```bash
strings image.dd | grep "picoCTF{"
```

5秒で終わります。引っかかれば即終了、引っかからなければ mount や binwalk に進む。この順番を覚えておくと DISKO 1 系の問題で時間を無駄にしません。

## 詳細記事

実際のコマンド出力・FAT16 のセクタ構造・xxd でオフセットを確認する方法まで書いた英語記事はこちら：

→ [DISKO 1 picoCTF Writeup](https://alsavaudomila.com/disko-1-picoctf-writeup/)
