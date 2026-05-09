---
title: "CTFで使うbinwalk — 大量のfalse positiveを読み解く"
emoji: "🔎"
type: "tech"
topics: ["ctf", "forensics", "binwalk", "linux", "security"]
published: true
---

BurnerCTF 2025の問題でbinwalkを実行したら、10件以上の検出結果が流れた。最初は「当たりだ」と思ったが、30分後にその全部がfalse positiveだったと気づいた。

picoCTF Matryoshka Dollでも似た罠にはまった。今度はTIFF偽陽性だ。

## 2種類のfalse positiveを知っておく

**DER証明書 false positive（ネットワーク系）**

PNGのIDATチャンクはzlibで圧縮されている。この圧縮データ中に、DER証明書のマジックバイト（`30 82`）と偶然一致するパターンが含まれることがある。binwalkは文脈を考慮せずにバイト列を照合するため、大量の「証明書らしいもの」を報告してしまう。

**TIFF false positive（Appleシステム由来のPNG）**

ICCカラープロファイルを埋め込んだPNGファイルでよく起きる。ICCプロファイルのヘッダが `4D 4D 00 2A`（ビッグエンディアンTIFFのマジックバイトと同一）で始まるため、binwalkが「TIFF image data」として報告する。

picoCTF Matryoshka Dollでbinwalkを実行すると、4層すべてのPNGでオフセット0xC9A（=3226バイト目）に同じTIFF検出が出る。これがICCプロファイルの痕跡だと気づくまで20分無駄にした。

**見分け方の核心**: 同じオフセットが複数のファイルで繰り返し出現するなら、それは構造的なアーティファクト。本当の埋め込みファイルは各ファイルでオフセットが変わる。

## 本物のシグナルを見つける判断基準

- ファイル先頭付近に密集した検出 → IDATチャンク内のfalse positiveを疑う
- ファイルサイズに近い後半オフセットに孤立した検出 → 末尾に追記されたファイルの可能性が高い
- 同種の検出が数バイト間隔で連続 → 単一の圧縮ブロックを走査している

`ls -la` でファイルサイズを確認してから、検出オフセットと比較するのが最初の手順。

## -e の罠

`binwalk -e` をそのまま実行すると、false positiveも含めて全部抽出されてフォルダがカオスになる。有望なオフセットが分かっているなら `dd` で手動抽出するか、`--dd='tiff image:tiff'` のように種別を絞る方が結果が読みやすい。

## binwalkを使わない場面

- LSBステガノグラフィ → `zsteg`（ビットレベルの操作はシグネチャに現れない）
- メタデータ確認 → `exiftool`
- PNGチャンク破損の診断 → `pngcheck`

binwalkが何も検出しなかったからといって、隠しデータがないとは言えない。

## 詳細記事

false positiveの仕組み・BurnerCTF 2025の実例・foremost/stringsとの使い分け比較表を含む英語記事はこちら：

→ [binwalk in CTF](https://alsavaudomila.com/binwalk-in-ctf-how-to-analyze-binaries-and-extract-hidden-files/)
