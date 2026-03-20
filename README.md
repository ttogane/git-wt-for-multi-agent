# gh-wt

`gh-wt` は `git worktree` と `tmux` を使った並列開発用の zsh ユーティリティです。

## 前提

- zsh
- git
- tmux
- 任意: fzf（`gh-wt list` の選択UI）
- 任意: cursor / code（`gh-wt open` で利用）

## セットアップ

`~/.zshrc` で `~/.worktree_utils` を読み込む:

```zsh
source "$HOME/.worktree_utils"
```

反映:

```zsh
source ~/.zshrc
```

## コマンド

### Switch / Checkout（tmux attach/new）

```bash
gh-wt switch <branch>
gh-wt switch -c <branch>
gh-wt checkout <branch>
gh-wt checkout -b <branch>
gh-wt switch --open <branch>
gh-wt checkout --open <branch>
```

- 既存 worktree があれば再利用
- なければ作成
- `tmux` セッションへ attach/new（`wt_<repo>_<branch-slug>`）

### Open（Editorのみ）

```bash
gh-wt open <branch>
```

- 対象 worktree を editor で新規ウィンドウ起動
- profile / user-data-dir は `~/.worktree_utils` の設定に従う

### Branch 操作

```bash
gh-wt delete <branch>
gh-wt delete
gh-wt rename <old> <new>
gh-wt rename <new>
```

- `delete`: dirty worktree は削除失敗（引数なしは `fzf` 選択）
- `rename`: branch 名と worktree dir 名を同期 rename
- `rename <new>` は current branch を old とみなす

### List

```bash
gh-wt list
gh-wt list --plain
```

- `gh-wt list`: `fzf` で branch/worktree を選択し、Enter で `open`
- `gh-wt list --plain`: 一覧をテキスト表示

## 推奨フロー

1. `gh-wt switch <branch>` で tmux セッション作業に入る
2. GUI が必要なときだけ `gh-wt open <branch>`
3. 不要 branch は `gh-wt delete <branch>`（または `gh-wt delete`）
