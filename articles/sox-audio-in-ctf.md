---
title: "CTFで使うSoX — 音声ファイルをコマンドラインで解析・変換する"
emoji: "🎵"
type: "tech"
topics: ["ctf", "forensics", "audio", "linux", "security"]
published: true
---

SoXはコマンドラインで音声ファイルを解析・変換・加工できるツールです。Audacityが「見る」ツールならSoXは「測る」ツールです。

## `sox -n stat` のRough frequencyが教えてくれること

TJCTF 2024の「beep-boop-robot」という問題で確認した。WAVファイルを受け取り、名前から「DTMF（電話のタッチトーン）かな」と思って解析に入った。

```
$ sox robot.wav -n stat 2>&1 | grep "Rough"
Rough   frequency:                  483
```

DTMFなら700〜1600 Hzの範囲にくるはず。483 Hzは低すぎる。

実はこのファイルは約1000 Hzの搬送波を「ON/OFF」で切り替えているモールス符号だった。信号がOFFの時間帯はゼロクロスが発生しない。約50%のデューティサイクルで使われている場合、ゼロクロス数から計算されるRough frequencyは搬送波の半分（≒500 Hz）になる。483 Hzはこれと完全に一致する。

`sox stat`の数値を正しく読めれば、GUIを開く前にこの判断ができる。

## AudacityとSoXの使い分け

音声ファイルを渡されたとき：

1. `sox challenge.wav -n stat` で統計情報を確認（Rough frequency・RMSで変調方式を推定）
2. `sox challenge.wav -n spectrogram -o spec.png` でスペクトログラムを生成
3. 視覚的な詳細確認やインタラクティブな探索にはAudacity

SoXは速度変換・ピッチ変換・チャンネル抽出・無音区間カットなど、スクリプト化が必要な加工に向いています。

## セキュリティ的な意味

音声ファイルは監視カメラの録音・電話録音など証拠として使われることがあります。SoXのような変換・解析ツールはフォレンジック調査でも使われる技術です。モールス符号の検出はDTMFとの混同が多いので、Rough frequencyの読み方を知っておくと実用的です。

## 詳細記事

DTMFとモールスの判別方法・実コマンド出力・ツール比較表を含む英語記事はこちら：

→ [SoX in CTF](https://alsavaudomila.com/sox-in-ctf-how-to-analyze-and-manipulate-audio-files/)
