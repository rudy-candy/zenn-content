---
title: "【picoCTF】Inspect HTML — 「見えない」と「ない」は別物だった"
emoji: "🔍"
type: "tech"
topics: ["ctf", "picoctf", "cylabacademy", "web", "security"]
published: false
---

ページを開いてクリックしても何もない。でもフラグはそこにあった——ブラウザが表示しないだけで。

## 最初にやった無駄な操作

Networkタブを開いて通信を確認した。怪しいリクエストはない。
Consoleタブを開いた。何も出力されていない。
Ctrl+Fでページ内を「flag」で検索した。ヒットなし。

5分間、フラグが「ない」と思って探し続けた。でも本当は最初からあった。

## HTMLコメントはブラウザに表示されない

F12でElementsタブを開いてHTMLをスクロールすると、こんな行が見つかる：

```html
<!--picoCTF{1n5p3t0r_0f_h7ml_1fd8425b}-->
```

`<!-- -->` で囲まれた部分はHTMLコメント。ブラウザはパースするけど表示しない。ページを眺めているだけでは絶対に見つからない。

Ctrl+Uでページソースを開いて、そこでCtrl+Fで「picoCTF」を検索する方法でも同じ結果が出る。

## フラグの意味

`1n5p3t0r_0f_h7ml` = inspector of html。

チャレンジの名前が「Inspect HTML」で、フラグが「inspector of html」。問題が解き方をタイトルで教えて、フラグが正解者だけに分かる形でもう一度答える——自己言及的な構造になっている。

## 覚えておくこと

- **Ctrl+F はレンダリング済みテキストを検索する**：HTMLコメント・`display:none`要素・`hidden`属性の中身はヒットしない
- **Elementタブ・Ctrl+UはHTMLそのものを見る**：コメントを含めて全部見える
- **HTMLコメントは実際の開発でも漏洩源になる**：APIキー・内部パス・デバッグ情報がコメントに残っているケースは本番環境でも起きている

## 詳細記事

DevToolsの各タブの使い分け・curlによる一括抽出・`display:none`との違いを含む英語記事：

→ [Inspect HTML picoCTF Writeup](https://alsavaudomila.com/inspect-html-picoctf-writeup/)
