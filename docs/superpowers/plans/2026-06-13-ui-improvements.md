# UI Improvements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Improve BpmTapper UX (slider label, range indicator, reset button, ripple effect) and clean up site-wide UI (nav, colors, title, top-row removal).

**Architecture:** All changes are isolated UI edits across 6 existing files. No new files created. BpmTapper.razor and its CSS are fully rewritten; NavMenu and MainLayout files receive targeted edits.

**Tech Stack:** Blazor WASM (.NET 10), C#, Bootstrap 5, scoped CSS (`.razor.css`)

---

## File Map

| Action | File | Changes |
|---|---|---|
| Rewrite | `Pages/BpmTapper.razor` | Slider label, range divider, reset button move, ripple |
| Rewrite | `Pages/BpmTapper.razor.css` | New styles for all above + remove `:active` |
| Rewrite | `Layout/NavMenu.razor` | Remove Counter/Weather, change title |
| Edit | `Layout/NavMenu.razor.css` | Lighten `.top-row` background |
| Edit | `Layout/MainLayout.razor` | Remove `top-row` div |
| Rewrite | `Layout/MainLayout.razor.css` | Remove top-row styles, fix article padding, update sidebar gradient |

---

### Task 1: Rewrite BpmTapper.razor

**Files:**
- Rewrite: `Pages/BpmTapper.razor`

**Context:** The current file has: slider with "平均範囲" label + "%" span + Reset button all in one row at top; plain tap button; history table without range indicator. The new version restructures this significantly.

- [ ] **Step 1: Replace the entire file content**

Write `Pages/BpmTapper.razor` with this exact content:

```razor
@page "/bpm-tapper"

<PageTitle>Tap BPM</PageTitle>

<div class="tap-bpm-page">
    <!-- AdSense: Tap BPM - ここに広告コードを挿入 -->

    <div class="controls-section">
        <div class="bpm-display-area text-center">
            <div class="bpm-value">@(mainBpm.HasValue ? ((int)Math.Round(mainBpm.Value)).ToString() : "---")</div>
            <div class="bpm-label">BPM</div>
        </div>
        <div class="slider-area px-3 pb-1">
            <input type="range" class="form-range"
                   min="0" max="100" step="1"
                   @bind="sliderValue" @bind:event="oninput" />
            <div class="avg-label text-center text-muted small">@avgTapLabel</div>
        </div>
    </div>

    <div class="tap-section">
        <button class="tap-button @(isPressed ? "pressed" : "")"
                @onclick="OnTap"
                @onpointerdown="() => isPressed = true"
                @onpointerup="() => isPressed = false"
                @onpointerleave="() => isPressed = false">
            TAP
            <span class="ripple @(isRippling ? "active" : "")"></span>
        </button>
    </div>

    <div class="reset-section">
        <button class="reset-button" @onclick="OnReset">Reset</button>
    </div>

    <div class="history-section">
        @if (tapHistory.Count == 0)
        {
            <p class="text-center text-muted py-3">Tap the button to start</p>
        }
        else
        {
            <div class="table-responsive">
                <table class="table table-sm table-striped mb-0">
                    <thead class="table-dark">
                        <tr>
                            <th>#</th>
                            <th>Time</th>
                            <th>Interval (ms)</th>
                            <th>Instant BPM</th>
                        </tr>
                    </thead>
                    <tbody>
                        @{
                            int intervalCount = tapHistory.Count - 1;
                            int n = intervalCount > 0 ? Math.Max(1, (int)Math.Floor(intervalCount * sliderValue / 100.0)) : 0;
                            bool showDivider = intervalCount > 0 && n < intervalCount;
                            int dividerBeforeIndex = tapHistory.Count - n - 1;
                        }
                        @for (int i = tapHistory.Count - 1; i >= 0; i--)
                        {
                            if (showDivider && i == dividerBeforeIndex)
                            {
                                <tr class="range-divider"><td colspan="4"></td></tr>
                            }
                            var tap = tapHistory[i];
                            bool outOfRange = showDivider && i <= dividerBeforeIndex;
                            <tr class="@(outOfRange ? "out-of-range" : "")">
                                <td>@tap.Index</td>
                                <td>@tap.TimeDisplay</td>
                                <td>@(tap.IntervalMs.HasValue ? ((int)tap.IntervalMs.Value).ToString() : "-")</td>
                                <td>@(tap.InstantBpm.HasValue ? ((int)Math.Round(tap.InstantBpm.Value)).ToString() : "-")</td>
                            </tr>
                        }
                    </tbody>
                </table>
            </div>
        }
    </div>
</div>

@code {
    private record TapRecord(int Index, DateTime Timestamp, double? IntervalMs, double? InstantBpm)
    {
        public string TimeDisplay => Timestamp.ToString("HH:mm:ss.fff");
    }

    private List<TapRecord> tapHistory = new();
    private int sliderValue = 100;
    private bool isRippling = false;
    private bool isPressed = false;

    private string avgTapLabel
    {
        get
        {
            int intervalCount = tapHistory.Count - 1;
            if (intervalCount <= 0) return "（タップすると BPM を計測します）";
            int n = Math.Max(1, (int)Math.Floor(intervalCount * sliderValue / 100.0));
            return n == 1 ? "直近 1 タップ（瞬間 BPM）" : $"直近 {n} タップを平均";
        }
    }

    private double? mainBpm
    {
        get
        {
            if (tapHistory.Count < 2) return null;

            var intervals = tapHistory
                .Skip(1)
                .Select(t => t.IntervalMs!.Value)
                .ToList();

            int n = Math.Max(1, (int)Math.Floor(intervals.Count * sliderValue / 100.0));
            var recent = intervals.TakeLast(n).ToList();
            return 60000.0 / recent.Average();
        }
    }

    private async Task OnTap()
    {
        var now = DateTime.Now;
        double? interval = null;
        double? instantBpm = null;

        if (tapHistory.Count > 0)
        {
            interval = (now - tapHistory[^1].Timestamp).TotalMilliseconds;
            instantBpm = 60000.0 / interval.Value;
        }

        tapHistory.Add(new TapRecord(tapHistory.Count + 1, now, interval, instantBpm));

        isRippling = true;
        StateHasChanged();
        await Task.Delay(600);
        isRippling = false;
    }

    private void OnReset() => tapHistory.Clear();
}
```

- [ ] **Step 2: Build to verify**

```bash
dotnet build
```

Expected: `Build succeeded.` with 0 errors.

- [ ] **Step 3: Commit**

```bash
git add Pages/BpmTapper.razor
git commit -m "feat: improve BpmTapper UX - slider label, range indicator, reset button, ripple"
```

---

### Task 2: Rewrite BpmTapper.razor.css

**Files:**
- Rewrite: `Pages/BpmTapper.razor.css`

**Context:** Removes `.tap-button:active` (broken on iOS Safari), adds `.tap-button.pressed` (Blazor-managed), adds ripple keyframes, new reset button and range divider styles.

Note: `@keyframes` in `.razor.css` files is valid — Blazor's CSS isolation compiler correctly leaves `@keyframes` as global CSS.

- [ ] **Step 1: Replace the entire file content**

Write `Pages/BpmTapper.razor.css` with this exact content:

```css
.tap-bpm-page {
    display: flex;
    flex-direction: column;
}

.controls-section {
    border-bottom: 1px solid var(--bs-border-color, #dee2e6);
    padding-top: 1rem;
}

.bpm-display-area {
    padding-bottom: 0.5rem;
}

.bpm-value {
    font-size: clamp(4rem, 20vw, 7rem);
    font-weight: 700;
    line-height: 1;
    font-variant-numeric: tabular-nums;
    letter-spacing: -2px;
}

.bpm-label {
    font-size: 1.25rem;
    color: var(--bs-secondary, #6c757d);
    font-weight: 500;
    margin-top: 0.25rem;
}

.avg-label {
    padding-bottom: 0.5rem;
    min-height: 1.5rem;
}

.tap-section {
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 2rem 1rem 1rem;
}

.tap-button {
    width: min(70vw, 260px);
    height: min(70vw, 260px);
    border-radius: 50%;
    background-color: #0d6efd;
    color: #fff;
    font-size: clamp(1.5rem, 6vw, 2.5rem);
    font-weight: 700;
    border: none;
    cursor: pointer;
    touch-action: manipulation;
    user-select: none;
    -webkit-user-select: none;
    box-shadow: 0 4px 20px rgba(13, 110, 253, 0.35);
    transition: transform 0.06s ease, box-shadow 0.06s ease;
    position: relative;
    overflow: hidden;
}

/* iOS Safari 対応: :active の代わりに Blazor Pointer Events で管理 */
.tap-button.pressed {
    transform: scale(0.94);
    box-shadow: 0 2px 8px rgba(13, 110, 253, 0.25);
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

.reset-section {
    display: flex;
    justify-content: center;
    padding: 0.5rem 1rem 1rem;
}

.reset-button {
    width: min(70vw, 260px);
    height: 3rem;
    border-radius: 9999px;
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

.history-section {
    max-height: 40vh;
    overflow-y: auto;
}

.range-divider td {
    padding: 0;
    height: 2px;
    background-color: #0d6efd;
    opacity: 0.5;
}

.out-of-range {
    opacity: 0.35;
}
```

- [ ] **Step 2: Build to verify**

```bash
dotnet build
```

Expected: `Build succeeded.` with 0 errors.

- [ ] **Step 3: Commit**

```bash
git add Pages/BpmTapper.razor.css
git commit -m "feat: rewrite BpmTapper CSS - ripple, pressed state, reset button, range indicator"
```

---

### Task 3: Update NavMenu

**Files:**
- Rewrite: `Layout/NavMenu.razor`
- Edit: `Layout/NavMenu.razor.css`

**Context:** Remove Counter and Weather nav links, change brand title to "96X's Utilities", lighten the sidebar top-row background color to match the new sky-blue theme.

- [ ] **Step 1: Replace NavMenu.razor entirely**

Write `Layout/NavMenu.razor` with this exact content:

```razor
<div class="top-row ps-3 navbar navbar-dark">
    <div class="container-fluid">
        <a class="navbar-brand" href="">96X's Utilities</a>
        <button title="Navigation menu" class="navbar-toggler" @onclick="ToggleNavMenu">
            <span class="navbar-toggler-icon"></span>
        </button>
    </div>
</div>

<div class="@NavMenuCssClass nav-scrollable" @onclick="ToggleNavMenu">
    <nav class="nav flex-column">
        <div class="nav-item px-3">
            <NavLink class="nav-link" href="" Match="NavLinkMatch.All">
                <span class="bi bi-house-door-fill-nav-menu" aria-hidden="true"></span> Home
            </NavLink>
        </div>
        <div class="nav-item px-3">
            <NavLink class="nav-link" href="bpm-tapper">
                <span class="bi bi-music-note-nav-menu" aria-hidden="true"></span> Tap BPM
            </NavLink>
        </div>
    </nav>
</div>

@code {
    private bool collapseNavMenu = true;

    private string? NavMenuCssClass => collapseNavMenu ? "collapse" : null;

    private void ToggleNavMenu()
    {
        collapseNavMenu = !collapseNavMenu;
    }
}
```

- [ ] **Step 2: Update top-row background in NavMenu.razor.css**

In `Layout/NavMenu.razor.css`, change line 7 from:

```css
    background-color: rgba(0,0,0,0.4);
```

to:

```css
    background-color: rgba(0,0,0,0.15);
```

The `.top-row` block should then read:

```css
.top-row {
    min-height: 3.5rem;
    background-color: rgba(0,0,0,0.15);
}
```

- [ ] **Step 3: Build to verify**

```bash
dotnet build
```

Expected: `Build succeeded.` with 0 errors.

- [ ] **Step 4: Commit**

```bash
git add Layout/NavMenu.razor Layout/NavMenu.razor.css
git commit -m "feat: remove Counter/Weather nav, rename title, lighten nav header"
```

---

### Task 4: Update MainLayout — Remove top-row, Update Sidebar Color

**Files:**
- Edit: `Layout/MainLayout.razor`
- Rewrite: `Layout/MainLayout.razor.css`

**Context:** The page-level "About" bar (`<div class="top-row px-4">`) is being removed. Its CSS styles must also be cleaned up. The `.sidebar` gradient is updated to sky blue. The selector `.top-row, article { padding-left... }` must be simplified to just `article { ... }` to preserve desktop article padding.

Current `Layout/MainLayout.razor`:
```razor
@inherits LayoutComponentBase
<div class="page">
    <div class="sidebar">
        <NavMenu />
    </div>

    <main>
        <div class="top-row px-4">
            <a href="https://learn.microsoft.com/aspnet/core/" target="_blank">About</a>
        </div>

        <article class="content px-4">
            @Body
        </article>

        <footer class="site-footer px-4 py-4">
            ...
        </footer>
    </main>
</div>
```

- [ ] **Step 1: Remove the top-row div from MainLayout.razor**

Delete the following 3 lines from `Layout/MainLayout.razor`:

```razor
        <div class="top-row px-4">
            <a href="https://learn.microsoft.com/aspnet/core/" target="_blank">About</a>
        </div>
```

After the edit, `<main>` should open directly into `<article class="content px-4">` with no top-row div between them. The full file after edit:

```razor
@inherits LayoutComponentBase
<div class="page">
    <div class="sidebar">
        <NavMenu />
    </div>

    <main>
        <article class="content px-4">
            @Body
        </article>

        <footer class="site-footer px-4 py-4">
            <div class="footer-inner d-flex align-items-center gap-3">
                <img src="icon-192.png" alt="96X" class="footer-avatar" />
                <div>
                    <div class="footer-name">96X</div>
                    <!-- TODO: 一言コメントをここに入れる -->
                    <div class="footer-bio text-muted small">ゲーム開発と楽曲制作の合間に、自分が欲しかった『ちょっとした道具』を作って置いています。</div>
                    <div class="footer-links mt-1">
                        <a href="https://x.com/96X_SBRB" target="_blank" rel="noopener" class="footer-link">
                            &#120143; @@96X_SBRB
                        </a>
                    </div>
                </div>
            </div>
        </footer>
    </main>
</div>
```

- [ ] **Step 2: Rewrite MainLayout.razor.css**

Write `Layout/MainLayout.razor.css` with this exact content (removes all `.top-row` rules, updates sidebar gradient, fixes article padding selector):

```css
.page {
    position: relative;
    display: flex;
    flex-direction: column;
}

main {
    flex: 1;
}

.sidebar {
    background-image: linear-gradient(180deg, #0ea5e9 0%, #0369a1 100%);
}

@media (min-width: 641px) {
    .page {
        flex-direction: row;
    }

    .sidebar {
        width: 250px;
        height: 100vh;
        position: sticky;
        top: 0;
    }

    article {
        padding-left: 2rem !important;
        padding-right: 1.5rem !important;
    }
}

.site-footer {
    border-top: 1px solid #dee2e6;
    margin-top: 2rem;
}

.footer-avatar {
    width: 48px;
    height: 48px;
    border-radius: 50%;
    object-fit: cover;
    flex-shrink: 0;
}

.footer-name {
    font-weight: 600;
    font-size: 1rem;
}

.footer-bio {
    margin-top: 0.1rem;
}

.footer-link {
    color: inherit;
    text-decoration: none;
    font-size: 0.875rem;
}

.footer-link:hover {
    text-decoration: underline;
}
```

- [ ] **Step 3: Build to verify**

```bash
dotnet build
```

Expected: `Build succeeded.` with 0 errors.

- [ ] **Step 4: Commit**

```bash
git add Layout/MainLayout.razor Layout/MainLayout.razor.css
git commit -m "feat: remove About top-row, update sidebar to sky blue"
```

---

### Task 5: Verify icon-192.png and Final Push

**Files:** none (git + verification only)

- [ ] **Step 1: Verify icon-192.png is committed**

```bash
git log --oneline -- wwwroot/icon-192.png
```

Expected: at least one commit line (e.g. `3a1a243 tweak: アイコン変更`). If no output, the icon was never committed — check if the file exists and commit it.

- [ ] **Step 2: Run the app and verify all changes**

```bash
dotnet run
```

Open the URL shown in terminal, verify:
- [ ] Slider shows "（タップすると BPM を計測します）" initially
- [ ] After 3+ taps, slider label shows "直近 N タップを平均"
- [ ] Slider at leftmost: "直近 1 タップ（瞬間 BPM）"
- [ ] Slowing slider to 50%: rows below divider line go dim (opacity 0.35)
- [ ] Blue divider line visible between in-range and out-of-range rows
- [ ] TAP button shrinks on press (works on desktop click-hold); ripple ring expands on release
- [ ] Reset button is a wide oval below the TAP button, above history
- [ ] Counter and Weather gone from nav
- [ ] Title shows "96X's Utilities"
- [ ] Sidebar is sky blue (not dark navy/purple)
- [ ] No "About" bar at top of content area
- [ ] Footer still visible on all pages

Stop with Ctrl+C.

- [ ] **Step 3: Push master**

```bash
git push origin master
```

- [ ] **Step 4: Inform user about icon cache**

Tell the user:
- If the icon still shows old image after deployment, it's a **Cloudflare CDN cache** issue
- Fix: Go to Cloudflare dashboard → Caching → Purge Cache → Custom Purge → enter `https://96xtools.dev/icon-192.png`
- Alternative: hard refresh in browser (Ctrl+Shift+R / Cmd+Shift+R)
