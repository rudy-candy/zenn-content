---
title: "CTFで使うAudacity — 音声ファイルを渡されたらまずスペクトログラムを見る"
emoji: "🎵"
type: "tech"
topics: ["ctf", "forensics", "audacity", "steganography", "security"]
published: true
---

CTFの音声Forensicsで最初にやることは決まっています。Audacityを開いて、スペクトログラム表示に切り替える。これだけで問題の半分が解ける場合があります。

## なぜスペクトログラムか

音声の「波形表示」は時間軸の振幅を見ます。スペクトログラムは周波数の変化を視覚化します。

CTFの問題では、人間の耳には聞こえない周波数帯（20kHz以上など）にデータや文字が隠されていることがよくあります。波形表示では何も見えませんが、スペクトログラムに切り替えると文字や画像が浮かび上がります。

## Audacityを使う判断基準

音声ファイルを渡されたときのフロー：

1. まずAudacityでスペクトログラム確認（視覚的なデータ隠蔽を確認）
2. 何も見つからなければsoxで統計情報を確認
3. ビット操作が疑われる場合はffmpegで別形式に変換して再確認

## セキュリティ的な意味

音声ステガノグラフィはマルウェアのC2通信で実際に使われた事例があります。通常の音声ファイルに見せかけて命令を送受信する手法です。Audacityのスペクトログラムで検出できるケースもあります。

## 詳細記事

スペクトログラムの設定方法・周波数帯の絞り方・CTF頻出パターンを含む英語記事はこちら：

→ [Audacity in CTF](https://alsavaudomila.com/audacity-in-ctf-audio-forensics-techniques-and-common-challenge-patterns/)
