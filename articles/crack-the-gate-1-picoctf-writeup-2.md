---
title: "【picoCTF】Crack the Gate 1 — 問題名に釣られてブルートフォースを試みた話"
emoji: "🔐"
type: "tech"
topics: ["ctf", "picoctf", "cylabacademy", "web", "security"]
published: false
---

Web Exploitationの問題です。「Crack the Gate（ゲートを突破せよ）」という名前を見た瞬間、パスワードをクラックする問題だと思い込みました。これが罠でした。

## やらかした10分間

ログインページが出てきた。ユーザー名もパスワードも何も分からない。

「Crack the Gate」という名前のまま受け取って、BurpSuiteのIntruderを起動した。`admin/admin`、`admin/password`、`root/root` ——一通り試す。全部同じレスポンス。エラーメッセージは「ユーザー名が違う」と「パスワードが違う」を区別しない。

ユーザー名すら分からない状況でブルートフォースを続けるのは、暗闇でダーツを投げているのと同じだと気づくのに10分かかった。

## 正解はページのソースコードにあった

Ctrl+Uでページソースを開いたら、HTMLコメントがあった。

```
<!-- ABGR: Wnpx - grzcbenel olcnff: hfr urnqre "K-Qri-Npprff: lrf" -->
```

英字しか使われていない。数字がない、Base64でも16進数でもない。英字のみの暗号文で単語の区切りが自然に見えるとき——ROT13がほぼ確定。

Pythonで一行デコードする：

```python
import codecs
print(codecs.decode('ABGR: Wnpx - grzcbenel olcnff: hfr urnqre "K-Qri-Npprff: lrf"', 'rot_13'))
```

出力：

```
NOTE: Jack - temporary bypass: use header "X-Dev-Access: yes"
```

「Jack」が「一時的なバイパス」として残した開発用ヘッダーだった。本番環境に残ったまま。

## ヘッダーを付与してフラグを取る

BurpSuiteのRepeaterで `X-Dev-Access: yes` ヘッダーを追加してPOSTリクエストを送ると、認証なしでフラグが返ってくる。

```
picoCTF{brut4_f0rc4_83812a02}
```

フラグに `brut4_f0rc4`（ブルートフォース）と書いてある。最初に試みた間違ったアプローチがフラグになっていた。

## 学べること

- **ソースコードを先に見る習慣**：ログインページに当たったら、ツールを開く前にCtrl+Uでソースを確認する
- **ROT13の識別**：英字のみ・単語構造が見える → まずROT13
- **"security by obscurity"は機能しない**：ROT13でエンコードしても、ソースを見れば一瞬で分かる

チャレンジ名が「Crack the Gate」だったから最初にブルートフォースに手が伸びた。問題名はヒントとは限らない。

## 詳細記事

ROT13識別の具体的な確認方法・Burp Suite Repeaterの操作・curlによる代替手法を含む英語記事：

→ [Crack the Gate 1 picoCTF Writeup](https://alsavaudomila.com/crack-the-gate-1-picoctf-writeup-2/)
