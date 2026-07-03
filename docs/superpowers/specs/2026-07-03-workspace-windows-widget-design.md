# workspace_windows ウィジェット設計書

日付: 2026-07-03
ステータス: 承認済み

## 概要

komorebi-bar に、現在フォーカスされているワークスペース内の**全ウインドウ**を
Windows のタスクバーのように一覧表示する新ウィジェット `workspace_windows` を追加する。
フォーカス中のウインドウはハイライトされ、任意のウインドウをクリックすると
そのウインドウにフォーカスが移る。

既存の `focused_window`（= `focused_container`）ウィジェットはフォーカス中の
コンテナ（スタック）内のウインドウしか表示しない。本ウィジェットはこれを置き換える
ものではなく、独立したウィジェットとして共存する。

## 要件（確認済み）

- 表示形式は既存の `DisplayFormat` で設定可能（icon / text / icon_and_text /
  icon_and_text_on_selected / text_and_icon_on_selected）
- スタックは全ウインドウをフラットに展開（1ウインドウ = 1ボタン）
- フォーカス済みウインドウの再クリックは何もしない（最小化トグルはスコープ外）
- 対象は現在のワークスペースのみ（モノクル・タイル・フロート・最大化を含む）

## アーキテクチャ

クリック→フォーカスは HWND 指定の新 socket メッセージで実現する（案A-1）。
バーの表示順とコア内部のインデックスの対応付け問題を避け、
モノクル/タイル/スタック/フロート/最大化の全ケースを1メッセージで処理する。

```
[komorebi-bar]                          [komorebi core]
state通知 ──────────────────────────▶ (既存の購読機構)
ウインドウ一覧を描画 (hwnd保持)
クリック ── FocusWindowByHwnd(hwnd) ──▶ location_from_hwnd で検索
                                        → モニタ/WS/コンテナ/ウインドウをフォーカス
```

## 変更内容

### コア側（komorebi / komorebic）

1. **`komorebi/src/core/mod.rs`**
   `SocketMessage::FocusWindowByHwnd(isize)` を追加（`EagerFocus(String)` の隣）。

2. **`komorebi/src/workspace.rs`**
   `location_from_hwnd(&self, hwnd: isize) -> Option<WorkspaceWindowLocation>` を追加。
   `location_from_exe` のミラー実装。タイル/モノクルは既存の
   `Container::idx_for_window(hwnd)`、最大化/フロートは `window.hwnd == hwnd` 比較。

3. **`komorebi/src/process_command.rs`**
   `EagerFocus` ハンドラの「モニタ/WS切替（遅延ロード）→ location に応じた
   フォーカス → 必要ならWSロード」部分を共通ヘルパー関数に抽出し、
   `EagerFocus`（exe検索）と `FocusWindowByHwnd`（hwnd検索）の両方から呼ぶ。

4. **`komorebic/src/main.rs`**
   サブコマンド `focus-window-by-hwnd <HWND>` を追加（`eager-focus` と同パターン）。
   デバッグ・スクリプト用途。

### バー側（komorebi-bar）

5. **設定追加** — `KomorebiConfig` に:

   ```rust
   pub workspace_windows: Option<KomorebiWorkspaceWindowsConfig>,
   ```

   ```rust
   pub struct KomorebiWorkspaceWindowsConfig {
       pub enable: bool,
       pub display: Option<DisplayFormat>,
   }
   ```

   設定例:

   ```json
   "komorebi": {
     "workspace_windows": { "enable": true, "display": "IconAndTextOnSelected" }
   }
   ```

6. **`WindowInfo` に `hwnd: isize` を追加**。`From<&Window>` で保存するだけ
   （アイコン取得で既に hwnd を使用している）。

7. **データ収集の統一（リファクタ）**
   `MonitorInfo::update` は常に `from_all_containers` で全コンテナを収集する
   （`show_all_icons` によるデータ分岐を廃止）。後方互換のため、workspaces
   ウィジェットの非 `all_*` 系レンダラは `try_from` でのレンダラ選択時に
   「フォーカス中コンテナのみ描画する」バリアントを選ぶ。`has_icons` は
   全コンテナ基準の値になるため、フォーカス中コンテナのみ描画するバリアントでは
   `ws.focused_container()` のアイコン有無を直接判定する（全体フラグに依存しない）。

8. **最大化ウインドウの収集漏れ修正**
   `from_all_containers` に `ws.maximized_window` を含める（フロートと同様に
   `from_window` で追加）。表示順: モノクル → タイル → フロート → 最大化。
   既存 `all_icons` 表示で最大化ウインドウのアイコンが消えるバグの修正を兼ねる。

9. **新ウィジェット `WorkspaceWindowsBar`**
   `FocusedContainerBar` と同じ「config 時にレンダラ関数を選択する」構造。
   - データソース: `monitor_info.focused_workspace()` の全コンテナ → 全ウインドウをフラット展開
   - ハイライト: `container.is_focused && window_idx == container.focused_window_idx`
     のウインドウのみ `SelectableFrame::new(true)` + 選択色
   - クリック: `send_with_mouse_follow_off(monitor_info, FocusWindowByHwnd(hwnd))`。
     フォーカス済みウインドウのクリックはノーオプ
   - 描画部品はアイコン描画・タイトル切り詰め（`MAX_LABEL_WIDTH`）を
     `FocusedContainerBar` の `show_icon` / `show_title` 相当で流用
   - `render()` の呼び出し順: workspaces → **workspace_windows** → layer → …

## エラー処理

- HWND のウインドウが存在しない（クリック直前に閉じた等）: コア側で検索が
  ヒットせずノーオプ。バーは次の state 通知で自動的に表示から消える
- socket 送信失敗: 既存 `send_messages` がログ出力して継続（踏襲）

## テスト

- **ユニットテスト**: `location_from_hwnd` の4ケース（タイル/スタック/モノクル/
  フロート）。`container.rs` の既存テストと同じ `Window::from(fake_hwnd)`
  パターンで検証。`cargo test -p komorebi`
- **手動検証**:
  1. ビルドした komorebi + komorebi-bar を起動
  2. タイル2枚＋スタック1組＋フロート1枚を配置し、全ウインドウの表示と
     フォーカスハイライトを確認
  3. 非フォーカスウインドウのクリック → フォーカス移動＋ハイライト追従
  4. スタック内の背面ウインドウのクリック → スタック内切替
  5. `komorebic focus-window-by-hwnd <HWND>` の単体動作
  6. 回帰: `workspaces.display` の `"all_icons"` / `"icon"` の見た目が不変であること

## ビルド・スキーマ

- `cargo build --release --no-default-features`（`--locked` は付けない。
  現時点の Cargo.lock は再解決差分があるため）
- スキーマ再生成: `schema.bar.json`（新設定が入る）、`schema.json` /
  `schema.asc.json`（差分なし見込みだが念のため）

## スコープ外（YAGNI）

- フォーカス済みウインドウ再クリックでの最小化トグル
- 他ワークスペースのウインドウ表示
- ドラッグ並び替え、ホバープレビュー
- upstream への PR（ローカル改造。将来検討する場合は CONTRIBUTING.md の
  CLA 要件を確認すること）
