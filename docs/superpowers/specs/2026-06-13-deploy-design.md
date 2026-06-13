# デプロイパイプライン設計

**日付**: 2026-06-13  
**対象**: Blazor WASM (.NET 10) → Cloudflare Pages + 独自ドメイン

---

## 概要

GitHub の `main` ブランチへの push をトリガーに、GitHub Actions が .NET ビルドを行い、Wrangler CLI で Cloudflare Pages へ自動デプロイする。カスタムドメイン `96xtools.dev` で公開。

---

## アーキテクチャ

```
git push origin main
        ↓
GitHub Actions (ubuntu-latest)
  1. actions/setup-dotnet@v4 (.NET 10)
  2. dotnet publish -c Release -o publish
  3. cloudflare/wrangler-action@v3
        ↓
Cloudflare Pages (project: utilities)
        ↓
https://96xtools.dev
```

---

## 変更対象ファイル

| ファイル | 変更内容 |
|---|---|
| `.github/workflows/deploy.yml` | GitHub Pages 用ワークフローを Cloudflare Pages 用に全面書き換え |

---

## 新しい deploy.yml

```yaml
name: Deploy to Cloudflare Pages

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - name: Publish
        run: dotnet publish -c Release -o publish

      - name: Deploy to Cloudflare Pages
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          command: pages deploy publish/wwwroot --project-name=utilities
```

### GitHub Pages 版からの主な変更点
- `Fix base href` ステップ削除（ルートドメイン運用のため `/` のまま）
- `Add nojekyll` ステップ削除（GitHub Pages 専用、不要）
- `peaceiris/actions-gh-pages` → `cloudflare/wrangler-action@v3`
- `permissions: contents: write` 削除（不要）

---

## 事前手作業（完了済み）

| ステップ | 内容 | 状態 |
|---|---|---|
| 1 | Cloudflare API トークン作成（Edit Cloudflare Workers テンプレート） | ✅ 完了 |
| 2 | Cloudflare アカウント ID 確認 | ✅ 完了 |
| 3 | GitHub Secrets に `CLOUDFLARE_API_TOKEN` / `CLOUDFLARE_ACCOUNT_ID` を登録 | ✅ 完了 |
| 4 | Cloudflare Pages プロジェクト `utilities` 作成、カスタムドメイン `96xtools.dev` 設定 | ✅ 完了 |

---

## デプロイフロー（完成後）

1. `git push origin main` で自動起動
2. GitHub Actions が .NET 10 環境でビルド（約 2〜3 分）
3. Wrangler が `publish/wwwroot` の静的ファイルを Cloudflare Pages へアップロード
4. `https://96xtools.dev` に自動反映（Cloudflare CDN 経由）

---

## 考慮事項

- **base href**: カスタムドメインのルートで運用するため `<base href="/">` のまま変更不要
- **キャッシュ**: Cloudflare Pages は自動でグローバル CDN キャッシュ。デプロイ時に古いキャッシュは自動パージされる
- **プレビューデプロイ**: Wrangler 経由のデプロイでも Cloudflare Pages のダッシュボードでデプロイ履歴・ロールバックが可能
