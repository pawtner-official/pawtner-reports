# Mac mini Remote Control 常駐セットアップ

iPhone / claude.ai から「常に待機している Claude Code セッション」に接続し、
Mac mini 上の T7（`/Volumes/T7`）を読み書きさせるための常駐設定です。

## なぜ必要か

Claude Code のセッションは実行場所が 2 種類あります。

| Environment | 実行場所 | T7 が見えるか |
|---|---|---|
| Local | Mac mini 本体 | 見える |
| Cloud | Anthropic のクラウド VM | 見えない |

iPhone やブラウザで新規セッションを作ると Cloud になるため T7 に届きません。
一方 Remote Control は Mac mini 上で走るセッションを iPhone から操作する仕組みで、
実行とファイルアクセスは Mac mini に残ります。

ただし Remote Control は「Mac mini 側でプロセスが動いていること」が前提で、
iPhone からプロセスを起動することはできません。そこで launchd で常駐させ、
いつでも iPhone から繋がる状態を維持します。

## セットアップ

Mac mini のターミナルで、この 2 行を貼るだけです。

```bash
curl -fsSL https://raw.githubusercontent.com/pawtner-official/pawtner-reports/claude/t7-unmount-incident-6rtkb9/tools/claude-rc-setup.sh -o /tmp/claude-rc-setup.sh
bash /tmp/claude-rc-setup.sh
```

作業対象を変える場合:

```bash
PROJECT_DIR=/Volumes/T7/別フォルダ SESSION_NAME=MacMini-Sub bash /tmp/claude-rc-setup.sh
```

スクリプトが自動で行うこと:

- 信頼ダイアログの誘導（未承認なら claude を起動し、終了後に自動で続行）
- `~/bin/claude-rc.sh` の生成（T7 のマウントを最大 5 分待ってから起動）
- `~/Library/LaunchAgents/com.pawtner.claude-rc.plist` の生成・登録
- `KeepAlive` によるネットワーク断・プロセス終了からの自動復帰
- `pmset` でスリープ無効化・停電復帰後の自動起動・WOL を設定
- 起動確認とセッション URL の表示
- フルディスクアクセス設定パネルを開く

自動化できない残作業は 2 つだけです。

1. **フルディスクアクセス** — ターミナル.app を追加（パネルは自動で開きます）
2. **自動ログイン** — システム設定 → ユーザとグループ → 自動ログイン

## iPhone からの使い方

Claude アプリ → **Code** タブ → `MacMini-T7`（PC アイコン + 緑のドット）をタップ。

サーバーモードなので、iPhone 側から追加のセッションを立ち上げることもできます。

**注意**: iPhone で「新規タスク」を作ると Cloud セッションになり T7 に届きません。
必ず一覧から `MacMini-T7` を開いてください。

## 運用

```bash
launchctl kickstart -k gui/$(id -u)/com.pawtner.claude-rc   # 再起動
launchctl bootout   gui/$(id -u)/com.pawtner.claude-rc      # 停止
tail -f ~/Library/Logs/claude-rc.err                        # ログ
```

## T7 のアンマウント対策

`T7 が約1分間アンマウントされていました` のようなアラートが出る場合、
起動待ちループでは復帰しきれないことがあります。以下を先に潰しておいてください。

- ハブ経由をやめて本体直挿しにする
- `sudo pmset -a disksleep 0` でディスクスリープを無効化
- 頻発する場合はケーブル交換、ディスクユーティリティの First Aid
