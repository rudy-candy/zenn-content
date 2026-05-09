---
title: "【picoCTF】Local Authority — パスワードがJavaScriptに書いてあった"
emoji: "🔑"
type: "tech"
topics: ["ctf", "picoctf", "cylabacademy", "web", "security"]
published: true
---

ログインフォームがあった。試しに `admin` / `admin` を入れたら弾かれた。次の手を考えようとしたところで気づいた——パスワードはすでに自分のブラウザに届いていた。

## サーバーはパスワードを知らない

Burp Suiteでログインのリクエストを見ていた。POST を送る → サーバーが「失敗」と返す。その繰り返し。

でも HTTP History をスクロールしていると、ページ読み込み時にロードされた JavaScript ファイルがある。開いてみると：

```javascript
function checkPassword(username, password)
{
  if( username === 'admin' && password === 'strongPassword098765' )
  {
    return true;
  }
  else
  {
    return false;
  }
}
```

パスワードが書いてある。

ログインの検証はサーバーではなく、ブラウザの中で動く JavaScript がやっていた。つまりページを読み込んだ瞬間に、正しいパスワードが手元に届いていた。

## DevToolsでも同じことができる

Burp Suite なしでも、F12 → Sources タブ → JS ファイルをクリックするだけで同じコードが読める。

Ctrl+F で `password` や `===` を検索すると、認証チェックのロジックがすぐ見つかる。

## フラグが言いたいこと

`picoCTF{j5_15_7r4n5p4r3n7_05df90c8}` → `j5_15_7r4n5p4r3n7` = **JS is transparent**

JavaScript はブラウザが実行する。ブラウザはユーザーのマシンで動く。つまり中身は誰でも読める——サーバーサイドで検証しない限り、クライアントサイドに置いたロジックは「隠した」ことにならない。

## チャレンジ名の意味

「Local Authority」= 判断が local（ブラウザ）で行われる。名前がそのまま答えになっている。

## 詳細記事

Burp HTTP History の読み方・DevTools Sources の操作・実務でのクライアントサイド認証の危険性を含む英語記事：

→ [Local Authority picoCTF Writeup](https://alsavaudomila.com/local-authority-picoctf-writeup/)
