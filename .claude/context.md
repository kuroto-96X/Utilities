# Utilities - Claude Code コンテキスト

## プロジェクト概要
Blazor WASM（.NET 10）で作られた単機能ツール集サイト。
日本語・英語ユーザー向けに様々な便利ツールを提供する。

## リポジトリ・公開情報
- リポジトリ: https://github.com/kuroto-96X/Utilities
- 公開URL: https://96xtools.dev
- ホスティング: Cloudflare Pages（プロジェクト名: utilities）
- デプロイ: masterブランチへのpushで自動デプロイ（GitHub Actions）

## 技術スタック
- Blazor WASM (.NET 10)
- C#
- GitHub Actions（自動ビルド・デプロイ）
- Cloudflare Pages（ホスティング・CDN・Git連携）
- 独自ドメイン: 96xtools.dev（Cloudflare Pagesのカスタムドメインで設定済み）

## デプロイの仕組み
1. `master` ブランチに push すると GitHub Actions が起動
2. `dotnet publish -c Release -o publish -p:BlazorEnableCompression=false` でビルド
3. SPA用リダイレクトルール（`/* /index.html 200`）を `publish/wwwroot/_redirects` に生成
4. `peaceiris/actions-gh-pages` が `publish/wwwroot/` の中身を `deploy` ブランチに push
5. Cloudflare Pages が `deploy` ブランチを検知して自動デプロイ
6. 公開URL: https://96xtools.dev に反映される

※ Cloudflare Pages は `deploy` ブランチをビルドなしで直接配信する設定（ビルドは GitHub Actions 側で完結）

## プロジェクト構成
```
Utilities/
├── Pages/
│   ├── Home.razor           ← トップページ
│   └── （ツールを追加していく）
├── Layout/
│   ├── MainLayout.razor     ← 共通レイアウト
│   └── NavMenu.razor        ← 共通ナビゲーション
├── wwwroot/
│   ├── index.html
│   └── css/
├── .github/
│   └── workflows/
│       └── deploy.yml       ← GitHub Actionsデプロイ設定
└── .claude/
    └── context.md           ← このファイル
```

## 新しいツールの追加ルール
- `Pages/` に `.razor` ファイルを1つ追加する
- ファイル先頭に `@page "/ツール名"` でルートを設定する
- `Layout/NavMenu.razor` にナビゲーションリンクを追加する
- 1ファイルで完結させる（外部依存は最小限に）

## コーディング規約
- 言語: C#（Blazor）
- コメント: 日本語でOK
- コンポーネント名: PascalCase
- ルートパス: kebab-case（例: `/json-formatter`）

## タスク完了条件
以下をすべて満たしてからコミットすること：
1. `dotnet build` がエラーなく通る
2. `dotnet run` でローカル動作確認済み
3. `git add . && git commit -m "メッセージ" && git push origin master` まで完了
4. GitHub Actions が緑になり https://96xtools.dev に反映されることを確認するよう作業者に伝える
