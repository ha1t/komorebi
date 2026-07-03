# workspace_windows ウィジェット実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** komorebi-bar に現在のワークスペースの全ウインドウをタスクバー風に表示し、クリックでフォーカスできる `workspace_windows` ウィジェットを追加する。

**Architecture:** クリック→フォーカスは新 socket メッセージ `FocusWindowByHwnd(isize)` で実現。コア側は `location_from_exe` のミラーである `location_from_hwnd` で検索し、`EagerFocus` と共通のフォーカス処理を使う。バー側はデータ収集を常に全コンテナ方式に統一（レンダラ側でフィルタして後方互換維持）し、`FocusedContainerBar` の描画部品を再利用した新ウィジェットを追加する。

**Tech Stack:** Rust (stable-msvc), egui/eframe (bar), clap (CLI), serde/schemars (設定スキーマ)

**設計書:** `docs/superpowers/specs/2026-07-03-workspace-windows-widget-design.md`

**ビルド上の注意:**
- すべて `--no-default-features` なしの通常 `cargo build`/`cargo test` はスキーマ生成 feature を含むのでOK。リリース確認は `--no-default-features` 付き
- `--locked` は付けない（現時点の Cargo.lock は再解決差分があり失敗する）
- 作業ブランチ: `feature/workspace-windows-widget`（作成済み）

---

### Task 1: コア — `location_from_hwnd` の追加（TDD）

**Files:**
- Modify: `komorebi/src/workspace.rs:157-163`（`WorkspaceWindowLocation` に PartialEq 追加）
- Modify: `komorebi/src/workspace.rs:913` 付近（`location_from_exe` の直後に新関数）
- Test: `komorebi/src/workspace.rs` 末尾の `#[cfg(test)] mod tests`（1885行〜）

- [ ] **Step 1: `WorkspaceWindowLocation` に PartialEq を追加**

`komorebi/src/workspace.rs:157` の derive を変更（テストで `assert_eq!` を使うため）:

```rust
// 変更前
#[derive(Debug)]
pub enum WorkspaceWindowLocation {

// 変更後
#[derive(Debug, PartialEq)]
pub enum WorkspaceWindowLocation {
```

- [ ] **Step 2: 失敗するテストを書く**

`komorebi/src/workspace.rs` の `mod tests` 内（既存テストの後ろ）に追加。
`floating_windows_mut()` は `&mut VecDeque<Window>` を返すので `push_back` を使う:

```rust
#[test]
fn test_location_from_hwnd() {
    let mut ws = Workspace::default();

    // タイルコンテナ0: 2枚のスタック
    let mut stack = Container::default();
    stack.windows_mut().push_back(Window::from(10));
    stack.windows_mut().push_back(Window::from(11));
    ws.add_container_to_back(stack);

    // タイルコンテナ1: 単独ウインドウ
    let mut single = Container::default();
    single.windows_mut().push_back(Window::from(20));
    ws.add_container_to_back(single);

    // モノクルコンテナ
    let mut monocle = Container::default();
    monocle.windows_mut().push_back(Window::from(30));
    ws.monocle_container = Some(monocle);

    // フローティングウインドウ
    ws.floating_windows_mut().push_back(Window::from(40));

    // 最大化ウインドウ
    ws.maximized_window = Some(Window::from(50));

    assert_eq!(
        ws.location_from_hwnd(11),
        Some(WorkspaceWindowLocation::Container(0, 1))
    );
    assert_eq!(
        ws.location_from_hwnd(20),
        Some(WorkspaceWindowLocation::Container(1, 0))
    );
    assert_eq!(
        ws.location_from_hwnd(30),
        Some(WorkspaceWindowLocation::Monocle(0))
    );
    assert_eq!(
        ws.location_from_hwnd(40),
        Some(WorkspaceWindowLocation::Floating(0))
    );
    assert_eq!(
        ws.location_from_hwnd(50),
        Some(WorkspaceWindowLocation::Maximized)
    );
    assert_eq!(ws.location_from_hwnd(999), None);
}
```

- [ ] **Step 3: テストが失敗する（コンパイルエラーになる）ことを確認**

Run: `cargo test -p komorebi location_from_hwnd`
Expected: FAIL — `no method named location_from_hwnd found`

- [ ] **Step 4: `location_from_hwnd` を実装**

`komorebi/src/workspace.rs` の `location_from_exe`（881-913行）の直後に追加。
exe版と同じ探索順（タイル → 最大化 → モノクル → フロート）:

```rust
pub fn location_from_hwnd(&self, hwnd: isize) -> Option<WorkspaceWindowLocation> {
    for (container_idx, container) in self.containers().iter().enumerate() {
        if let Some(window_idx) = container.idx_for_window(hwnd) {
            return Some(WorkspaceWindowLocation::Container(
                container_idx,
                window_idx,
            ));
        }
    }

    if let Some(window) = self.maximized_window
        && window.hwnd == hwnd
    {
        return Some(WorkspaceWindowLocation::Maximized);
    }

    if let Some(container) = &self.monocle_container
        && let Some(window_idx) = container.idx_for_window(hwnd)
    {
        return Some(WorkspaceWindowLocation::Monocle(window_idx));
    }

    for (window_idx, window) in self.floating_windows().iter().enumerate() {
        if window.hwnd == hwnd {
            return Some(WorkspaceWindowLocation::Floating(window_idx));
        }
    }

    None
}
```

- [ ] **Step 5: テストが通ることを確認**

Run: `cargo test -p komorebi location_from_hwnd`
Expected: PASS (1 passed)

- [ ] **Step 6: コミット**

```powershell
git add komorebi/src/workspace.rs
git commit -m "feat(wm): HWNDからウインドウ位置を検索する location_from_hwnd を追加"
```

---

### Task 2: コア — `FocusWindowByHwnd` メッセージとハンドラ

**Files:**
- Modify: `komorebi/src/core/mod.rs:104`（`EagerFocus(String)` の直後に variant 追加）
- Modify: `komorebi/src/process_command.rs:223-296`（EagerFocus ハンドラの共通化＋新ハンドラ）
- Modify: `komorebi/src/process_command.rs`（import 追加）

- [ ] **Step 1: SocketMessage に variant を追加**

`komorebi/src/core/mod.rs:104` の `EagerFocus(String),` の直後に追加:

```rust
    EagerFocus(String),
    FocusWindowByHwnd(isize),
```

- [ ] **Step 2: `Workspace` の import を追加**

`komorebi/src/process_command.rs:94` 付近（`use crate::workspace::WorkspaceLayer;` の隣）:

```rust
use crate::workspace::Workspace;
use crate::workspace::WorkspaceLayer;
use crate::workspace::WorkspaceWindowLocation;
```

（`Workspace` が100行目以降で既に import されている場合は追加不要。重複するとコンパイルエラーになる）

- [ ] **Step 3: 共通ヘルパー関数を抽出**

`komorebi/src/process_command.rs` の `process_socket_message` を含む `impl WindowManager`
ブロック内（`process_socket_message` の直前）に、現在の `EagerFocus` アーム（223-296行）の
本体を一般化したメソッドを追加する。検索条件をクロージャで受け取る以外、既存コードと同一:

```rust
/// Searches all monitors/workspaces for a window matching `locate`, then focuses
/// the monitor, workspace, container and window it belongs to.
fn focus_window_by_location<F>(&mut self, locate: F) -> eyre::Result<()>
where
    F: Fn(&Workspace) -> Option<WorkspaceWindowLocation>,
{
    let focused_monitor_idx = self.focused_monitor_idx();

    let mut window_location = None;
    let mut monitor_to_focus = None;
    let mut needs_workspace_loading = false;

    'search: for (monitor_idx, monitor) in self.monitors_mut().iter_mut().enumerate() {
        for (workspace_idx, workspace) in monitor.workspaces().iter().enumerate() {
            if let Some(location) = locate(workspace) {
                window_location = Some(location);

                if monitor_idx != focused_monitor_idx {
                    monitor_to_focus = Some(monitor_idx);
                }

                // Focus workspace if it is not already the focused one, without
                // loading it so that we don't give focus to the wrong window, we will
                // load it later after focusing the wanted window
                let focused_ws_idx = monitor.focused_workspace_idx();
                if focused_ws_idx != workspace_idx {
                    monitor.last_focused_workspace = Option::from(focused_ws_idx);
                    monitor.focus_workspace(workspace_idx)?;
                    needs_workspace_loading = true;
                }

                break 'search;
            }
        }
    }

    if let Some(monitor_idx) = monitor_to_focus {
        self.focus_monitor(monitor_idx)?;
    }

    if let Some(location) = window_location {
        match location {
            WorkspaceWindowLocation::Monocle(window_idx) => {
                self.focus_container_window(window_idx)?;
            }
            WorkspaceWindowLocation::Maximized => {
                if let Some(window) = &mut self.focused_workspace_mut()?.maximized_window {
                    window.focus(self.mouse_follows_focus)?;
                }
            }
            WorkspaceWindowLocation::Container(container_idx, window_idx) => {
                let focused_container_idx = self.focused_container_idx()?;
                if container_idx != focused_container_idx {
                    self.focused_workspace_mut()?.focus_container(container_idx);
                }

                self.focus_container_window(window_idx)?;
            }
            WorkspaceWindowLocation::Floating(window_idx) => {
                if let Some(window) = self
                    .focused_workspace_mut()?
                    .floating_windows_mut()
                    .get_mut(window_idx)
                {
                    window.focus(self.mouse_follows_focus)?;
                }
            }
        }

        if needs_workspace_loading {
            let mouse_follows_focus = self.mouse_follows_focus;
            if let Some(monitor) = self.focused_monitor_mut() {
                monitor.load_focused_workspace(mouse_follows_focus)?;
            }
        }
    }

    Ok(())
}
```

注意: 既存の EagerFocus アーム内のコードと1文字単位で一致させること（`Option::from` などの
イディオムを維持）。唯一の違いは `workspace.location_from_exe(exe)` が `locate(workspace)` に
なる点。

- [ ] **Step 4: EagerFocus アームを置き換え、FocusWindowByHwnd アームを追加**

`komorebi/src/process_command.rs:223-296` の `SocketMessage::EagerFocus(ref exe) => { ... }`
アーム全体を以下の2アームに置き換える:

```rust
SocketMessage::EagerFocus(ref exe) => {
    self.focus_window_by_location(|workspace| workspace.location_from_exe(exe))?;
}
SocketMessage::FocusWindowByHwnd(hwnd) => {
    self.focus_window_by_location(|workspace| workspace.location_from_hwnd(hwnd))?;
}
```

- [ ] **Step 5: ビルドとテストの確認**

Run: `cargo build -p komorebi && cargo test -p komorebi`
Expected: ビルド成功、既存テスト＋Task 1 のテストすべて PASS

- [ ] **Step 6: コミット**

```powershell
git add komorebi/src/core/mod.rs komorebi/src/process_command.rs
git commit -m "feat(wm): FocusWindowByHwnd メッセージを追加し EagerFocus とフォーカス処理を共通化"
```

---

### Task 3: komorebic — CLI サブコマンド `focus-window-by-hwnd`

**Files:**
- Modify: `komorebic/src/main.rs:999-1003`（`EagerFocus` 構造体の直後に引数構造体）
- Modify: `komorebic/src/main.rs:1140-1142`（`SubCommand` enum の `EagerFocus` の直後）
- Modify: `komorebic/src/main.rs:2098-2100`（ハンドラの `EagerFocus` の直後）

- [ ] **Step 1: 引数構造体を追加**

`komorebic/src/main.rs` の `struct EagerFocus`（999-1003行）の直後に追加:

```rust
#[derive(Parser)]
struct FocusWindowByHwnd {
    /// Window handle (HWND) of the window to focus
    hwnd: isize,
}
```

- [ ] **Step 2: SubCommand enum に variant を追加**

`komorebic/src/main.rs:1142` の `EagerFocus(EagerFocus),` の直後に追加:

```rust
    /// Focus the first managed window matching the given exe
    #[clap(arg_required_else_help = true)]
    EagerFocus(EagerFocus),
    /// Focus the managed window with the given HWND
    #[clap(arg_required_else_help = true)]
    FocusWindowByHwnd(FocusWindowByHwnd),
```

- [ ] **Step 3: ハンドラを追加**

`komorebic/src/main.rs:2098-2100` の `SubCommand::EagerFocus(args) => { ... }` の直後に追加:

```rust
SubCommand::FocusWindowByHwnd(args) => {
    send_message(&SocketMessage::FocusWindowByHwnd(args.hwnd))?;
}
```

- [ ] **Step 4: ビルドとヘルプ表示の確認**

Run: `cargo build -p komorebic && cargo run -p komorebic -- focus-window-by-hwnd --help`
Expected: ビルド成功、ヘルプに `Focus the managed window with the given HWND` と `<HWND>` 引数が表示される

- [ ] **Step 5: コミット**

```powershell
git add komorebic/src/main.rs
git commit -m "feat(cli): focus-window-by-hwnd サブコマンドを追加"
```

---

### Task 4: バー — `WindowInfo.hwnd` の追加と最大化ウインドウの収集漏れ修正

**Files:**
- Modify: `komorebi-bar/src/widgets/komorebi.rs:1009-1025`（`WindowInfo`）
- Modify: `komorebi-bar/src/widgets/komorebi.rs:944-964`（`from_all_containers`）

- [ ] **Step 1: `WindowInfo` に hwnd フィールドを追加**

`komorebi-bar/src/widgets/komorebi.rs:1009-1025` を変更:

```rust
/// Stores basic information about a single window in a container.
/// Contains the window's title, its icon if available, and its hwnd.
#[derive(Clone, Debug)]
pub struct WindowInfo {
    pub title: Option<String>,
    pub icon: Option<ImageIcon>,
    pub hwnd: isize,
}

impl From<&Window> for WindowInfo {
    fn from(value: &Window) -> Self {
        Self {
            title: value.title().ok(),
            icon: ImageIcon::try_load(value.hwnd, || {
                windows_icons::get_icon_by_hwnd(value.hwnd)
                    .or_else(|| windows_icons_fallback::get_icon_by_process_id(value.process_id()))
            }),
            hwnd: value.hwnd,
        }
    }
}
```

- [ ] **Step 2: `from_all_containers` に最大化ウインドウを含める**

`komorebi-bar/src/widgets/komorebi.rs:944-964` の `from_all_containers` を変更。
フォーカス優先度のガードに最大化ウインドウを加え、末尾に最大化を追加する
（表示順: モノクル → タイル → フロート → 最大化）:

```rust
pub fn from_all_containers(ws: &Workspace) -> Vec<Self> {
    let has_focused_float = ws.floating_windows().iter().any(|w| w.is_focused())
        || ws.maximized_window.as_ref().is_some_and(|w| w.is_focused());

    // Monocle container first if present
    let monocle = ws
        .monocle_container
        .as_ref()
        .map(|c| Self::from_container(c, !has_focused_float));

    // All tiled containers, focus only if there's no monocle/focused float
    let has_focused_monocle_or_float = has_focused_float || monocle.is_some();
    let tiled = ws.containers().iter().enumerate().map(|(i, c)| {
        let is_focused = !has_focused_monocle_or_float && i == ws.focused_container_idx();
        Self::from_container(c, is_focused)
    });

    // All floating windows
    let floats = ws.floating_windows().iter().map(Self::from_window);
    // The maximized window, if any
    let maximized = ws.maximized_window.as_ref().map(Self::from_window);
    // All windows
    monocle
        .into_iter()
        .chain(tiled)
        .chain(floats)
        .chain(maximized)
        .collect()
}
```

- [ ] **Step 3: ビルド確認**

Run: `cargo build -p komorebi-bar`
Expected: ビルド成功（warning なし）

- [ ] **Step 4: コミット**

```powershell
git add komorebi-bar/src/widgets/komorebi.rs
git commit -m "feat(bar): WindowInfo に hwnd を追加し最大化ウインドウの収集漏れを修正"
```

---

### Task 5: バー — データ収集の全コンテナ方式への統一

`show_all_icons` フラグによるデータ分岐を廃止し、常に全コンテナを収集する。
後方互換のため workspaces ウィジェットのアイコン描画はレンダラ選択時にフィルタする。

**Files:**
- Modify: `komorebi-bar/src/widgets/komorebi.rs:388-496`（`WorkspacesBar` レンダラ分割）
- Modify: `komorebi-bar/src/widgets/komorebi.rs:761-897`（`MonitorInfo` から show_all_icons 除去）
- Modify: `komorebi-bar/src/widgets/komorebi.rs:966-982`（`from_focused_container` 削除）
- Modify: `komorebi-bar/src/render.rs:68,107-126,155`（`RenderConfig.show_all_icons` 除去）
- Modify: `komorebi-bar/src/config.rs:142-149`（`show_all_icons_on_komorebi_workspace` 削除）
- Modify: `komorebi-bar/src/bar.rs:985-990`（`update` 呼び出しの引数変更）

- [ ] **Step 1: `WorkspacesBar` のアイコン描画を all / focused の2バリアントに分割**

`komorebi-bar/src/widgets/komorebi.rs:443-464` の `show_icons` を `show_icons_all` に
リネームし、`show_icons_focused` を追加:

```rust
/// Draws all window icons for the workspace, using larger size for the focused container.
/// Returns response if icons exist, or None.
fn show_icons_all(&self, ctx: &Context, ui: &mut Ui, ws: &WorkspaceInfo) -> Option<Response> {
    ws.has_icons.then(|| {
        Frame::NONE
            .inner_margin(Margin::same(ui.style().spacing.button_padding.y as i8))
            .show(ui, |ui| {
                for container in &ws.containers {
                    for icon in container.windows.iter().filter_map(|win| win.icon.as_ref()) {
                        ui.add(
                            Image::from(&icon.texture(ctx))
                                .maintain_aspect_ratio(true)
                                .fit_to_exact_size(if container.is_focused {
                                    self.icon_size
                                } else {
                                    self.text_size
                                }),
                        );
                    }
                }
            })
            .response
    })
}

/// Draws the focused container's window icons only.
/// Returns response if icons exist, or None.
fn show_icons_focused(&self, ctx: &Context, ui: &mut Ui, ws: &WorkspaceInfo) -> Option<Response> {
    let container = ws.focused_container()?;
    if !container.windows.iter().any(|win| win.icon.is_some()) {
        return None;
    }
    Some(
        Frame::NONE
            .inner_margin(Margin::same(ui.style().spacing.button_padding.y as i8))
            .show(ui, |ui| {
                for icon in container.windows.iter().filter_map(|win| win.icon.as_ref()) {
                    ui.add(
                        Image::from(&icon.texture(ctx))
                            .maintain_aspect_ratio(true)
                            .fit_to_exact_size(self.icon_size),
                    );
                }
            })
            .response,
    )
}
```

- [ ] **Step 2: `WorkspacesBar::try_from` のレンダラ選択を分割**

`komorebi-bar/src/widgets/komorebi.rs:395-432` のレンダラ選択 match を変更。
`All*` 系と `Existing(*)` 系で呼ぶバリアントを分ける（外観の後方互換を維持）:

```rust
let renderer: fn(&Self, &Context, &mut Ui, &WorkspaceInfo) =
    match value.display.unwrap_or(DisplayFormat::Text.into()) {
        // 1a: Show all windows' icons if any, fallback if none | Only hover workspace name
        AllIcons => |bar, ctx, ui, ws| {
            bar.show_icons_all(ctx, ui, ws)
                .unwrap_or_else(|| bar.show_fallback_icon(ctx, ui, ws))
                .on_hover_text(&ws.name);
        },
        // 1b: Show focused container's icons if any, fallback if none | Only hover workspace name
        Existing(DisplayFormat::Icon) => |bar, ctx, ui, ws| {
            bar.show_icons_focused(ctx, ui, ws)
                .unwrap_or_else(|| bar.show_fallback_icon(ctx, ui, ws))
                .on_hover_text(&ws.name);
        },
        // 2a: Show all icons, with no fallback | Label workspace name (no hover)
        AllIconsAndText => |bar, ctx, ui, ws| {
            bar.show_icons_all(ctx, ui, ws);
            Self::show_label(ctx, ui, ws);
        },
        // 2b: Show focused container's icons, with no fallback | Label workspace name (no hover)
        Existing(DisplayFormat::IconAndText) => |bar, ctx, ui, ws| {
            bar.show_icons_focused(ctx, ui, ws);
            Self::show_label(ctx, ui, ws);
        },
        // 3a: All icons, fallback if no icons and not selected | Label if selected else hover
        AllIconsAndTextOnSelected => |bar, ctx, ui, ws| {
            if bar.show_icons_all(ctx, ui, ws).is_none() && !ws.is_selected {
                bar.show_fallback_icon(ctx, ui, ws);
            }
            if ws.is_selected {
                Self::show_label(ctx, ui, ws);
            } else {
                ui.response().on_hover_text(&ws.name);
            }
        },
        // 3b: Focused icons, fallback if no icons and not selected | Label if selected else hover
        Existing(DisplayFormat::IconAndTextOnSelected) => |bar, ctx, ui, ws| {
            if bar.show_icons_focused(ctx, ui, ws).is_none() && !ws.is_selected {
                bar.show_fallback_icon(ctx, ui, ws);
            }
            if ws.is_selected {
                Self::show_label(ctx, ui, ws);
            } else {
                ui.response().on_hover_text(&ws.name);
            }
        },
        // 4: Show focused icons if selected (no fallback) | Label workspace name
        Existing(DisplayFormat::TextAndIconOnSelected) => |bar, ctx, ui, ws| {
            if ws.is_selected {
                bar.show_icons_focused(ctx, ui, ws);
            }
            Self::show_label(ctx, ui, ws);
        },
        // 5: Never show icon (no icons at all) | Label workspace name always
        Existing(DisplayFormat::Text) => |_, ctx, ui, ws| {
            Self::show_label(ctx, ui, ws);
        },
    };
```

- [ ] **Step 3: `MonitorInfo` から show_all_icons を除去し、常に全コンテナを収集**

`komorebi-bar/src/widgets/komorebi.rs` で以下を変更:

(a) `MonitorInfo` 構造体（762-772行）から `pub show_all_icons: bool,` を削除し、
`Default` 実装（774-788行）から `show_all_icons: false,` を削除。

(b) `update`（815-842行）を変更:

```rust
/// Updates monitor state from the given State, setting all fields based on the selected
/// monitor and its workspaces
pub fn update(&mut self, monitor_index: Option<usize>, state: State) {
    self.monitor_usr_idx_map = state.monitor_usr_idx_map;

    match monitor_index {
        Some(idx) if idx < state.monitors.elements().len() => self.monitor_index = idx,
        // The bar's monitor is diconnected, so the bar is disabled no need to check anything
        // any further otherwise we'll get `OutOfBounds` panics.
        _ => return,
    };
    self.mouse_follows_focus = state.mouse_follows_focus;

    let monitor = &state.monitors.elements()[self.monitor_index];
    self.work_area_offset = monitor.work_area_offset;
    self.focused_workspace_idx = Some(monitor.focused_workspace_idx());

    // Layout
    let focused_ws = &monitor.workspaces()[monitor.focused_workspace_idx()];
    self.layout = Self::resolve_layout(focused_ws, state.is_paused);

    self.workspaces.clear();
    self.workspaces.extend(Self::workspaces(
        self.hide_empty_workspaces,
        self.focused_workspace_idx,
        monitor.workspaces().iter().enumerate(),
    ));
}
```

(c) `workspaces` ビルダー（844-880行）を変更:

```rust
/// Builds an iterator of WorkspaceInfo for the monitor.
fn workspaces<'a, I>(
    hide_empty_ws: bool,
    focused_ws_idx: Option<usize>,
    iter: I,
) -> impl Iterator<Item = WorkspaceInfo> + 'a
where
    I: Iterator<Item = (usize, &'a Workspace)> + 'a,
{
    iter.map(move |(index, ws)| {
        let containers = ContainerInfo::from_all_containers(ws);
        WorkspaceInfo {
            name: ws
                .name
                .to_owned()
                .unwrap_or_else(|| format!("{}", index + 1)),
            focused_container_idx: containers.iter().position(|c| c.is_focused),
            has_icons: containers
                .iter()
                .any(|container| container.windows.iter().any(|window| window.icon.is_some())),
            containers,
            layer: ws.layer,
            should_show: !hide_empty_ws || focused_ws_idx == Some(index) || !ws.is_empty(),
            is_selected: focused_ws_idx == Some(index),
        }
    })
}
```

(d) 使われなくなった `ContainerInfo::from_focused_container`（966-982行）を削除。

- [ ] **Step 4: `RenderConfig` / `KomobarConfig` / `bar.rs` から show_all_icons を除去**

(a) `komorebi-bar/src/render.rs:68` の `pub show_all_icons: bool,` を削除。
(b) `komorebi-bar/src/render.rs:107-114` の `let show_all_icons = ...` の計算と、
    116-131行の構造体初期化内の `show_all_icons,` を削除。
(c) `komorebi-bar/src/render.rs:155` の `new()` 内の `show_all_icons: false,` を削除。
(d) `komorebi-bar/src/config.rs:142-149` の `show_all_icons_on_komorebi_workspace` 関数を削除。
(e) `komorebi-bar/src/bar.rs:985-990` の呼び出しを変更:

```rust
if let Some(monitor_info) = &self.monitor_info {
    monitor_info
        .borrow_mut()
        .update(self.monitor_index, notification.state);
```

(f) 上記削除で unused import warning が出た場合（config.rs の
`WorkspacesDisplayFormat` など）は該当 import を削除する。

- [ ] **Step 5: ビルド確認**

Run: `cargo build -p komorebi-bar`
Expected: ビルド成功、warning なし（dead_code / unused_imports が出たら該当箇所を削除）

- [ ] **Step 6: コミット**

```powershell
git add komorebi-bar/src
git commit -m "refactor(bar): ワークスペースのコンテナ収集を常に全コンテナ方式に統一"
```

---

### Task 6: バー — `workspace_windows` ウィジェットの追加

**Files:**
- Modify: `komorebi-bar/src/widgets/komorebi.rs:53-67`（`KomorebiConfig` に設定追加）
- Modify: `komorebi-bar/src/widgets/komorebi.rs:105-116` 付近（設定構造体の追加）
- Modify: `komorebi-bar/src/widgets/komorebi.rs:140-190`（`Komorebi` 構造体・From・render）
- Modify: `komorebi-bar/src/widgets/komorebi.rs:508-559`（`FocusedContainerBar::from_format` 抽出）

- [ ] **Step 1: 設定構造体を追加**

`komorebi-bar/src/widgets/komorebi.rs` の `KomorebiFocusedContainerConfig`（105-116行）の
直後に追加:

```rust
#[derive(Copy, Clone, Debug, Serialize, Deserialize)]
#[cfg_attr(feature = "schemars", derive(schemars::JsonSchema))]
/// Komorebi widget workspace windows configuration
pub struct KomorebiWorkspaceWindowsConfig {
    /// Enable the Komorebi Workspace Windows widget
    pub enable: bool,
    /// Display format of the workspace windows (default: IconAndTextOnSelected)
    pub display: Option<DisplayFormat>,
}
```

`KomorebiConfig`（53-67行）の `focused_container` フィールドの直後に追加:

```rust
    /// Configure the Workspace Windows widget (taskbar-like list of all
    /// windows on the focused workspace)
    pub workspace_windows: Option<KomorebiWorkspaceWindowsConfig>,
```

- [ ] **Step 2: `FocusedContainerBar::from_format` を抽出**

`komorebi-bar/src/widgets/komorebi.rs:508-559` の `FocusedContainerBar::try_from` を、
フォーマット決定と構築の2段に分割する（レンダラ選択 match の中身は無変更）:

```rust
impl FocusedContainerBar {
    /// Creates a `FocusedContainerBar` from the given configuration.
    ///
    /// Selects a render strategy based on the display format or legacy icon setting.
    /// Returns `None` if the widget is disabled.
    fn try_from(value: KomorebiFocusedContainerConfig) -> Option<Self> {
        if !value.enable {
            return None;
        }

        // Handle legacy setting - convert show_icon to display format
        #[allow(deprecated)]
        let format = value
            .display
            .unwrap_or(if value.show_icon.unwrap_or(false) {
                IconAndText
            } else {
                Text
            });

        Some(Self::from_format(format))
    }

    /// Creates a bar rendering a list of windows with the given display format.
    /// Used by both the Focused Container and the Workspace Windows widgets.
    fn from_format(format: DisplayFormat) -> Self {
        // Select renderer strategy based on display format for better performance
        let renderer: fn(&FocusedContainerBar, &Context, &mut Ui, &WindowInfo, Color32, bool) =
            match format {
                Icon => |_self, ctx, ui, info, _color, _focused| {
                    FocusedContainerBar::show_icon::<true>(_self, ctx, ui, info);
                },
                Text => |_self, _ctx, ui, info, color, _focused| {
                    FocusedContainerBar::show_title(_self, ui, info, color);
                },
                IconAndText => |_self, ctx, ui, info, color, _focused| {
                    FocusedContainerBar::show_icon::<false>(_self, ctx, ui, info);
                    FocusedContainerBar::show_title(_self, ui, info, color);
                },
                IconAndTextOnSelected => |_self, ctx, ui, info, color, focused| {
                    FocusedContainerBar::show_icon::<false>(_self, ctx, ui, info);
                    if focused {
                        FocusedContainerBar::show_title(_self, ui, info, color);
                    }
                },
                TextAndIconOnSelected => |_self, ctx, ui, info, color, focused| {
                    if focused {
                        FocusedContainerBar::show_icon::<false>(_self, ctx, ui, info);
                    }
                    FocusedContainerBar::show_title(_self, ui, info, color);
                },
            };

        FocusedContainerBar {
            renderer,
            icon_size: Vec2::splat(12.5 * 1.4),
        }
    }

    // show_icon / show_title は無変更
```

- [ ] **Step 3: `Komorebi` 構造体と `From` 実装にフィールドを追加**

`komorebi-bar/src/widgets/komorebi.rs:170-179` の `Komorebi` 構造体の
`focused_container` の直後に追加:

```rust
    pub workspace_windows: Option<FocusedContainerBar>,
```

`From<&KomorebiConfig>`（140-168行）の `Self { ... }` 内、`focused_container` の直後に追加:

```rust
            workspace_windows: cfg.workspace_windows.and_then(|c| {
                c.enable.then(|| {
                    FocusedContainerBar::from_format(c.display.unwrap_or(IconAndTextOnSelected))
                })
            }),
```

- [ ] **Step 4: レンダリング関数を追加して配線**

`impl BarWidget for Komorebi`（181-190行）の `render` を変更（workspaces の直後に挿入）:

```rust
impl BarWidget for Komorebi {
    fn render(&mut self, ctx: &Context, ui: &mut Ui, config: &mut RenderConfig) {
        self.render_workspaces(ctx, ui, config);
        self.render_workspace_windows(ctx, ui, config);
        self.render_workspace_layer(ctx, ui, config);
        self.render_layout(ctx, ui, config);
        self.render_config_switcher(ui, config);
        self.render_locked_container(ctx, ui, config);
        self.render_focused_container(ctx, ui, config);
    }
}
```

`impl Komorebi` 内（`render_focused_container` の直前）に追加:

```rust
/// Renders a taskbar-like list of all windows on the focused workspace.
/// The focused window is highlighted; clicking any other window focuses it.
fn render_workspace_windows(&mut self, ctx: &Context, ui: &mut Ui, config: &mut RenderConfig) {
    let Some(bar) = &self.workspace_windows else {
        return;
    };
    let monitor_info = &*self.monitor_info.borrow();
    let Some(workspace) = monitor_info.focused_workspace() else {
        return;
    };
    config.apply_on_widget(false, ui, |ui| {
        for container in &workspace.containers {
            for (idx, window) in container.windows.iter().enumerate() {
                let selected = container.is_focused && idx == container.focused_window_idx;
                let text_color = if selected {
                    ctx.style().visuals.selection.stroke.color
                } else {
                    ui.style().visuals.text_color()
                };

                let response = SelectableFrame::new(selected).show(ui, |ui| {
                    (bar.renderer)(bar, ctx, ui, window, text_color, selected)
                });

                if response.clicked() && !selected {
                    let _ = Self::send_with_mouse_follow_off(
                        monitor_info,
                        FocusWindowByHwnd(window.hwnd),
                    );
                }
            }
        }
    });
}
```

注: `FocusWindowByHwnd` は既存の `use komorebi_client::SocketMessage::*;`（35行）経由で
解決される。

- [ ] **Step 5: ビルド確認**

Run: `cargo build -p komorebi-bar`
Expected: ビルド成功、warning なし

- [ ] **Step 6: コミット**

```powershell
git add komorebi-bar/src/widgets/komorebi.rs
git commit -m "feat(bar): ワークスペースの全ウインドウを表示する workspace_windows ウィジェットを追加"
```

---

### Task 7: スキーマ再生成と全体ビルド

**Files:**
- Modify: `schema.bar.json`（`workspace_windows` が入る）
- Modify: `schema.json` / `schema.asc.json`（差分なしの見込み）

- [ ] **Step 1: 全ワークスペースのビルドとテスト**

Run: `cargo build --release --no-default-features -p komorebic -p komorebic-no-console -p komorebi -p komorebi-bar -p komorebi-gui -p komorebi-shortcuts && cargo test -p komorebi`
Expected: ビルド成功、全テスト PASS

- [ ] **Step 2: スキーマ再生成**

Run（pwsh、リポジトリルートで。デフォルト features でビルドされる）:

```powershell
cargo run --package komorebic -- static-config-schema > schema.json
cargo run --package komorebic -- application-specific-configuration-schema > schema.asc.json
cargo run --package komorebi-bar -- --schema > schema.bar.json
```

Expected: `git diff schema.bar.json` に `workspace_windows` の定義が追加される。
`schema.json` / `schema.asc.json` は差分なし（差分が出た場合は内容を確認し、
自分の変更に起因しないフォーマット差分のみなら `git checkout` で戻す）。

- [ ] **Step 3: コミット**

```powershell
git add schema.bar.json
git commit -m "chore(schema): workspace_windows 追加に伴い schema.bar.json を再生成"
```

---

### Task 8: 手動検証（ユーザーと一緒に実施）

**Files:** なし（検証のみ）

- [ ] **Step 1: 検証用のバー設定を準備**

ユーザーの `komorebi.bar.json` の `komorebi` ウィジェット設定に追加（バックアップを取ってから）:

```json
"workspace_windows": { "enable": true, "display": "icon_and_text_on_selected" }
```

- [ ] **Step 2: ビルドした komorebi / komorebi-bar で起動**

実行中の komorebi がある場合は停止してから、`target\release\` のバイナリで起動する。
（ユーザーの環境に合わせて実施。`komorebic stop --bar` → ビルド版起動、など）

- [ ] **Step 3: 表示の確認**

- タイル2枚＋スタック1組＋フロート1枚を配置し、バーに全ウインドウが並ぶこと
- フォーカス中のウインドウだけハイライト＋タイトル表示されること
- 最大化したウインドウもリストに表示されること

- [ ] **Step 4: クリックフォーカスの確認**

- 非フォーカスのタイルウインドウをクリック → フォーカスが移り、ハイライトが追従する
- スタック内の背面ウインドウをクリック → スタック内でウインドウが切り替わる
- フローティングウインドウをクリック → フォーカスされる
- `komorebic focus-window-by-hwnd <HWND>` を直接実行 → 同様に動作する
  （HWND は `komorebic state` の出力から取得できる）

- [ ] **Step 5: 回帰確認**

- `workspaces.display: "all_icons"` の表示が従来どおりであること
- `workspaces.display: "icon"`（フォーカス中コンテナのアイコンのみ）が従来どおりであること
- 既存の `focused_window`（`focused_container`）ウィジェットの表示・クリックが従来どおりであること
