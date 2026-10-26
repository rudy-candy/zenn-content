---
title: "PHP2Shellは自分の環境に効くのか — 安全に「脆弱か/直ったか/侵入されてないか」を確かめる"
emoji: "🔎"
type: "tech"
topics: ["php", "security", "脆弱性診断", "wordpress", "cve"]
published: true
---

:::message
前回の[PHP2Shell（CVE-2026-109618/109619）の解説記事](https://zenn.dev/rudy_candy/articles/php2shell-cve-2026-109618)の続編です。本記事は**攻撃の再現（シェル奪取の手順）は扱いません**。自分の環境が「脆弱か／直ったか／すでに侵入されていないか」を、**防御・診断の視点で安全に確かめる**手順をまとめます。検証はすべて、自分の管理下の環境・隔離コンテナで行っています。
:::

PHP2Shell（`session.upload_progress` のUAFによるRCE、CVSS 9.2）が出たとき、エンジニアが本当に知りたいのは「で、うちは大丈夫なのか」です。ここを、手を動かして確かめます。

## 確認①：自分のPHPは修正版か

まずバージョン。ただし落とし穴があって、**サーバーには複数のPHPが同居している**ことがあります。CLIで見た版と、実際にサイトを動かしているFPMの版は別物のことがある。

```bash
# CLI版（これだけ見て安心しない）
php -v

# 入っているFPMを全部
for f in /usr/sbin/php-fpm*; do echo "$f: $($f -v | head -1)"; done

# どのFPMがサイトを動かしているか（nginxのfastcgi_pass）
grep -rn fastcgi_pass /etc/nginx/
```

今回私が見た環境では、8.1 / 8.2 / 8.3 のFPMが全部起動していて、サイトは `php8.3-fpm.sock`＝**8.3系**が担当していました。見るべきは「サイトが使っている版」です。

修正版の目安はサポート中の8系（例：8.4.27、8.5.12、そして8.3系は**8.3.35**）。自分のブランチの修正版番号は php.net の案内で確認してください。EOLの 5.x / 7.x / 8.0 / 8.1 は公式修正がありません。

## 確認②：脆弱な機能（session.upload_progress）が有効か

PHP2Shellの成立には、`session.upload_progress` が有効であることが要ります。そして**これは初期設定で有効**です。

```bash
# 落とし穴：CLIのphp.iniとFPMのphp.iniは別ファイル
php --ini | grep "Loaded Configuration"   # ← これはCLIの話

# FPMの「実効値」を正しく見る（FPMのiniをCLIバイナリに読ませる）
php8.3 -c /etc/php/8.3/fpm/php.ini -i | grep session.upload_progress.enabled
```

`On => On` と出れば、脆弱な機能が有効な状態です。設定ファイルに明示行が無ければ「既定のOn」なので、`grep` で出てこない＝安全、ではない点に注意してください。

## 検証でわかった要点：修正版でも「機能は既定でOnのまま」

ここは手を動かして確かめました。修正版の PHP 8.3.35（Dockerの公式イメージ、10/21ビルド）で設定を見ます。

```bash
docker run --rm php:8.3-cli php -v
# → PHP 8.3.35 (cli) (built: Oct 21 2026 ...)

docker run --rm php:8.3-cli php -r \
  'echo ini_get("session.upload_progress.enabled"), "\n";'
# → 1   ← 修正版でも、機能は既定で有効(On)
```

つまり、**「更新」と「Off」は役割が違います**。

- **更新（8.3.35へ）**：バグ（UAF）そのものを塞ぐ。これが本命
- **Off（`session.upload_progress.enabled = Off`）**：脆弱な機能の入口を閉じる緩和。更新までの時間稼ぎ、または多層防御

修正版に上げても機能はOnのままなので、「更新したから設定は触らなくていい」ではなく、使っていないなら両方やっておくのが堅いです。緩和が効くことも確認できます。

```bash
docker run --rm php:8.3-cli php -d session.upload_progress.enabled=0 -r \
  'echo ini_get("session.upload_progress.enabled"), "\n";'
# → 0
```

WordPressはこの機能を使わないので、WPサイトなら `Off` にしても実害はありません。

## 多層防御：nginxのバッファリング

nginx + PHP-FPM 構成では、`fastcgi_request_buffering`（**既定でon**）が効いていると、攻撃に必要な「途中で止まるリクエスト」がFPMまで届きにくく、成立条件を崩せます。

```bash
# off に明示していないか確認（無ければ既定のon＝緩和が効く）
grep -rniE "fastcgi_request_buffering\s+off" /etc/nginx/ || echo "off指定なし＝既定on"
```

これはあくまで一枚の防御なので、これに依存せず、更新とOffで根本を塞ぐのが前提です。

## 「もう侵入されていないか」を確かめる（WordPress前提の診断の型）

脆弱性が出たら、「これから塞ぐ」と同時に「もう入られていないか」を見ます。root無しでも、ここまでは確認できます（WP-CLIとadmグループでのログ閲覧を利用）。

```bash
# 1. コアファイルの改ざん（公式チェックサムと照合）
wp core verify-checksums --path=/var/www/wordpress

# 2. uploads配下の .php（Webシェルの定番置き場。基本ゼロのはず）
find /var/www/wordpress/wp-content/uploads -name '*.php'

# 3. 直近に変更された .php（プラグイン更新を除いて精査）
find /var/www/wordpress -name '*.php' -mtime -10

# 4. 管理者アカウント（身に覚えのない管理者がいないか）
wp user list --role=administrator --fields=ID,user_login,user_registered --path=/var/www/wordpress

# 5. アクセスログのシェル探索（返りコードが鍵）
zcat -f /var/log/nginx/access.log* | grep -iE '/wp-content/uploads/.*\.php'
#    → 全部 404 なら「探されたが存在しない」。200 があればシェル設置を疑う

# 6. POSTの5xx（exploit失敗やクラッシュの痕跡）
zcat -f /var/log/nginx/access.log* | awk '$6 ~ /POST/ && $9 ~ /^5/'

# 7. cron（常駐の仕込み）
crontab -l; ls -l /etc/cron.d/
```

実際にこれを回した環境では、`verify-checksums` 成功・管理者は本人1名・uploadsにシェルなし・シェル探索は**全部404**・cronは既知のものだけ、で**侵入痕跡なし**と判断できました。

### 限界も正直に

- **FPMのエラーログ（segfault痕跡）はroot権限が要る**：`sudo grep -iE "signal 11|segfault" /var/log/php*-fpm.log*`
- ログはローテートで一定期間しか残らない。ただしPHP2Shellは数日前の脆弱性なので、直近ログで足りる
- 「痕跡なし」は100%の無罪証明ではない。ただし複数の独立チェックが揃えば、実務上は十分な確度になる

## まとめ：確認チェックリスト

- [ ] サイトを動かしている**FPMの版**を特定し、修正版か確認（CLI版だけ見ない）
- [ ] `session.upload_progress.enabled` の**実効値**をFPMのiniで確認（明示が無い＝既定On）
- [ ] 更新（修正版へ）とOff（緩和）は役割が別。使っていないなら両方
- [ ] nginxの `fastcgi_request_buffering` が既定onのまま（offにしていない）か
- [ ] 侵入確認：checksums／uploadsの.php／管理者／ログの404／POSTの5xx／cron
- [ ] EOLのPHP（8.1以下）は移行を検討

脆弱性対応は「塞ぐ」と「入られていないか見る」の両輪です。どちらも、攻撃を再現しなくても、ここまで確かめられます。
