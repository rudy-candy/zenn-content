---
title: "【picoCTF】Log Hunt — grepの罠は「重複ログエントリ」だった"
emoji: "🔍"
type: "tech"
topics: ["ctf", "picoctf", "forensics", "linux", "security"]
published: true
---

`server.log` というログファイルにフラグが分割して隠されています。grepで抽出したら、同じフラグフラグメントが複数回出てくる——これを罠だと気づけるかどうかが問題の核心です。

## 問題の構造

ログには `FLAGPART:` というラベル付きでフラグの断片が記録されています：

```
[1990-08-09 10:00:10] INFO FLAGPART: picoCTF{us3_
[1990-08-09 10:02:55] INFO FLAGPART: y0urlinux_
[1990-08-09 10:05:54] INFO FLAGPART: sk1lls_
[1990-08-09 10:05:55] INFO FLAGPART: sk1lls_   ← 1秒後にまた出る
[1990-08-09 10:10:54] INFO FLAGPART: cedfa5fb}
```

`strings server.log | grep "FLAGPART"` で全部抽出できます。

## sk1lls_ が2回出る理由

10:05:54 と 10:05:55 に `sk1lls_` が連続して出てきます。

これは**重複ログエントリ**——同じイベントが1秒以内に2回記録されたもの。実サーバーではリトライ処理や並行スレッドで普通に起こります。「フラグパーツが5個ある」ではなく「同じパーツが重複している」と判断するのが正解。

`sort -u` で一意な値だけ取り出すと4つ：

```bash
$ grep "FLAGPART" server.log | awk '{print $NF}' | sort -u
cedfa5fb}
picoCTF{us3_
sk1lls_
y0urlinux_
```

4つを最初の出現順に並べるとフラグが完成：
`picoCTF{us3_y0urlinux_sk1lls_cedfa5fb}`

「use your linux skills（Linuxスキルを使え）」——フラグ自体がこの問題の答えを言っています。

## 実務での対応

ログ解析で `grep` を使う場面は実際のインシデント対応でも頻繁にあります。

```bash
# 特定のIPからの失敗ログインを抽出
grep "Failed password" /var/log/auth.log | grep "192.168.1.100"

# 特定のUser-Agentのアクセスをタイムスタンプ順に整理
grep "ExploitBot" /var/log/nginx/access.log | sort | uniq
```

重複排除（`sort -u` / `uniq`）はここでも同じように使います。

## 詳細記事

strings・grep・sort -u の使い方・重複ログの仕組みを含む英語記事：

→ [Log Hunt picoCTF Writeup](https://alsavaudomila.com/log-hunt-picoctf-writeup/)
