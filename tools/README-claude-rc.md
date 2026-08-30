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

Mac mini のターミナルで一度だけ実行します。

```bash
# 1. 信頼ダイアログを通す（未実施の場合のみ）
cd /Volumes/T7/MASTER && claude    # 「信頼する」を選んで終了

# 2. 常駐設定
bash tools/claude-rc-setup.sh
```

作業対象を変える場合:

```bash
PROJECT_DIR=/Volumes/T7/別フォルダ SESSION_NAME=MacMini-Sub bash tools/claude-rc-setup.sh
```

スクリプトが行うこと:

- `~/bin/claude-rc.sh` を生成（T7 のマウントを最大 5 分待ってから起動）
- `~/Library/LaunchAgents/com.pawtner.claude-rc.plist` を生成・登録
- `KeepAlive` によりネットワーク断やプロセス終了から自動復帰
- 起動確認とログの場所を表示

スクリプト実行後、案内される手作業が 3 つあります
（スリープ無効化・フルディスクアクセス・プッシュ通知）。

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
