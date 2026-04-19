---
title: "CTFで使うsteghide — パスフレーズはメタデータの中にあった"
emoji: "🔏"
type: "tech"
topics: ["ctf", "picoctf", "forensics", "steganography", "security"]
published: true
---

「パスフレーズが分からない」と詰まっているとき、答えはすでに手元にあることがある。picoCTF の Hidden in Plainsight でそれを学んだ。

## パスフレーズを探す前に file を読む

ダウンロードした `img.jpg` に対してまず `file` を実行すると、出力の中に `comment: "c3RlZ2hpZGU6Y0VGNmVuZHZjbVE9"` という文字列があった。JPEGのコメントフィールドにbase64文字列が入っている——これは明らかに怪しい。

デコードすると `steghide:cEF6endvcmQ=`。さらに右側をデコードすると `pAzzword`。パスフレーズが二重base64でメタデータに埋め込まれていた。`steghide extract` を実行する前に、答えはすでに `file` コマンドの出力にあった。

## steghideを使う・使わない判断

steghideが有効なのはJPEGまたはWAVファイルのとき。PNGには対応していないので、PNG画像が渡されたらzstegに切り替える。

binwalkで何も見つからなくても諦めないこと。steghideはDCT係数を操作してデータを埋め込むため、ファイル末尾への付加と違いbinwalkのシグネチャ検出では発見できない。

パスフレーズの当たりがつかないときはStegseekでブルートフォースをかける。Stegcrackerより圧倒的に速い。

## セキュリティ的な背景

steghideはマルウェアがC2通信を隠すために使われた事例がある。一見普通のJPEGファイルに命令を埋め込み、URLフィルタリングをすり抜ける手法だ。インシデント対応でJPEGファイルが大量に送受信されていた場合、steghideで中身を確認することがある。

## 詳細記事

Hidden in Plainsightの完全なソルブフロー（実コマンド出力・二重base64デコード・フラグ取得まで）とStegseekへの切り替え判断を含む英語記事はこちら：

→ [steghide in CTF: Extract Flags](https://alsavaudomila.com/steghide-in-ctf-how-to-hide-and-extract-data-from-files/)
