# Tap BPM + Site Footer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a Tap BPM calculator page at `/bpm-tapper` and a site-wide author footer to the Utilities Blazor WASM app.

**Architecture:** Two self-contained changes on `feature/bpm-tapper` branch. (1) `Pages/BpmTapper.razor` + scoped CSS: tap-driven BPM calculator with slider-controlled average window and history table. (2) Footer block added to `Layout/MainLayout.razor` + its CSS: author info visible on every page. Both merged to master to trigger auto-deploy.

**Tech Stack:** Blazor WASM (.NET 10), C#, Bootstrap 5, scoped CSS (`.razor.css`)

---

## File Map

| Action | File | Purpose |
|---|---|---|
| Create | `Pages/BpmTapper.razor` | Tap BPM page — all UI + BPM logic |
| Create | `Pages/BpmTapper.razor.css` | Scoped styles for BPM page |
| Modify | `Layout/NavMenu.razor` | Add "Tap BPM" nav link |
| Modify | `Layout/NavMenu.razor.css` | Add music note icon SVG |
| Modify | `Layout/MainLayout.razor` | Add site-wide footer |
| Modify | `Layout/MainLayout.razor.css` | Add footer styles |

---

### Task 1: Create Feature Branch

**Files:** none (git only)

- [ ] **Step 1: Create and switch to branch**

```bash
git checkout -b feature/bpm-tapper
```

Expected output: `Switched to a new branch 'feature/bpm-tapper'`

---

### Task 2: Add Tap BPM to Navigation

**Files:**
- Modify: `Layout/NavMenu.razor`
- Modify: `Layout/NavMenu.razor.css`

- [ ] **Step 1: Add nav link to NavMenu.razor**

In `Layout/NavMenu.razor`, add the following block after the last `</div>` of the Weather nav item (the last nav item before `</nav>`):

```razor
        <div class="nav-item px-3">
            <NavLink class="nav-link" href="bpm-tapper">
                <span class="bi bi-music-note-nav-menu" aria-hidden="true"></span> Tap BPM
            </NavLink>
        </div>
```

- [ ] **Step 2: Add music note icon to NavMenu.razor.css**

Append to the end of `Layout/NavMenu.razor.css`:

```css
.bi-music-note-nav-menu {
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='16' height='16' fill='white' viewBox='0 0 16 16'%3E%3Cpath d='M9 13c0 1.105-1.12 2-2.5 2S4 14.105 4 13s1.12-2 2.5-2 2.5.895 2.5 2z'/%3E%3Cpath fill-rule='evenodd' d='M9 3v10H8V3h1z'/%3E%3Cpath d='M8 2.82a1 1 0 0 1 .804-.98l3-.6A1 1 0 0 1 13 2.22V4L8 5V2.82z'/%3E%3C/svg%3E");
}
```

- [ ] **Step 3: Build to verify no errors**

```bash
dotnet build
```

Expected: `Build succeeded.` with 0 errors.

- [ ] **Step 4: Commit**

```bash
git add Layout/NavMenu.razor Layout/NavMenu.razor.css
git commit -m "feat: add Tap BPM nav link"
```

---

### Task 3: Create BpmTapper.razor

**Files:**
- Create: `Pages/BpmTapper.razor`

- [ ] **Step 1: Create the file with complete content**

Create `Pages/BpmTapper.razor` with this exact content:

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
        <div class="slider-area d-flex align-items-center gap-2 px-3 pb-2">
            <span class="text-nowrap small text-muted">平均範囲</span>
            <input type="range" class="form-range flex-grow-1"
                   min="0" max="100" step="1"
                   @bind="sliderValue" @bind:event="oninput" />
            <span class="text-nowrap small">@sliderValue%</span>
            <button class="btn btn-outline-secondary btn-sm" @onclick="OnReset">Reset</button>
        </div>
    </div>

    <div class="tap-section">
        <button class="tap-button" @onclick="OnTap">TAP</button>
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
                        @for (int i = tapHistory.Count - 1; i >= 0; i--)
                        {
                            var tap = tapHistory[i];
                            <tr>
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

    private void OnTap()
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
    }

    private void OnReset() => tapHistory.Clear();
}
```

- [ ] **Step 2: Build to catch any syntax errors**

```bash
dotnet build
```

Expected: `Build succeeded.` with 0 errors.

- [ ] **Step 3: Commit**

```bash
git add Pages/BpmTapper.razor
git commit -m "feat: add Tap BPM page component"
```

---

### Task 4: Style the BPM Page

**Files:**
- Create: `Pages/BpmTapper.razor.css`

- [ ] **Step 1: Create scoped CSS file**

Create `Pages/BpmTapper.razor.css` with this exact content:

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

.tap-section {
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 2rem 1rem;
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
}

.tap-button:active {
    transform: scale(0.94);
    box-shadow: 0 2px 8px rgba(13, 110, 253, 0.25);
}

.history-section {
    max-height: 40vh;
    overflow-y: auto;
}
```

- [ ] **Step 2: Run the app and verify manually**

```bash
dotnet run
```

Open the URL shown in the terminal (usually `http://localhost:5000`), then navigate to `/bpm-tapper`.

Verify each item:
- [ ] BPM shows `---` before any taps
- [ ] "Tap the button to start" placeholder visible
- [ ] Round blue TAP button centered on screen
- [ ] After 1st tap: history row appears with `-` for interval and BPM
- [ ] After 2nd tap: BPM value appears and updates
- [ ] Slider at 0%: BPM shows only the last interval's BPM
- [ ] Slider at 100%: BPM shows full average
- [ ] Reset button clears all history and resets BPM to `---`
- [ ] "Tap BPM" link visible in sidebar nav
- [ ] On mobile (DevTools → device emulation): button is large enough to tap comfortably

Stop the dev server with Ctrl+C.

- [ ] **Step 3: Commit**

```bash
git add Pages/BpmTapper.razor.css
git commit -m "feat: add Tap BPM scoped styles"
```

---

### Task 5: Add Site-Wide Footer

**Files:**
- Modify: `Layout/MainLayout.razor`
- Modify: `Layout/MainLayout.razor.css`

Current `Layout/MainLayout.razor` structure for reference:
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
    </main>
</div>
```

- [ ] **Step 1: Add footer to MainLayout.razor**

Replace the closing `</main>` tag with the footer + closing tag:

```razor
        <footer class="site-footer px-4 py-4">
            <div class="footer-inner d-flex align-items-center gap-3">
                <img src="icon-192.png" alt="96X" class="footer-avatar" />
                <div>
                    <div class="footer-name">96X</div>
                    <!-- TODO: 一言コメントをここに入れる -->
                    <div class="footer-bio text-muted small">一言コメント</div>
                    <div class="footer-links mt-1">
                        <a href="https://x.com/96X_SBRB" target="_blank" rel="noopener" class="footer-link">
                            &#120143; @@96X_SBRB
                        </a>
                    </div>
                </div>
            </div>
        </footer>
    </main>
```

After the edit, the full file should look like:

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
            <div class="footer-inner d-flex align-items-center gap-3">
                <img src="icon-192.png" alt="96X" class="footer-avatar" />
                <div>
                    <div class="footer-name">96X</div>
                    <!-- TODO: 一言コメントをここに入れる -->
                    <div class="footer-bio text-muted small">一言コメント</div>
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

Note: `@@` in Razor outputs a literal `@` character. So `@@96X_SBRB` renders as `@96X_SBRB` in the browser.

- [ ] **Step 2: Add footer styles to MainLayout.razor.css**

Append to the end of `Layout/MainLayout.razor.css`:

```css
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

- [ ] **Step 3: Build and verify**

```bash
dotnet build
```

Expected: `Build succeeded.` with 0 errors.

```bash
dotnet run
```

Verify:
- [ ] Footer appears on Home page (`/`)
- [ ] Footer appears on Counter page (`/counter`)
- [ ] Footer appears on Tap BPM page (`/bpm-tapper`)
- [ ] Avatar image, name "96X", X link `@96X_SBRB` all visible
- [ ] On mobile: footer stacks cleanly

Stop with Ctrl+C.

- [ ] **Step 4: Commit**

```bash
git add Layout/MainLayout.razor Layout/MainLayout.razor.css
git commit -m "feat: add site-wide author footer"
```

---

### Task 6: Push and Handoff

**Files:** none

- [ ] **Step 1: Push feature branch**

```bash
git push -u origin feature/bpm-tapper
```

- [ ] **Step 2: Inform user**

Tell the user:
- Branch `feature/bpm-tapper` is pushed to GitHub
- To deploy: merge `feature/bpm-tapper` → `master` on GitHub
- GitHub Actions will build and push to the `deploy` branch automatically
- After Actions completes (≈2–3 min), verify at `https://96xtools.dev/bpm-tapper`
- **One-line bio placeholder** is still in the footer — edit `Layout/MainLayout.razor` line with `<!-- TODO: 一言コメントをここに入れる -->` once the text is decided
