---
title: "CTFで使うsteghide — パスフレーズはメタデータの中にあった"
emoji: "🔏"
type: "tech"
topics: ["ctf", "picoctf", "forensics", "steganography", "security"]
published: true
---

:::message
2026年10月追記：picoCTF は CyLab Security Academy（learn.cylabacademy.org）に名前が変わり、フラグの形式も `picoCTF{...}` から `academy{...}` に変わりました。この記事のフラグとコマンドは当時のものです。今の配布ファイルで解くときは、`picoCTF{` を `academy{` に読み替えてください（当時のフラグは今は通りません）。
:::

steghideには知らないと絶対にハマる罠がある。「パスフレーズが分からない」と詰まっているとき、答えはすでに手元にあることがある。picoCTF の Hidden in Plainsight でそれを学んだ。

## 同じエラーが2つの意味を持つ

steghideの最大の落とし穴は、エラーメッセージが状況を教えてくれないことだ。

- 間違ったパスフレーズを入れたとき → `steghide: could not extract any data with that passphrase!`
- データが何も埋め込まれていないファイルに使ったとき → `steghide: could not extract any data with that passphrase!`

全く同じメッセージが出る。「ツールが壊れているのか、パスフレーズが違うのか、そもそも何もないのか」を判断できない。正解は `steghide info` でまず確認すること。容量と埋め込みファイルの有無を教えてくれる。

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

手順とコマンド出力を全部書いた日本語の完全版は、noteに置いています（有料）。

→ [Steghideは「パスワードの壁」をどう超えるか？CTFステガノグラフィで差をつける実戦Workflow（note・¥500）](https://note.com/rudy_candy/n/n8f513074d5ce)

