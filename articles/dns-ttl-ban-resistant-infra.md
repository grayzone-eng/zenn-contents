---
title: "BAN対策としてのDNS TTL戦略——風俗グループのインフラを30分で切り替えるための設計"
emoji: "⏱"
type: "tech"
topics: ["dns", "nginx", "docker", "infra", "devops"]
published: true
---

# BAN対策としてのDNS TTL戦略

一般的なWebサービスでは、DNS TTLは深く考えることなく設定される。Route 53のデフォルトは300秒、多くのレジストラのデフォルトは3600秒か86400秒だ。「キャッシュが長いほどDNSサーバーへのクエリが減ってパフォーマンスが上がる」という理解で、長めに設定するケースも多い。

しかし私が担当する業界では、TTLは事業継続に直結する数字だ。

---

## 背景：なぜBAN対策がインフラ設計の中心になるのか

アダルト産業・風俗業界のWebシステムが直面する最大のリスクは、**大手クラウドプロバイダーによる突然のアカウント凍結**だ。

- AWS・GCP・Azure：利用規約でアダルトコンテンツを明示禁止
- Cloudflare：アダルトサイトのアカウント削除事例が多数ある
- DigitalOcean：Terms of Serviceに明示的な禁止条項がある

これらのサービスは、通常24〜72時間前後の猶予すら与えずにアカウントを凍結することがある。その瞬間からサービスは停止し、バックアップを取り出す手段も失われる可能性がある。

この前提に立つと、インフラ設計の優先順位が根本的に変わる。

```
一般的な設計の優先順位:
1. コスト最適化
2. パフォーマンス
3. スケーラビリティ
4. 可用性

BAN対策を前提にした設計の優先順位:
1. 移行可能性（Portability）
2. 復旧速度（RTO: Recovery Time Objective）
3. 可用性
4. コスト
```

TTLはこの「復旧速度」に直接影響する数値だ。

---

## TTLとDNS伝播の仕組み

まず基本を整理する。

### TTLとは何か

TTL（Time To Live）は、DNSレコードをキャッシュしてよい時間（秒単位）を指定する値だ。

```
grayzone.me.  300  IN  A  152.42.194.21
              ^^^
              TTL（秒）
```

この設定の意味は「このAレコードを300秒間キャッシュしてよい」ということだ。

### キャッシュの連鎖

DNSクエリは以下の順で処理される。

```
ユーザーのブラウザ
  ↓
OSのDNSキャッシュ（stub resolver）
  ↓
ISPのDNSリゾルバー（フルサービスリゾルバー）
  ↓
権威DNSサーバー（NamecheapやRoute 53などのDNSサービス）
```

各レイヤーがTTLの時間だけキャッシュを保持する。TTLを300秒に設定すると、理論上は300秒後には全世界で新しいIPアドレスが参照されるが、実際には各リゾルバーのキャッシュタイミングがずれるため、完全な切り替えまで最大でTTL秒かかると考えておく必要がある。

### 切り替え時間の計算

```
BAN発生
  ↓
検知（監視ツール or 手動）: 〜5分
  ↓
代替VPSへの環境デプロイ: 〜15分（Docker Composeがあれば）
  ↓
DNSのAレコードを新IPに変更: 1分
  ↓
TTL分の伝播待ち: TTL秒
  ↓
サービス復旧
```

**合計 = 検知時間 + デプロイ時間 + TTL秒**

TTLが86400秒（24時間）の場合、最悪24時間はサービスが停止する。TTLが300秒なら、伝播待ちはわずか5分になる。

---

## 実際の設定

私のインフラでは以下のTTLを設定している。

### Namecheapでの設定

| Type | Host | Value          | TTL |
|------|------|----------------|-----|
| A    | @    | 152.42.194.21  | 300 |
| A    | www  | 152.42.194.21  | 300 |

300秒（5分）が実用上の下限だ。多くのDNSプロバイダーは60秒や120秒も設定できるが、あまり短くするとDNSサーバーへの負荷が増加し、一部のリゾルバーが無視するケースもある。300秒がバランスの取れた数字だ。

### 設定変更のタイミング

重要なのは、**BAN発生後にTTLを下げても遅い**という点だ。

TTLを短くする効果は、変更前にキャッシュを保持しているリゾルバーが古いキャッシュを破棄した後にしか現れない。つまり：

```
現在のTTL: 86400秒
↓
TTLを300秒に変更
↓
でもキャッシュは残り86400秒で上書きされない
↓
最大86400秒後にようやく300秒TTLのキャッシュが使われ始める
```

**TTLは平常時から短く保っておく必要がある。**

---

## Runbookとの組み合わせ

TTL戦略は単体では機能しない。30分以内の復旧を実現するには、以下がセットになっている必要がある。

### 1. 環境のコード化（IaC）

Docker Composeで環境を完全にコード化しておく。

```yaml
# docker-compose.yml
services:
  nginx:
    image: nginx:1.25-alpine
  wordpress:
    image: wordpress:6.5-php8.2-fpm-alpine
  mysql:
    image: mysql:8.0
  redis:
    image: redis:7-alpine
  certbot:
    image: certbot/certbot:latest
```

新しいVPSに `git clone` して `docker compose up -d` するだけで環境が再現できる状態を保つ。

### 2. 代替VPSの事前契約

BAN発生後に代替VPSを契約しても、審査・起動に時間がかかる。以下のようなサービスを候補として把握しておく。

| サービス    | 特徴                           |
|------------|-------------------------------|
| FlokiNET   | アイスランド・ルーマニア拠点。アダルトコンテンツ明示許可 |
| Hetzner    | ドイツ・フィンランド拠点。コストパフォーマンスが高い |
| Vultr      | 即時起動。世界各地にデータセンター |

理想は**常時1台のホットスタンバイ**だが、コストが課題なら**VPSの起動手順をRunbookに記録しておき5分で起動できる状態を維持する**だけでも大きく違う。

### 3. バックアップの外部保存

BAN時はVPSのデータへのアクセスも失う可能性がある。DBバックアップはVPS外に保存する。

```yaml
# docker-compose.ymlの一部
pgbackup:
  image: prodrigestivill/postgres-backup-local:16
  volumes:
    - ./backups:/backups  # ホスト側にマウント
  environment:
    SCHEDULE: "@daily"
    BACKUP_KEEP_DAYS: 7
```

さらにバックアップをBunny.netやRcloneでオフサイトに同期しておくと安全だ。

### 4. 監視とアラート

BANの検知が遅れると復旧も遅れる。最低限のHTTP監視を入れておく。

```bash
# cronで5分ごとに監視する例
*/5 * * * * curl -sf https://grayzone.me/wp-json/ || \
  curl -s -X POST "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
  -d "chat_id=${CHAT_ID}&text=🚨 grayzone.me が応答しません"
```

Telegram Botと組み合わせることで、障害発生から数分以内に通知が届く体制が作れる。

---

## 実際の切り替え手順（Runbook）

BAN発生時に実行する手順を事前に文書化しておく。

```markdown
## BAN発生時の対応手順

### 検知（目標: 5分以内）
- [ ] Telegram通知を確認
- [ ] curl https://grayzone.me で応答確認
- [ ] DigitalOceanダッシュボードでアカウント状態確認

### 代替VPS起動（目標: 15分以内）
- [ ] FlokiNETで新規VPS起動（Ubuntu 22.04）
- [ ] SSHキー設定
- [ ] git clone https://github.com/grayzone-eng/infra-iac
- [ ] .envを設定（パスワードマネージャーから取得）
- [ ] docker compose up -d
- [ ] バックアップからDBをリストア

### DNS切り替え（目標: 1分）
- [ ] NamecheapでAレコードを新VPSのIPに変更
- [ ] TTLは300秒のまま維持

### 確認（TTL経過後）
- [ ] dig grayzone.me +short で新IPを確認
- [ ] https://grayzone.me でサービス確認
- [ ] Telegram通知が止まっていることを確認
```

---

## TTLと証明書の注意点

DNS切り替え後、SSL証明書の再取得が必要になる。Let's Encryptのwebroot認証はDNSが新しいIPに向いていないと失敗するため、**DNS伝播を確認してからCertbotを実行する**手順にする必要がある。

```bash
# DNS伝播確認
dig grayzone.me +short
# 新しいIPが返ってきたら証明書取得
docker compose run --rm --entrypoint certbot certbot certonly \
  --webroot --webroot-path=/var/www/certbot \
  --email grayzone.eng@proton.me --agree-tos --no-eff-email \
  -d grayzone.me -d www.grayzone.me
```

なお、Let's Encryptには**週あたり同一ドメインへの発行制限（5枚）**があるため、テストや練習は `--staging` フラグを使うこと。

---

## まとめ

DNS TTLは「キャッシュの設定」ではなく「事業継続の設計」だ。

| 項目             | 推奨値   | 理由                         |
|-----------------|---------|------------------------------|
| 通常運用時のTTL  | 300秒   | 伝播速度と負荷のバランス       |
| 最低TTL         | 60秒    | 多くのリゾルバーが尊重する下限 |
| 変更のタイミング | 平常時  | BAN後では遅い                 |

TTLを短く保つのは単体では意味をなさない。Docker ComposeによるIaC・定期バックアップ・Runbook・監視との組み合わせで初めて「30分以内の復旧」という目標が現実になる。

この設計思想は、アダルト産業に限らず、クラウドプロバイダーのポリシーリスクを抱えるあらゆるサービスに応用できる。

---

*関連リポジトリ: [infra-iac](https://github.com/grayzone-eng/infra-iac)*
