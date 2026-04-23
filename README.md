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
