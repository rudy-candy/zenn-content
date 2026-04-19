---
title: "picoCTF Hidden in Plainsight — パスフレーズはJPEGのコメント欄にあった"
emoji: "🔍"
type: "tech"
topics: ["ctf", "picoctf", "forensics", "steganography", "security"]
published: true
---

「画像に何か隠されている」と分かっていても、どこを見ればいいか分からないことがある。picoCTF Hidden in Plainsightはそのギャップを突く問題だった。

## 最初の15分間でやった無駄なこと

まず画像を目で見た。普通の640x640のJPEG。次にsteghideをパスフレーズなしで実行した。失敗。binwalkも試した。何も出なかった。

実はこの時点で答えはすでに手元にあった。

## `file` コマンドの出力を最後まで読む

`file img.jpg` の出力の中に `comment: "c3RlZ2hpZGU6Y0VGNmVuZHZjbVE9"` という文字列があった。JPEGのコメントフィールドにbase64文字列が入っている。

デコードすると `steghide:cEF6endvcmQ=`。コロンの左がツール名、右がさらにbase64でエンコードされたパスフレーズ。もう一度デコードすると `pAzzword`。

パスフレーズは最初のコマンドの出力の中にあった。ただし二重にエンコードされていたので、一度目に気づいてもすぐには使えない構造になっていた。

## steghideで抽出

```
$ steghide extract -sf img.jpg
Enter passphrase: pAzzword
wrote extracted data to "flag.txt".
```

フラグ: `picoCTF{h1dd3n_1m4g3_67479645}`

## この問題から得た教訓

CTFでsteganographyの問題を見たとき、多くの人はすぐにsteghideやzstegを試す。でもパスフレーズが分からなければツールは動かない。パスフレーズを探す場所は「問題の外」ではなく「ファイルの中」にあることが多い。

`file` の出力、`exiftool` のメタデータ、`strings` で出てくる可読文字列——これらを丁寧に読む習慣が、こういう問題を解く鍵になる。

## 詳細記事

コマンドの全出力・試行錯誤の流れ・steghideの仕組みを含む英語記事はこちら：

→ [Hidden in Plainsight picoCTF Writeup](https://alsavaudomila.com/hidden-in-plainsight-picoctf-writeup/)
