---
title: "CTFでstringsコマンドを使うとき — 「フラグが見えた」の罠"
emoji: "🔍"
type: "tech"
topics: ["ctf", "forensics", "linux", "security", "picoctf"]
published: true
---

picoCTFの最初の問題で `strings` を実行したら、フラグがそのまま出てきた。「CTF余裕じゃん」と思ったのは最初の5分だけで、次の問題では同じことをやっても何も出てこなかった。

`strings` は万能ツールではない。「何が出るか」より「何が出ないか」を知っておく方が競技では役に立つ。

## 最初にstringsを使うべき理由

picoCTFの [DISKO 1](https://alsavaudomila.com/disko-1-picoctf-writeup/) というチャレンジで、生のディスクイメージをもらった。マウント、binwalk、Sleuth Kitを1時間試した後、ようやく試したのがこれ：

```bash
strings disko-1.dd | grep "pico"
picoCTF{1t5_ju5t_4_5tr1n9_be6031da}
```

1秒で終わった。フラグはファイルシステムの外、生のディスクセクタに直接書かれていた。マウント系のツールはファイルシステムを読む仕組みなので、そもそもアプローチが間違っていた。

[Corrupted File](https://alsavaudomila.com/corrupted-file-picoctf-writeup/) でも同じだった。壊れたファイルを開いた最初の10秒でこれを走らせたら：

```bash
strings corrupted_file | head -20
JFIF
Exif
picoCTF{r3st0r1ng_th3_by73s_b67c1558}
```

フラグが出てきた。おまけにJFIF・Exifマーカーで「これはJPEGで、ヘッダが壊れている」という診断まで得られた。

## すぐ試すべき場面

未知のバイナリ・メモリダンプ・拡張子不明のファイルをもらったら、最初の60秒でこれを走らせる：

```bash
strings challenge.bin | grep -i "ctf{"
strings -e l challenge.bin | grep -i "ctf{"   # Windows系バイナリ向け
```

`-e l` はUTF-16 LE（リトルエンディアン）のUnicode文字列を取り出すオプション。Windowsの実行ファイルは内部でUnicodeを使っていることが多く、これをやらないとフラグが見えない。1回見落としてから毎回チェックするようになった。

## 「フラグが見えた」が罠になるとき

picoCTFのリバース問題で `strings` を実行したら、フラグ形式の文字列が出てきた。即提出したら不正解。バイナリに意図的に埋め込まれたデコイだった。

本物のフラグかどうかを確認するには、前後の文字列を見る。`"Wrong password."` や `"Congratulations!"` のような処理結果のメッセージと同じ場所にあるなら本物の可能性が高い。単独で浮いていたら疑った方がいい。

## noiseを減らす

デフォルトは4文字以上の文字列を全部出力するので、コンパイラのメタデータや断片的なバイト列が大量に混ざる。最低長を上げるとすっきりする：

```bash
strings -n 8 challenge.bin | grep -i "flag\|ctf\|key\|pass"
```

ただし短いフラグを使う大会では `-n` を上げすぎると見落とす。picoCTFはフラグが長い傾向があるので `-n 8` でほぼ問題ない。

## stringsで見つからなかったら

- フラグがXOR・AESで暗号化されている → `strings` では見えない（binwalkやGhidraへ）
- バイナリにファイルが埋め込まれている → `strings` は `PK` の文字列は出るが抽出は `binwalk`
- LSBステガノグラフィ → `zsteg`
- Windowsバイナリで空振り → `strings -e l` を試す

`strings` が空振りしても「フラグがない」という意味ではない。入口が違うだけ。

## 詳細記事

実際のコマンド出力・オプション比較表・ワークフローを含む英語記事はこちら：

→ [strings Command in CTF: Hidden Data Guide](https://alsavaudomila.com/strings-command-in-ctf-how-to-extract-hidden-data-from-binaries/)

実際にstringsが活躍したwriteupはこちら：

- [DISKO 1 picoCTF Writeup](https://alsavaudomila.com/disko-1-picoctf-writeup/) — 生ディスクセクタに隠されたフラグ
- [Corrupted File picoCTF Writeup](https://alsavaudomila.com/corrupted-file-picoctf-writeup/) — 壊れたファイルからフラグを取り出す
