---
title: "【picoCTF】RED — メタデータの詩がアクロスティックで「CHECKLSB」と言っていた"
emoji: "🔴"
type: "tech"
topics: ["ctf", "picoctf", "forensics", "steganography", "security"]
published: true
---

真っ赤な128×128のPNG画像。目で見ても何も分からない。でも答えはちゃんと「そこ」に書いてあった。

## やった失敗

最初にsteghideを試した。「PNG画像にフラグが隠されている」→「ステガノグラフィならsteghideでしょ」という反射的な判断。

結果：steghideはJPEGとBMP専用なのでエラーで終了。

次にbinwalkを試した。zlibストリームが見つかった（これはPNGの通常の圧縮データ）。「なんか見つかった！」と思ったが、それはただの画像データだった。

2つのツール、2つの失敗、5分の無駄。

## 本当の答えはexiftoolにあった

`exiftool red.png` を実行すると、予想外のフィールドが出てきた：

```
Poem : Crimson heart, vibrant and bold,
       Hearts flutter at your sight.
       Evenings glow softly red,
       Cherries burst with sweet life.
       Kisses linger with your warmth.
       Love deep as merlot.
       Scarlet leaves falling softly,
       Bold in every stroke.
```

カスタムメタデータフィールドに詩が埋め込まれていた。各行の最初の文字を並べると……

**C-H-E-C-K-L-S-B = "CHECKLSB"**

アクロスティック（行の頭文字で作るメッセージ）。「LSBを確認しろ」という直接的な指示が詩に隠されていた。

## zstegのインストールで詰まる

LSB解析の定番ツール `zsteg` はaptに入っていないのでRubyGem経由でインストールする。ただし依存ライブラリを先に入れないと2段階のエラーが出る：

1. `ruby-dev` なし → `mkmf.rb can't find header files for ruby`
2. `libmagickwand-dev` なし → `No such file or directory - MagickWand.h`

正解の順番：
```
sudo apt install ruby ruby-dev imagemagick libmagickwand-dev
sudo gem install zsteg
```

## zstegの出力を読む

`zsteg red.png` を実行すると複数行出てくる。「OpenPGP Public Key」「OpenPGP Secret Key」という表示があって最初驚くが、これは偽陽性。単色に近い画像のビットパターンが偶然そのmagic bytesに一致しているだけ。

本物はこの行：
```
b1,rgba,lsb,xy .. text: "cGljb0NURntyM2RfMXNf..."
```

Base64の文字列が出てくる。デコードするとフラグ。

## この問題から学んだこと

「ツール選択より先にメタデータを確認する」という習慣ができた。exiftoolは1コマンドで全メタデータを出してくれる。steghideやbinwalkより先に確認すべきだった。

また、「赤い詩は装飾ではなく道具だった」という発想の転換。チャレンジ名が "RED" で、詩も赤いテーマ——そのテーマの統一感が答えへの注意を逸らす罠になっていた。

## 詳細記事

exiftool実出力・zsteg依存エラーのコマンド出力・LSBの容量計算・偽陽性の技術的解説を含む英語記事：

→ [RED picoCTF Writeup](https://alsavaudomila.com/red-picoctf-writeup/)
