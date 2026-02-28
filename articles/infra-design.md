---
title: "大手クラウドが使えない業界でのインフラ設計——BANリスクとポータビリティの両立"
emoji: "🛡"
type: "tech"
topics: ["docker", "nginx", "redis", "postgresql", "infra"]
published: true
---

# 大手クラウドが使えない業界でのインフラ設計

AWSもCloudflareも使えない。でもWebサービスを安定して動かさなければいけない。

この制約の下でインフラを設計したとき、「BANリスクをどう評価するか」と「いつでも移行できるポータビリティをどう確保するか」という2つの問いが軸になった。この記事では、その設計判断のプロセスを整理する。

---

## BANリスクの正確な評価

「アダルトコンテンツだからAWSが使えない」という話をすると、「実際にBanされた事例はあるの？」という反応がある。正確に言うと、問題は2層ある。

**第1層: 利用規約の問題**

AWS、Cloudflare、Vercel等の主要クラウドサービスは、利用規約でアダルトコンテンツを明示的に禁止している。これは「グレーゾーン」ではなく明確なTOS違反だ。

**第2層: 予告なし凍結のリスク**

TOS違反が確認された場合、凍結は予告なく実行される。

```
通常の障害: 原因調査 → 修正 → 復旧
BAN発生時: 凍結（予告なし）→ データアクセス不可 → 復旧不能の可能性
```

通常の障害との決定的な違いは、**データへのアクセス自体が失われる可能性がある**点だ。S3のバケットごと凍結された場合、画像・動画データが取り出せなくなるリスクがある。

---

## サービス選定の軸

この前提で、代替インフラを選定する際の軸を3つに絞った。

### 1. アダルトコンテンツへの明示的な許可

「禁止していない」ではなく「明示的に許可している」サービスを選ぶ。

| サービス | 判定 | 根拠 |
|---------|------|------|
| **FlokiNET** | ✓ 明示許可 | TOS内でアダルトコンテンツを明示許可。アイスランド・ルーマニア拠点。 |
| **Bunny.net** | ✓ 明示許可 | CDN/ストレージ共にアダルトOKを明記。EU拠点でDMCA対応実績あり。 |
| **Hetzner** | △ グレー | 明示禁止はしていないが、明示許可もしていない。コンテンツ非保存のサーバに使用。 |
| **AWS** | ✗ 禁止 | TOS § 3.2 で明示禁止。 |
| **Cloudflare** | ✗ 禁止 | アカウント削除事例が多数報告されている。 |

### 2. データの分離

「どのサービスがBANされても最小限の被害で済む」構成にする。

```
Vercel        — Next.jsフロントエンド（静的ファイルのみ）
               コンテンツを乗せないのでBAN対象外
    ↓
FlokiNET VPS  — WordPress（APIサーバ）+ PostgreSQL
               BANされても新VPSに30分で移行できる
    ↓
Bunny.net     — 画像・動画CDN
               BANされてもURLの向き先を変えるだけ
```

各レイヤーが独立しているため、1つがBanされても他のレイヤーに影響が及ばない。

### 3. 移行コストの最小化

上記の分離設計に加えて、**インフラをコードで管理する（IaC）**ことが移行コスト削減の核心だ。

---

## Docker Composeによるポータビリティ設計

環境をDockerで管理する最大の理由は、「**どのLinuxサーバでも同じ環境を再現できる**」ことだ。

```yaml
services:
  nginx:
    image: nginx:1.25-alpine

  wordpress:
    image: wordpress:6.5-php8.2-fpm-alpine
    depends_on:
      postgres:
        condition: service_healthy  # PostgreSQL起動確認後に起動

  postgres:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s

  redis:
    image: redis:7.2-alpine
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --maxmemory 128mb
      --maxmemory-policy allkeys-lru  # メモリ上限に達したらLRUで削除
```

:::message
**PostgreSQLを選んだ理由**

WordPressのデフォルトはMySQLだが、PostgreSQLを選んだ理由は3つある。
1. Next.js / Prismaとの親和性が高い（シフト管理等の業務ツールと同一DBで管理できる）
2. JSON型・全文検索等の機能が豊富
3. OSSの移行コストが同等
:::

---

## Redisの役割分担

認証システムにRedisを導入したのは、単純な「高速化」のためではない。**役割の明確化**が目的だ。

```
PostgreSQL  — 永続データ（シフト・ユーザー・業務記録）
Redis       — 揮発性データ（トークン・カウンタ・キャッシュ）
```

具体的には以下の用途に使っている。

**リフレッシュトークンの管理**

```typescript
// Before（PostgreSQL）:
// INSERT INTO app.refresh_tokens (user_id, token, expires_at) VALUES (...)
// → 期限切れレコードの削除バッチが必要
// → SELECTのたびにINDEXを使ったクエリが発生

// After（Redis）:
await redis.setEx(
  `refresh_token:${token}`,
  60 * 60 * 24 * 7,  // 7日（TTLで自動削除）
  JSON.stringify({ userId, storeId, role })
)
// → O(1)のルックアップ
// → TTL期限切れをRedisが自動で処理
```

**ログインのレートリミット**

```typescript
// IPアドレス単位でログイン試行回数をカウント
const count = await redis.incr(`rate_limit:login:${ip}`)
if (count === 1) {
  await redis.expire(`rate_limit:login:${ip}`, 60 * 15)  // 15分
}

if (count > 10) {
  return res.status(429).json({ error: 'ログイン試行回数が多すぎます' })
}
```

メモリ上の変数で管理すると複数プロセス・複数サーバ間で共有できないが、Redisに移すことで水平スケールに対応できる。

---

## バックアップとRunbook

どれだけ設計を工夫しても、BAN発生時のオペレーションを事前に定義しておかないと復旧時間は伸びる。

Runbookとして以下を用意した。

```bash
# scripts/restore.sh runbook を実行すると以下を表示

[ ] Step 1 (0〜5分):  新しいVPSを起動
[ ] Step 2 (5〜10分): リポジトリをクローンして .env を設定
[ ] Step 3 (10〜15分): sudo ./scripts/setup.sh でDockerインストール〜SSL取得
[ ] Step 4 (15〜20分): ./scripts/restore.sh db latest でDBリストア
[ ] Step 5 (20〜25分): Bunny.netのメディアは向き先URLを変えるだけ
[ ] Step 6 (25〜30分): 動作確認
```

:::details DNSの事前準備
BAN発生後のDNS切り替えを5分以内に完了させるには、**平常時からTTLを300秒（5分）以下に設定しておく**ことが必要だ。デフォルトの3600秒（1時間）のままでは、新しいIPへの切り替えに最大1時間かかる。

```
通常時: TTL = 300秒
BAN発生: DNSをすぐに新IPに向け替え → 5分以内に反映
```

ドメインレジストラはNamecheapを使用。WHOISプライバシー保護が無料で付いていることと、アダルト関連サイトへの利用での凍結リスクが低いことが選定理由だ。
:::

---

## セキュリティ設計の詳細

Nginxの設定で実装しているセキュリティ対策を整理する。

```nginx
# レートリミット定義
limit_req_zone $binary_remote_addr zone=wp_login:10m rate=5r/m;
limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;

# セキュリティヘッダー
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;

# アップロードディレクトリでのPHP実行を禁止
# シェルアップロード攻撃（Webシェル）への対策
location ~* /uploads/.*\.php$ {
    deny all;
    return 403;
}

# wp-config.phpへの直接アクセスを禁止
location ~ wp-config\.php {
    deny all;
    return 403;
}
```

「アップロードディレクトリでのPHP実行禁止」は見落とされやすい設定だ。攻撃者がPHPファイルを画像としてアップロードし、直接実行するWebシェル攻撃への対策として必須になる。

---

## まとめ

大手クラウドが使えない環境でのインフラ設計において、重要な判断軸は「**いつBanされても30分以内に復旧できるか**」だった。

そのための設計として、以下を組み合わせた。

1. アダルトを明示許可したサービスのみを選定（FlokiNET・Bunny.net）
2. コンテンツ・アプリ・CDNを分離してBanの影響範囲を局所化
3. Docker ComposeによるIaCで環境の再現性を確保
4. Redisで揮発性データをPostgreSQLから分離
5. Runbookと定期的なバックアップで復旧手順を事前定義

「BanされたらどうするのかではなくBanされても30分で戻れるか」という問いへの答えが、この設計の核心にある。

:::details 参考: コスト比較（月額概算）
| 構成 | 月額 |
|------|------|
| AWS構成（EC2+S3+CloudFront） | $150〜400 |
| 本構成（FlokiNET+Bunny.net） | $60〜120 |

Bunny.netの転送コストはAWS CloudFrontの約1/10。動画配信が発生する場合のコスト差は特に大きい。
:::
