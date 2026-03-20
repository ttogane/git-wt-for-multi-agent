# worktree × VS Code × cmux/tmux による複数 Agent 並列開発環境 要件整理

## 目的

AI コーディング Agent を複数同時に走らせる前提で、以下を満たす軽量な開発環境を整える。

- `git worktree` でブランチごとに作業ディレクトリを分離
- VS Code / Cursor を worktree ごとに別ウィンドウで開く
- Profile / `--user-data-dir` により設定・拡張・環境変数の干渉を減らす
- `cmux` を可視化・通知レイヤーとして使う
- `tmux` をセッション永続化レイヤーとして使う

---

## 背景

現状の不満点:

- iTerm だけでは複数 Agent の並列管理がしづらい
- `git worktree` でディレクトリ分離はできるが、エディタ切り替えが面倒
- 複数 Agent 実行時に拡張機能・設定・環境変数が衝突しやすい
- Dev Container は分離強度は高いが、起動が重く今回は避けたい

---

## 採用方針

### 1. worktree を基本単位にする

1 ブランチ = 1 worktree = 1 エディタウィンドウ = 1 tmux セッション

`git worktree` は同じリポジトリに複数の working tree をぶら下げられる。`add` / `list --porcelain` / `move` / `remove` / `repair` が使える。Git は同じブランチが別 worktree で checkout 済みの場合、`add` を通常は拒否するため、この性質を安全策として利用する。 :contentReference[oaicite:0]{index=0}

### 2. VS Code は worktree ごとに別ウィンドウで開く

VS Code の「マルチウィンドウ」は、単に worktree ごとに別ウィンドウを開く運用を指す。CLI の `-n` / `--new-window` を使う。 :contentReference[oaicite:1]{index=1}

### 3. Profile と `--user-data-dir` を worktree 単位で分離する

VS Code は `--user-data-dir` を分けることで、環境変数・設定・拡張・UI 状態を別管理にできる。Node バージョン差分や PATH 差分がある並列作業では特に有効。 :contentReference[oaicite:2]{index=2}

また Profile はフォルダ/ワークスペースに関連付けでき、フォルダを開くたびにその Profile が有効になる。 :contentReference[oaicite:3]{index=3}

### 4. cmux と tmux は役割分担する

- `cmux`: 複数 Agent の可視化、通知、workspace/pane 管理
- `tmux`: セッションの永続化、切断後の再アタッチ

cmux は macOS 向けの軽量ネイティブ terminal で、複数 AI coding agents 管理向けに設計されている。workspace を CLI / socket API で操作できる。 :contentReference[oaicite:4]{index=4}

一方で cmux は現時点では「layout / metadata の復元のみ」で、live process state は復元しない。つまり app 再起動時に Claude Code / vim / tmux 自体の中身はそのまま復元されない。 :contentReference[oaicite:5]{index=5}

tmux はセッションが persistent で、SSH 切断や detach 後も再アタッチできる。 :contentReference[oaicite:6]{index=6}

したがって、**cmux を UI / 通知、tmux をプロセス永続化**として併用する。

---

## 実現したい要件

### 要件A: ブランチごとの worktree 自動生成

- `git switch <branch>` 相当の独自ラッパーコマンドを作る
- 挙動:
  - 対象ブランチに対応する worktree がある場合:
    - その worktree を使う
    - 対応する VS Code / Cursor ウィンドウを起動する
    - 対応する tmux セッションへ attach または起動する
  - 対象ブランチに対応する worktree がない場合:
    - worktree を新規作成する
    - 必要なら branch も新規作成する
    - エディタ起動 + tmux セッション作成を行う

### 要件B: branch 削除時に worktree も削除

- `git branch -D <branch>` をそのまま使うのではなく、独自ラッパー `wt-delete <branch>` を作る
- 挙動:
  - 対応 worktree がある場合は `git worktree remove`
  - その後 `git branch -D`
  - dirty な worktree は原則エラーにし、`--force` 時のみ削除

`git worktree remove [-f]` は公式に存在する。 :contentReference[oaicite:7]{index=7}

### 要件C: branch rename 時に worktree 名も追従

- `git branch -M old new` の代わりに独自ラッパー `wt-rename old new` を作る
- 挙動:
  - branch rename
  - 対応 worktree が存在すれば `git worktree move <old-path> <new-path>`
  - その後エディタを新 path で開き直す

`git worktree move <worktree> <new-path>` は公式コマンドとして存在する。 :contentReference[oaicite:8]{index=8}

### 要件D: worktree ごとに Profile を割り当てる

- worktree ごとに Profile 名を固定生成する
- 例:
  - `WT-myrepo-feature-login`
  - `WT-myrepo-bugfix-auth`

### 要件E: worktree ごとに `--user-data-dir` を分ける

- 例:
  - `~/.vscode-wt/myrepo/user-data/feature-login`
  - `~/.vscode-wt/myrepo/user-data/bugfix-auth`

これにより以下を worktree ごとに分離する。 :contentReference[oaicite:9]{index=9}

- 環境変数
- 設定
- 拡張機能
- UI 状態

### 要件F: cmux + tmux の連携

- `cmux`:
  - workspace 一覧の可視化
  - 状態表示
  - 通知
- `tmux`:
  - `wt:<repo>:<branch-slug>` の規則でセッション作成
  - 既存があれば attach、なければ new-session

cmux は `list-workspaces` / `new-workspace` などを CLI / socket API 経由で操作できる。 :contentReference[oaicite:10]{index=10}

tmux の passthrough が必要な場面では `allow-passthrough` を有効化する。 :contentReference[oaicite:11]{index=11}

---

## 推奨アーキテクチャ

## ディレクトリ構成

```text
repo-root/
  .git/

../.worktrees/
  myrepo/
    feature-login/
    bugfix-auth/
    chore-docs/

~/.vscode-wt/
  myrepo/
    user-data/
      feature-login/
      bugfix-auth/
      chore-docs/