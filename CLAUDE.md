# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

このリポジトリは単一の自己完結型HTMLファイル [my-todo.html](my-todo.html) です。ビルド不要・依存関係なし・サーバー不要の、日本語UIの個人用TODO/タスクボードアプリです。マークアップ・CSS・素のJSがすべてこの1ファイルに収まっています。`package.json`、バンドラー、テストスイート、リンターの設定はありません。

## 実行・開発方法

ビルドや開発サーバーは存在しません。[my-todo.html](my-todo.html) をブラウザで直接開き（ダブルクリック、`start my-todo.html`、またはブラウザタブへのドラッグ）、このファイルを直接編集します。変更を確認するにはブラウザをリロードしてください。

すべてが1ファイルの素のHTML/CSS/JSなので、編集前にGrepで `my-todo.html` 内を検索して該当する `<style>` ルールやJS関数を特定してください。分割されたモジュールはありません。

## アーキテクチャ

### データモデル

アプリはタスクオブジェクトの単一のフラット配列（`tasks`）を管理します。各タスクは `id, category, sub, content, period, work, status, due, note, prio, week, done` を持ちます。

- **category** — タスクが属する「セクション」（例：取引先名やプロジェクト名）。セクションの表示順は別途 `catOrder` で管理されます。
- **sub / content / period** — この3つが揃って「グループキー」（`groupKey()`）を構成します。同一セクション内で連続し、この3つが一致するタスクはボードのテーブル上で1つの視覚的グループとしてまとめて表示され、`sub`/`content`/`period` はグループ先頭行にのみ表示されます（2行目以降はCSS上ホバー/フォーカス時のみ表示、`.cont-cell` 参照）。
- **work** — タスクの実際の作業内容テキスト。保存時に自動で先頭へ `・` が付与されます。
- **prio** — `""` / `🔴` / `🟡` / `🟢` のいずれか。ボードと今週TODOビューで共有されます。
- **week** — タスクを今週TODOビューに含めるかどうか。`todoOrder` はこのビュー専用の独立した表示順を保持します（このビューでのドラッグ並び替えは `tasks` 側の順序に影響しません）。
- **status** — `STATUSES`（未着手/作業中/待ち/完了/未定）のいずれか。色は `STATUS_COLORS` で定義されています。

### 2つのビュー、1つのデータセット

`currentView` により以下を切り替えます。

- **board**（`renderBoard()` / `id="boardView"`） — 全タスクをセクションごとのテーブルで表示。`contenteditable` セルによるインライン編集、行およびセクション単位のドラッグ&ドロップ並び替え、セクションごとの列ソート（`sortStates`）、行/セクションの折り畳み、複数選択（ドラッグハンドルをShift/Ctrl+クリック）による一括移動、行間/セクション間ホバーでの挿入ゾーンを備えます。
- **todo**（`renderTodo()` / `id="todoView"`） — `week: true` のタスクを `prio` でグループ化したカードリスト表示。独自のドラッグ&ドロップ（`todoOrder`）とインライン編集を持ちます。

両ビューは同じ `tasks` 配列を読み書きするため、片方での編集は再描画後にもう片方にも反映されます。`switchView()` が表示/非表示の切り替えと、共有の追加ボタン群（`#actionTools`）を現在表示中のビューのフィルタ行へ移動する処理を担います。

### 永続化

- **自動保存**: `save()` は変更のたびに `{tasks, catOrder, todoOrder, collapsedCats, collapsedPrios, exportName}` を `localStorage`（キー: `STORAGE_KEY` = `um_todo_app_v1`）へシリアライズします。`load()` は起動時にこれを復元し、何も保存されていない場合はサンプルデータ `SEED` にフォールバックします。
- **バックアップ/復元**: `backupData()` は同じ状態をJSONファイルとしてダウンロードします（`BACKUP_FORMAT: "um_todo_app_backup"`）。`restoreFromFile()` はそのファイル（または生のlocalStorage形式のJSON）を読み込み、確認後に全状態を置き換えます。ブラウザ・PC間でのデータ移行手段はこれのみで、同期機能はありません。
- **Markdown書き出し**: `exportMarkdown()` は現在表示中のビュー（ボードまたは今週TODO）を `buildBoardMd()` / `buildTodoMd()` 経由で `.md` テーブルとして書き出します。ファイル名は `<月曜日の日付>_<種別>_<exportName>.md` です。

### 描画方式

フレームワークは使用していません。すべての変更操作は `save()` の後に `renderBoard()`/`renderTodo()` をフル実行し、対象コンテナの `innerHTML` をメモリ上の状態から再構築してイベントリスナーを全て再バインドします（`bindBoardEvents()` / `bindTodoSelectionAndDnD()`、およびドラッグ&ドロップ用の `bindDragAndDrop()` / `bindSectionDnD()`）。機能追加時も、ピンポイントなDOM操作ではなくこの「状態変更 → save() → 再描画() → 再バインド」というサイクルに従ってください。

### 補助ロジックの補足

- `parseDueDays()` / `priorityFromDays()` は `7/30`、`9月末`、`未定` のような緩い日本語の日付表現を解析して残り日数を算出します。`refreshTodoFromBoard()` はこれを使い、期限が6日以内のボードタスクから今週TODOを自動生成します。
- `getGroupIds()` / `groupKey()` はボードの行グループ表示を実現し、ドラッグ&ドロップでグループ全体（またはグループ内の1行のみ）をまとめて移動できるようにしています。
- セクションIDおよびタスクIDは再利用されません。`_id` は単調増加するカウンタで、起動時に保存データから再計算されます。
