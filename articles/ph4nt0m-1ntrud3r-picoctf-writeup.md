---
title: "【picoCTF】Ph4nt0m 1ntrud3r — Wiresharkで見えないものをtsharkで見る"
emoji: "🌐"
type: "tech"
topics: ["ctf", "picoctf", "forensics", "wireshark", "security"]
published: true
---

ネットワークフォレンジックの問題では、パケットキャプチャから「通信の中に何が隠されているか」を探します。Ph4nt0m 1ntrud3rはその典型で、TCPストリームをフィルタリングしてBase64エンコードされたデータを復元する問題です。

## 問題の概要

`.pcap` ファイルが渡されます。大量のパケットの中から、フラグが分割されて複数のTCPパケットに隠されています。

## この問題が教えてくれること

**GUIのWiresharkとCLIのtsharkを使い分ける**という視点です。

Wiresharkで全体を把握してから、tsharkで特定の条件のパケットだけを抽出・処理するのが効率的なワークフローです。さらにPythonでBase64フラグメントを結合して復元するところまでが一連の流れです。

パケットキャプチャ解析では「何を探すか」が決まっていないと無限に時間がかかります。怪しいポートへの通信、異常に長いペイロード、Base64らしき文字列……こうした「アタリ」を素早く見つける感覚を養える問題です。

## セキュリティ的な意味

実際の侵入検知・インシデント対応でも、大量のパケットログから不審な通信を抽出する作業はtsharkのフィルタリングが基本です。フォレンジックエンジニアが日常的に使うスキルセットそのものです。

## 詳細記事

tsharkフィルタの書き方・PythonでのBase64復元コード・実際の出力を含む英語記事はこちら：

→ [Ph4nt0m 1ntrud3r picoCTF Writeup](https://alsavaudomila.com/ph4nt0m-1ntrud3r-picoctf-writeup/)
