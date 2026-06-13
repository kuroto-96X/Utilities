# UI 改善設計

**日付**: 2026-06-13  
**対象**: BpmTapper 機能改善 + サイト全体 UI 整理

---

## 概要

BpmTapper ページの UX 改善（スライダー表示・計算範囲の可視化・リセットボタン移動・リップルエフェクト）と、サイト全体のナビゲーション整理・カラーテーマ変更を行う。

---

## スコープ

| 変更種別 | ファイル |
|---|---|
| BpmTapper UI 改善 | `Pages/BpmTapper.razor`（変更） |
| BpmTapper スタイル | `Pages/BpmTapper.razor.css`（変更） |
| ナビ整理・タイトル変更・カラー変更 | `Layout/NavMenu.razor`（変更） |
| ナビ CSS | `Layout/NavMenu.razor.css`（変更） |
| About リンク削除・top-row 削除 | `Layout/MainLayout.razor`（変更） |
| top-row CSS 削除 | `Layout/MainLayout.razor.css`（変更） |

---

## BpmTapper 改善

### 1. スライダー表示変更

**変更前**: `平均範囲` ラベル + パーセント表示  
**変更後**: ラベル削除、代わりに動的テキストのみ表示

| 状態 | 表示テキスト |
|---|---|
| タップなし / 1タップのみ | `（タップすると BPM を計測します）` |
| スライダー最左（n=1） | `直近 1 タップ（瞬間 BPM）` |
| それ以外 | `直近 N タップを平均` |

計算式（既存ロジックを変更しない）:
```csharp
var intervalCount = tapHistory.Count - 1; // インターバル数
int n = Math.Max(1, (int)Math.Floor(intervalCount * sliderValue / 100.0));
```

スライダーの `%` 表示スパンは削除。テキストはスライダーの下に `<div class="avg-label">` として表示。

### 2. 履歴の計算範囲インジケーター

- 履歴テーブルは最新順（上が最新）
- 計算に使われている上位 N 件に含まれる行: 通常表示
- 含まれない行（下部）: `opacity: 0.35` で薄く表示
- **区切り線**: 範囲内・範囲外の境界に `<tr class="range-divider">` を挿入（細い横線）

境界の計算:
```csharp
// tapHistory の何番目以降が "in range" か（元のリスト順）
// in range: tapHistory[Count - n .. Count - 1]
// out of range: tapHistory[0 .. Count - n - 1]
// 全タップが範囲内の場合、区切り線は不要
bool showDivider = tapHistory.Count >= 2 && n < intervalCount;
int dividerBeforeIndex = tapHistory.Count - n - 1; // このインデックスの前（逆順表示での下）に区切り線
```

テーブルの for ループ（i: Count-1 → 0）で、`i == dividerBeforeIndex` のタイミングで区切り行を挿入し、`i <= dividerBeforeIndex` の行に `.out-of-range` クラスを付与。

### 3. リセットボタンの移動とスタイル変更

- **位置**: タップボタンの下、履歴の上（`.tap-section` と `.history-section` の間に `.reset-section` を追加）
- **スタイル**: 横長楕円、幅は `min(70vw, 260px)`（タップボタンと同幅）、高さ `3rem`
- **デザイン**: アウトライン系（背景なし、ボーダーのみ）

```css
.reset-section {
    display: flex;
    justify-content: center;
    padding: 0.5rem 1rem 1rem;
}

.reset-button {
    width: min(70vw, 260px);
    height: 3rem;
    border-radius: 9999px; /* 完全な楕円 */
    border: 2px solid #6c757d;
    background: transparent;
    color: #6c757d;
    font-size: 1rem;
    font-weight: 600;
    cursor: pointer;
    touch-action: manipulation;
    transition: background 0.15s, color 0.15s;
}

.reset-button:active {
    background: #6c757d;
    color: #fff;
}
```

### 4. タップボタン リップルエフェクト

CSS `@keyframes` + Blazor の状態フラグで実装（JS不要）。

```csharp
private bool isRippling = false;

private async void OnTap()
{
    // BPM 計測処理（既存）
    // ...（既存のタップ処理）

    // リップル
    isRippling = true;
    StateHasChanged();
    await Task.Delay(600);
    isRippling = false;
    StateHasChanged();
}
```

テンプレート:
```razor
<button class="tap-button" @onclick="OnTap">
    TAP
    <span class="ripple @(isRippling ? "active" : "")"></span>
</button>
```

CSS:
```css
.tap-button {
    position: relative;
    overflow: hidden;
    /* 既存スタイルはそのまま */
}

.ripple {
    position: absolute;
    border-radius: 50%;
    width: 100%;
    height: 100%;
    top: 0;
    left: 0;
    background: rgba(255, 255, 255, 0.35);
    transform: scale(0);
    pointer-events: none;
}

.ripple.active {
    animation: ripple-expand 0.6s ease-out forwards;
}

@keyframes ripple-expand {
    to {
        transform: scale(2.5);
        opacity: 0;
    }
}
```

---

## サイト全体の整理

### 5. Counter・Weather をナビから削除

`Layout/NavMenu.razor` の Counter と Weather の `<div class="nav-item px-3">` ブロックを削除。  
ページファイル自体（`Pages/Counter.razor`, `Pages/Weather.razor`）は残す（URLで直アクセスは可能）。

### 6. icon-192.png 表示問題の修正

実装時に確認する事項:
- `wwwroot/icon-192.png` が最新ファイルであるか（git status / git diff で確認）
- ブラウザキャッシュ: 実装者はハードリフレッシュ（Ctrl+Shift+R）で確認
- Cloudflare CDN キャッシュ: デプロイ後 Cloudflare ダッシュボードからキャッシュをパージするか、ファイル名にバージョンを付加する方法も検討

### 7. "About" リンクと top-row の削除

`Layout/MainLayout.razor` の以下のブロックを丸ごと削除:
```razor
<div class="top-row px-4">
    <a href="https://learn.microsoft.com/aspnet/core/" target="_blank">About</a>
</div>
```

`Layout/MainLayout.razor.css` の `.top-row` 関連スタイル（`.top-row`、`.top-row ::deep a` 等）も削除。

---

## サイドバー改善

### 8. タイトル変更

`Layout/NavMenu.razor` の:
```razor
<a class="navbar-brand" href="">Utilities</a>
```
↓
```razor
<a class="navbar-brand" href="">96X's Utilities</a>
```

### 9. カラーテーマ変更（ライトブルー系）

`Layout/NavMenu.razor.css` の `.sidebar` のグラデーションを変更:

**変更前**:
```css
background-image: linear-gradient(180deg, rgb(5, 39, 103) 0%, #3a0647 70%);
```

**変更後**（ライトブルー系）:
```css
background-image: linear-gradient(180deg, #0ea5e9 0%, #0369a1 100%);
```

合わせて調整するスタイル:
- `.navbar-toggler` 背景: `rgba(255, 255, 255, 0.15)`（変更なし、元からほぼ白透過なのでOK）
- `.top-row`（NavMenu側）: `rgba(0, 0, 0, 0.2)` → `rgba(0, 0, 0, 0.15)`（背景に合わせて少し薄く）

---

## 完了条件

1. `dotnet build` がエラーなく通る
2. `dotnet run` でローカル確認:
   - スライダーのテキストが動的に変わる
   - 履歴の計算範囲外が薄く表示される
   - リセットボタンが楕円でタップボタン下に表示
   - TAP ボタンのリップルエフェクトが動作
   - Counter・Weather がナビから消えている
   - About リンク・top-row が消えている
   - タイトルが「96X's Utilities」
   - サイドバーがライトブルー
3. master にプッシュ後、GitHub Actions 緑 → https://96xtools.dev で確認
