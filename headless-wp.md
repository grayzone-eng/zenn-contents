---
title: "なぜWordPressをヘッドレス化するのか——業界特化型インフラ設計から考える"
emoji: "🏗"
type: "tech"
topics: ["wordpress", "nextjs", "docker", "nginx", "typescript"]
published: true
---

# なぜWordPressをヘッドレス化するのか

WordPressのヘッドレス化は、よく「パフォーマンス改善のため」と説明される。それは正しい。しかし私がこの構成を選んだ本当の理由は、**攻撃面の最小化**と**人的コストの維持**という、一見矛盾する2つの要件を同時に満たすためだった。

この記事では、一般的なクラウドサービスが利用しづらい業界でのWebシステム設計を通じて見えてきた、ヘッドレスWordPressの本質的な価値を整理する。

---

## 背景：なぜ普通の構成が使えないのか

多くのWebエンジニアがデフォルトで選ぶスタックがある。

```
AWS EC2 / Vercel + CloudFront + RDS MySQL + WordPress
```

機能的には十分で、ドキュメントも豊富だ。しかしアダルト産業・風俗業界では、これらのサービスの利用規約が壁になる。

- **AWS**: Terms of Service § 3.2 でアダルトコンテンツを明示禁止
- **Cloudflare**: アダルトサイトのアカウント削除事例が多数報告されている
- **Vercel**: 静的ファイルの配信は問題ないが、動画・画像のストレージには使えない

問題は「禁止されているから使えない」ではなく、**「いつ止まるかわからない」という不安定さがビジネスリスクになる**点だ。

:::message alert
アカウントのBANは予告なく発生する。止まった後で代替手段を探しても、営業損失は取り戻せない。
:::

---

## 選定したインフラ構成

この制約の中で私が選んだ構成は以下の通りだ。

```
ユーザー
    ↓
Nginx（リバースプロキシ・SSL終端）
    ↓
WordPress（PHP-FPM、Dockerネットワーク内部のみ）
    ↓
PostgreSQL（同じくDockerネットワーク内部）

コンテンツ配信:
Next.js（Vercel） ← WordPress REST API ← Docker内WordPress
画像・動画: Bunny.net CDN / Bunny Stream
```

**VPS**: FlokiNET（アイスランド拠点、アダルトOK）
**CDN**: Bunny.net（EU拠点、TOS明示許可）
**フロントエンド**: Vercel（静的生成のみ使用、コンテンツは乗せない）

Vercelを使うことに疑問を持つ人もいるかもしれない。しかしNext.jsのフロントエンドは静的なHTMLとJSを配信するだけで、センシティブなコンテンツは一切含まない。**問題になるのはコンテンツそのものであり、静的ファイルの配信サービスを使うこと自体ではない。**

---

## ヘッドレス化の本質的な価値

### 1. 攻撃面の最小化

従来のWordPressは、管理画面・フロントエンド・APIがすべて同一サーバに公開されている。

```
https://example.com/           → フロントエンド（公開）
https://example.com/wp-admin/  → 管理画面（公開）
https://example.com/wp-login.php → ログイン（公開）
https://example.com/xmlrpc.php   → 旧API（攻撃に悪用）
```

ヘッドレス化すると、外部に公開するエンドポイントが大幅に減る。

```nginx
# /wp-json/ のみ外部公開
location /wp-json/ {
    limit_req zone=api burst=20 nodelay;
    # ... PHP-FPMへプロキシ
}

# /wp-admin/ は管理者IPのみ
location /wp-admin/ {
    allow 203.0.113.0;  # 管理者IP
    deny all;
}

# xmlrpc.phpは完全ブロック
location = /xmlrpc.php {
    deny all;
    return 403;
}

# WordPressのフロントエンドは外部公開しない
location / {
    deny all;
    return 403;
}
```

WordPressはDockerのバックエンドネットワーク内に閉じ込め、Nginxのリバースプロキシ越しにしかアクセスできない構成にすることで、WordPress自体への直接攻撃を防ぐ。

### 2. 人的コストの維持（これが本当に重要）

ヘッドレス化の話をすると「管理画面もNext.jsで作り直したほうが良くないですか」という意見をもらうことがある。技術的には正しい。しかしこれは**現場のスタッフが管理画面を使えなくなる**ことを意味する。

WordPressのシェアは世界40%超だ。多くの非エンジニアが「記事を書く・画像を追加する」ことを当たり前にできる。この**習熟度という人的資産**を捨てることのコストを、技術的な最適化の文脈で評価してはいけない。

```
管理画面: WordPress（現場スタッフが慣れ親しんでいる）
         ↓ REST API
フロント: Next.js（エンジニアが最適化できる）
```

この分離こそが、ヘッドレス化の本質的な価値だ。

### 3. ISRによる更新の即時性

Next.jsのISR（Incremental Static Regeneration）を使うと、WordPressの更新を静的生成に組み込める。

```typescript
// app/casts/[slug]/page.tsx

export const revalidate = 60  // 60秒ごとに再生成

export async function generateStaticParams() {
  const casts = await getCasts()
  return casts.map(cast => ({ slug: cast.slug }))
}
```

WordPressの管理画面でキャスト情報を更新すると、**最大60秒以内にフロントエンドに反映される。**再ビルドは不要だ。更新頻度が高い業務サイトにとって、これは重要な特性になる。

---

## Docker Composeによるポータビリティ

BAN対策として最も重要なのは、**環境をコードとして管理すること**だ。

```yaml
# docker-compose.yml（一部）
services:
  nginx:
    image: nginx:1.25-alpine
    networks: [frontend, backend]

  wordpress:
    image: wordpress:6.5-php8.2-fpm-alpine
    expose: ["9000"]  # 外部に公開しない
    networks: [backend]  # バックエンドのみ

  postgres:
    image: postgres:16-alpine
    expose: ["5432"]  # 外部に公開しない
    networks: [backend]

networks:
  backend:
    internal: true  # 完全に内部のみ
```

`internal: true`にすることで、バックエンドネットワークはDockerホスト外から絶対にアクセスできない。

この構成をGitリポジトリで管理しておくと、新しいVPSに移行するときの手順が以下になる。

```bash
git clone https://github.com/grayzone-eng/infra-iac.git
cd infra-iac
cp .env.example .env  # パスワード等を設定
sudo ./scripts/setup.sh  # Dockerインストール・SSL取得・起動まで自動化
./scripts/restore.sh db latest  # 最新バックアップからDBリストア
```

**目標復旧時間: 30分以内。** BAN発生後、DNSのTTLを事前に300秒以下に設定しておけばDNS切り替えも5分以内に完了する。

---

## まとめ

ヘッドレスWordPressを選ぶ理由を一言でまとめると、「**現場の人的資産を維持しながら、技術的な最適化とセキュリティ強化を実現できる唯一の構成だから**」になる。

パフォーマンス改善のためだけならHeadless CMSは他にも選択肢がある。しかし「現場スタッフが慣れ親しんだ管理画面を捨てない」という制約の下では、WordPressのヘッドレス化が最も合理的な答えだと考えている。

:::details 参考: 使用技術スタック
- VPS: FlokiNET (Iceland)
- OS: Ubuntu 22.04 LTS
- Web server: Nginx 1.25
- CMS: WordPress 6.5 (Headless)
- DB: PostgreSQL 16
- Frontend: Next.js 14 (App Router / ISR)
- CDN: Bunny.net
- Container: Docker Compose
- SSL: Let's Encrypt / Certbot
:::
