---
title: "【picoCTF】Wireshark doo dooo do doo — 直接検索が失敗した理由"
emoji: "🦈"
type: "tech"
topics: ["ctf", "picoctf", "cylabacademy", "wireshark", "forensics"]
published: false
---

pcapファイルを渡されて「フラグを見つけろ」という問題です。最初の操作で詰まりました——Ctrl+Fで「picoCTF」を検索したら、ヒットしなかった。

## 最初の失敗

Wiresharkを開いて、まずCtrl+Fでフラグを直接検索した。Packet Bytes、Strings、全部試した。何も出ない。

「ファイルが間違ってる？」と一瞬思ったけど、チャレンジのダウンロードファイルをそのまま開いてる。問題ない。

そこで気づく——フラグが**エンコードされている**から検索に引っかからない。

## HTTPフィルターとFollow TCP Stream

`http` でフィルターをかけると、パケット数が一気に絞られる。HTTP 200 OKのレスポンスを探して、右クリック→Follow→TCP Stream。

TCP Streamは、分割されたパケットを1つの会話に再組み立てしてくれる機能。これを使わないとHTTPのボディを読むのが面倒になる。

レスポンスのボディにこんな文字列が出てきた：

```
cvpbPGS{c33xno00_1_f33_h_qrnqorrs}
```

フラグっぽい構造（波括弧・アンダースコア）なのに、文字が違う。

## ROT13の見分け方

最初の文字「c」を13個ずらすと「p」。「cvpb」→「pico」。確定。

```bash
$ echo 'cvpbPGS{c33xno00_1_f33_h_qrnqorrs}' | tr 'A-Za-z' 'N-ZA-Mn-za-m'
picoCTF{p33kab00_1_s33_u_deadbeef}
```

ROT13の識別ポイント：
- 英字のみずれている（数字・記号はそのまま）
- 波括弧やアンダースコアが生き残っている → フラグ構造が見える
- 最初の1文字を13ずらしてみて「p」「i」「c」になればROT13確定

## フラグの意味

`p33kab00_1_s33_u_deadbeef` = 「peekaboo, I see you, deadbeef」

「隠れんぼしてたけど見つけたよ、deadbeef（デバッグ定数）」——ネットワークトラフィックに隠れていたフラグを見つけるという問題の趣旨がそのままフラグに入っている。

## 覚えておくこと

- **Ctrl+F失敗 = エンコードを疑う**：ROT13・Base64・hexを順番に試す
- **Follow TCP Stream**：Wiresharkでボディを読むときの基本操作
- **ROT13の識別**：フラグ構造（波括弧）が残っていて英字だけ違う → 即ROT13

## 詳細記事

フィルター手順・FollowTCPStreamの操作・ROT13コマンドの完全解説は英語記事：

→ [Wireshark doo dooo do doo picoCTF Writeup](https://alsavaudomila.com/wireshark-doo-dooo-do-doo-picoctf-writeup/)
