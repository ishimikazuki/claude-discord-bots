# claude-discord-bots

Discord から Claude Code CLI を呼び出すマルチボット基盤。
1 bot = 1 project の構成で、メンションされると専用フォーラムにスレッドを立て、
Claude Code にプロンプトを渡してその結果を Discord に返します。macOS で
LaunchAgent として常駐運用することを想定しています。

## Features

- **1 bot = 1 project**: bot ごとに作業ディレクトリ (`dir`) を分離
- **Thread-scoped sessions**: Discord スレッド単位で Claude Code の `--resume` を保持
- **Git worktree**: スレッドごとに worktree を自動作成（`worktree_enabled`）
- **Role mention 対応**: ユーザーメンション・ロールメンション両対応
- **Forum auto-thread**: `#一般` 等のチャンネルでメンションされたら bot 専用
  フォーラムに新スレッドを自動で立ててそこで会話を継続
- **Attachment passthrough**: 添付ファイルは `_inbox/` に保存して Claude に渡し、
  `_outbox/` に書き出されたファイルは自動で Discord に返信添付
- **LaunchAgent**: macOS の GUI セッション LaunchAgent として常駐。login
  keychain アクセスが必要な Claude Code OAuth と相性が良い

## Requirements

- macOS (LaunchAgent 運用を前提)
- Python 3.11+
- [Claude Code CLI](https://docs.claude.com/claude-code) にログイン済み
  (`claude` がパス上にあること)
- Discord bot アカウント (各 bot につき 1 つの token)

## Setup

### 1. Clone & install

```bash
git clone https://github.com/<you>/claude-discord-bots.git
cd claude-discord-bots
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 2. Configure

```bash
cp config.example.json config.json
cp .env.example .env
```

`config.json` を自分の環境に合わせて編集：

- `bots.<name>.dir`: 作業ディレクトリ（`~` 展開対応）
- `bots.<name>.control_channel_id`: bot が応答するチャンネル ID (nullable)
- `notify_channel_id`: 通知用チャンネル ID (nullable)
- `allowed_users`: Discord user ID の許可リスト（空なら全員許可）

`.env` に Discord bot token を入れるか、macOS keychain に登録：

```bash
security add-generic-password -a "general-bot-token" -s "discord-bot" -w "YOUR_TOKEN"
```

### 3. Run a single bot (for testing)

```bash
python bot.py general
```

### 4. Install as LaunchAgent (macOS production)

```bash
# plist を生成 → ~/Library/LaunchAgents/ に配置 → bootstrap & kickstart
bash launchd/install-macmini.sh

# dry-run で内容確認だけしたい場合
DRY_RUN=1 bash launchd/install-macmini.sh
```

## Architecture

```
Discord message
  └─ #一般 で @bot メンション
       └─ bot 専用フォーラムに thread を作成
             └─ thread 内で以降の会話
                  └─ Claude Code CLI (--resume <sessionId>)
                       └─ 応答 / 添付を Discord に返信
```

- **Session 管理**: `sessions-<bot>.json` に thread_id → sessionId のマッピングを
  保存（gitignored）
- **Worktree**: スレッドごとに `<dir>/.worktrees/thread-<id>` を作成して
  Claude を実行。他スレッドと干渉しない
- **Token**: `.env` → macOS keychain の順で解決

## Multi-machine sync (手元マシン ⇄ 常駐 Mac mini)

この基盤は「手元の開発マシンでコードを書き、Mac mini に bot を常駐させて
Discord から操作する」構成を想定しています。**同期は rsync/ssh ではなく
GitHub を中継した git pull/push だけで完結**させるのが基本設計です。

### 2 種類のリポジトリ

| 種別 | リポジトリ | 役割 | 例 |
|---|---|---|---|
| **bot 基盤** | この repo（＋ private fork） | `bot.py` / `config.json` / `launchd/` を Mac mini に配布 | `claude-discord-bots` (public) / `discord-bots` (private) |
| **各 bot の作業対象** | `bots.<name>.dir` 配下の git repo | Claude が編集する実プロジェクト。bot ごとに別 repo | `knowledge-hub`, `yumekano-agent-CoE` など |

一般公開したくない設定（実 channel_id を入れた `config.json` など）は
**private fork** 側で管理し、public repo 側はテンプレート (`config.example.json`)
だけ置く構成が推奨です。

### 同期フロー

```
┌─ 手元マシン ──────────────┐        ┌─ GitHub ─┐        ┌─ Mac mini (常駐 bot) ─┐
│ 1. コード編集             │        │          │        │                        │
│ 2. git commit && push ────┼──────►│  origin  │◄───────┼─ auto git pull         │
│                            │        │   main   │        │    (セッション開始前) │
│                            │        │          │        │                        │
│ 3. Discord で @bot         │        │          │        │ 4. worktree でブランチ │
│    (from 手元マシン)       │        │          │        │    作成 → Claude 実行 │
│                            │        │          │        │ 5. 結果 commit/push ──┼──┐
│ 6. git pull で取り込み ◄───┼────────┤          │◄───────┤                        │  │
└────────────────────────────┘        └──────────┘        └────────────────────────┘  │
                                           ▲                                            │
                                           └────────────────────────────────────────────┘
```

### 2 層の自動 pull

1. **bot 基盤の pull**: 手元で `config.json` や `bot.py` を更新したら `git push`。
   Mac mini 側では LaunchAgent を再起動する前に手動 `git pull` するか、
   以下のような cron を設定（任意）:
   ```bash
   */5 * * * * cd ~/discord-bots && git pull --ff-only --quiet
   ```
2. **作業対象 repo の pull**: `config.json` の `auto_pull_before_session: true`
   を有効にすると、各スレッドで Claude を呼ぶ**直前**に `git pull --ff-only` が
   走ります (`bot.py` の `git_pull()`)。手元での編集が Mac mini に自動で反映
   されるので、ユーザーは「push して Discord で話しかける」だけで良い状態に
   なります。

### スレッドごとの worktree 分離

`worktree_enabled: true` の場合、スレッドごとに
`<dir>/.worktrees/thread-<thread_id>` を作成して Claude がそこで作業するため、
**複数スレッドの変更が衝突しません**。作業が完了したら Claude が commit
して push すれば、手元側は `git pull` でそのブランチを取り込めます。

### Mac mini 側に必要な GitHub 認証

Mac mini の bot からコードを push させたい場合、Mac mini 側にも push 権限のある
認証情報が必要です（SSH 鍵を GitHub に登録、あるいは `gh auth login`）。
pull だけなら public repo ならノンクレでも可。

### よくある運用パターン

- **開発**: 手元マシンで `bot.py` / `config.json` を更新 → push →
  Mac mini で `git pull && launchctl kickstart ...` で反映
- **利用**: Discord でメンション → Mac mini の bot が auto-pull → Claude 実行
- **結果取り込み**: 手元で `git pull` して Claude が書いたコードをレビュー

## File layout

```
bot.py                   # メインエントリポイント (1 bot = 1 process)
attachments.py           # Discord 添付の入出力ヘルパー
mention_helpers.py       # メンション判定 / 除去
config.example.json      # 設定テンプレート
.env.example             # トークン置き場のテンプレート
launchd/
  generate-plists.sh     # config.json から plist を生成
  install-macmini.sh     # bootout → bootstrap → kickstart の一括実行
tests/                   # pytest + shell tests
```

## Testing

```bash
pytest tests/
bash tests/test_generate_plists.sh
bash tests/test_install_macmini.sh
```

## Security notes

- `.env`, `sessions-*.json`, `logs/`, `config.json` は `.gitignore` で除外済み
- token は `.env` もしくは macOS keychain のみに保存してください
- public 公開時は `config.json` をコミットしないこと（`config.example.json` を使う）
- LaunchAgent の PATH に `~/.npm-global/bin` を含める必要があれば
  `launchd/generate-plists.sh` を編集

## License

MIT
