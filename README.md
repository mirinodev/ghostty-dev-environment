# Ghostty + Herdr ターミナル中心の軽量開発環境

> VSCode の代わりに「ターミナルに寄せる」発想で構築した開発環境。
> 2026-09 に画面分割を zellij から **Herdr** (AI エージェント用マルチプレクサ) へ切り替えた。

## なぜこの構成？

- **メモリ節約**: VSCode（Electron）を丸ごと排除できる
- **起動速度**: ターミナルベースなので一瞬で立ち上がる
- **AI エージェント並列運用**: Herdr が各ペインの Claude Code 等を自動検知し、サイドバーで状態 (作業中 / 入力待ち / 完了) を一覧できる。入力待ちや完了は、サイドバーの印で知らせる
- **セッション保持**: セッションは herdr server が持つので、Ghostty を閉じてもエージェントは走り続ける
- **カスタマイズ性**: 各ツールが独立しているので、好みに合わせて差し替え可能

## ツール構成

| 役割 | ツール | ポイント |
|------|--------|----------|
| ターミナル | [Ghostty](https://ghostty.org/) | 起動が爆速、GPU描画で軽量 |
| マルチプレクサ | [Herdr](https://herdr.dev/) | エージェント自動検知 + 状態通知。Socket API で AI 自身がペインを操作できる |
| ファイラー | [yazi](https://yazi-rs.github.io/) | プレビュー付き、Rust製で爆速 |
| Git操作 | [lazygit](https://github.com/jesseduffield/lazygit) | CLIなのにGUI並みの視認性 |
| 検索 | [fzf](https://github.com/junegunn/fzf) + [ripgrep](https://github.com/BurntSushi/ripgrep) | Cmd+Pより速いファジー検索 |
| AIコーディング | [Claude Code](https://claude.ai/claude-code) | ターミナルネイティブで相性抜群 |

## キーバインド早見表

### Herdr（画面分割・エージェント管理）

Herdr の prefix は `Ctrl+Q`。ただし日常操作は **Ghostty 側の Cmd キーが「prefix + コマンド」を 1 打で送る** ので、prefix を押す機会はほぼ無い。

| キー (Ghostty) | 動作 | Herdr 側の元の割当 |
|------|------|------|
| `Cmd+T` | 新規タブ | `prefix+c` |
| `Cmd+W` / `Cmd+Shift+W` | ペインを閉じる / タブを閉じる | `prefix+x` / `prefix+Shift+X` |
| `Cmd+D` / `Cmd+Shift+D` | 右に分割 / 下に分割 | `prefix+v` / `prefix+-` |
| `Cmd+1`〜`Cmd+9` | タブ N へ移動 | `prefix+1..9` |
| `Cmd+Shift+]` / `Cmd+Shift+[` | 次 / 前のタブ | `prefix+n` / `prefix+p` |
| `Cmd+]` | 次のペインへ | `prefix+Tab` |
| `Cmd+Opt+←↓↑→` | ペイン移動 | `prefix+h/j/k/l` |
| `Cmd+Shift+Enter` | ペインをズーム | `prefix+z` |
| `Cmd+R` | リサイズモード | `prefix+r` |
| `Cmd+B` | サイドバー (Agents 一覧) 表示切替 | `prefix+b` |
| `Cmd+N` | 新規ワークスペース (プロジェクト単位) | `prefix+Shift+N` |
| `Cmd+Shift+G` | 新規 git worktree | `prefix+Shift+G` |
| `Cmd+G` | ナビゲート (ワークスペース / ペインへジャンプ) | `prefix+g` |
| `Cmd+O` | 通知元のエージェントへジャンプ | `prefix+o` |
| `Cmd+Shift+R` / `Cmd+Opt+R` | タブ名 / ペイン名の変更 | `prefix+Shift+T` / `prefix+Shift+P` |
| `Cmd+K` | 画面クリア (`Ctrl+L` をペインへ) | — |
| `Ctrl+;` | `continue` + Enter を送る (Claude Code 再開用) | — |
| `Ctrl+Q` → `?` | Herdr の全キー一覧 | `prefix+?` |
| `Ctrl+Q` → `q` | デタッチ (エージェントは走り続ける) | `prefix+q` |

Herdr を AI から操作する例 (Socket API):

```bash
herdr workspace create --cwd ~/dev/foo --label foo   # ワークスペース作成
herdr pane split w1:p1 --direction right              # ペイン分割
herdr worktree create --branch feature/x              # worktree 作成
herdr agent list                                      # 検知中のエージェント一覧
herdr agent wait w1:p1 --until blocked                # 入力待ちになるまで待機
```

### lazygit

| キー | 動作 |
|------|------|
| `space` | ステージング切り替え |
| `c` | コミット |
| `p` | プッシュ |
| `P` | プル |
| `?` | ヘルプ |
| `q` | 終了 |

### yazi（ファイラー）

| キー | 動作 |
|------|------|
| `y` | yazi起動（シェル連携） |
| `j/k` | 上下移動 |
| `l` | 開く/入る |
| `h` | 親ディレクトリ |
| `q` | 終了 |

### fzf（検索）

| キー | 動作 |
|------|------|
| `Ctrl+T` | ファイル検索 |
| `Ctrl+R` | コマンド履歴検索 |
| `Alt+C` | ディレクトリ移動 |

## セットアップ手順

### 1. ツールのインストール

```bash
# 基本ツール
brew install herdr yazi lazygit fzf ripgrep

# yaziの依存関係
brew install ffmpeg sevenzip jq poppler fd imagemagick
```

### 2. シェル連携（.zshrc）

```bash
# yazi - ディレクトリ移動連携
function y() {
  local tmp="$(mktemp -t "yazi-cwd.XXXXXX")" cwd
  yazi "$@" --cwd-file="$tmp"
  if cwd="$(command cat -- "$tmp")" && [ -n "$cwd" ] && [ "$cwd" != "$PWD" ]; then
    builtin cd -- "$cwd"
  fi
  rm -f -- "$tmp"
}

# fzf
source <(fzf --zsh)
export FZF_DEFAULT_COMMAND='rg --files --hidden --glob "!.git"'
export FZF_CTRL_T_COMMAND="$FZF_DEFAULT_COMMAND"
```

### 3. Ghostty設定

```
# ~/.config/ghostty/config (抜粋。Cmd キー割当の全文は実ファイル参照)
command = direct:/opt/homebrew/bin/herdr
font-family = "JetBrainsMono Nerd Font Mono"
macos-option-as-alt = true

# Herdr の prefix (ctrl+q = \x11) を Cmd キーでバイパスする
keybind = super+t=text:\x11c
keybind = super+d=text:\x11v
keybind = super+shift+d=text:\x11-
```

### 4. Herdr設定

```toml
# ~/.config/herdr/config.toml
[theme]
name = "terminal"          # Ghostty の配色をそのまま使う

[terminal]
new_cwd = "follow"         # 新規ペインは元ペインの CWD を引き継ぐ

[keys]
prefix = "ctrl+q"          # 既定の ctrl+b は Claude Code / vim と衝突する

[ui]
prompt_new_tab_name = false

[ui.toast]
delivery = "off"           # 完了通知は Claude Code の Stop hook に任せる (terminal だと Ghostty 経由で二重に出る)

[ui.sound]
enabled = false            # 見ていない workspace で状態が変わるたびに鳴るので切る

[experimental]
switch_ascii_input_source_in_prefix = true   # 日本語 IME が ON でも prefix が通る
reveal_hidden_cursor_for_cjk_ime = true      # Claude Code で IME 候補窓がカーソル追従
cjk_ime_agents = ["claude"]
```

反映: `herdr server reload-config`。素の zsh が欲しい時: `open -na Ghostty --args -e zsh`

## 設定ファイルの場所

```
~/.config/
├── ghostty/config       # Ghostty設定 (Herdr 起動 + Cmd キー割当)
├── herdr/config.toml    # Herdr設定
├── yazi/                # yazi設定
└── lazygit/             # lazygit設定

~/.zshrc                 # シェル連携設定
```

## 参考

- Herdr 公式: https://herdr.dev/ (install / docs/configuration / docs/cli-reference)
- Herdr + git worktree 並列運用 (Claude Code から worktree を切って起動): https://zenn.dev/gemcook/articles/herdr-worktree-parallel
