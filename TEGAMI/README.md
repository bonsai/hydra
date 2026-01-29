# Email-based LLM Agent System

メールでLLMエージェントに相談できるシステム。各エージェントが専用のメールアドレスを持ち、5分ごとにメールをチェックして自動返信します。

## 概要

- **目的**: メールインターフェースでLLMエージェントと対話
- **仕組み**: Pythonサーバーが常駐し、5分間隔でメールボックスをチェック
- **特徴**: プロンプト管理がメールスレッドで可能、複数エージェント対応

## システム構成

```
email-llm-agent/
├── README.md
├── blueprint.md          # 詳細設計書
├── requirements.txt
├── config.yaml          # エージェント設定
├── src/
│   ├── main.py         # メインサーバー
│   ├── email_client.py # メール送受信
│   ├── llm_client.py   # LLM API呼び出し
│   └── agent.py        # エージェントロジック
└── logs/
    └── agent.log
```

## 主な機能

### 1. マルチエージェント対応
- 各エージェントが専用メールアドレスを持つ
- エージェントごとに異なる役割・プロンプト設定

### 2. メールベースのプロンプト管理
- メールスレッドで会話履歴を管理
- 件名やラベルでカテゴリ分類
- 添付ファイル対応（画像、PDF、テキスト）

### 3. スケジューラー機能
- 5分間隔で自動チェック
- 未読メールのみ処理
- エラー時のリトライ機能

## クイックスタート

### 1. インストール

```bash
git clone <repository>
cd email-llm-agent
pip install -r requirements.txt
```

### 2. 設定ファイル作成

`config.yaml` を編集:

```yaml
agents:
  - name: "General Assistant"
    email: "llm@localhost"
    smtp_server: "localhost"
    smtp_port: 1025
    imap_server: "localhost"
    imap_port: 1143
    password: "your_password"
    llm_model: "claude-sonnet-4-5-20250929"
    system_prompt: "あなたは親切なアシスタントです。"
    
  - name: "Code Expert"
    email: "code@localhost"
    smtp_server: "localhost"
    smtp_port: 1025
    imap_server: "localhost"
    imap_port: 1143
    password: "your_password"
    llm_model: "claude-sonnet-4-5-20250929"
    system_prompt: "あなたはプログラミングの専門家です。"

llm:
  api_key: "your_anthropic_api_key"
  max_tokens: 4096
  temperature: 0.7

scheduler:
  check_interval: 300  # 5分 = 300秒
  max_retries: 3
  retry_delay: 60
```

### 3. サーバー起動

```bash
python src/main.py
```

### 4. 使い方

1. `llm@localhost` にメールを送信
2. 件名と本文にあなたの質問を記載
3. 5分以内にLLMからの返信が届く
4. スレッドで会話を続けられる

## エージェント例

### General Assistant (`llm@localhost`)
- 一般的な質問に回答
- 翻訳、要約、アイデア出し

### Code Expert (`code@localhost`)
- プログラミング相談
- コードレビュー、デバッグ支援

### Research Agent (`research@localhost`)
- 調査・リサーチ支援
- 情報整理、レポート作成

## 技術スタック

- **Python 3.9+**
- **IMAP/SMTP**: メール送受信
- **Anthropic Claude API**: LLM処理
- **APScheduler**: 定期実行
- **PyYAML**: 設定管理

## セキュリティ

- メール認証情報は環境変数または暗号化
- API keyは `.env` ファイルで管理
- ログに機密情報を記録しない

## ログ

```bash
# ログファイル確認
tail -f logs/agent.log
```

## トラブルシューティング

### メールが届かない
1. SMTP/IMAP設定を確認
2. ファイアウォール設定を確認
3. ログでエラーメッセージを確認

### LLM応答がない
1. API keyが正しいか確認
2. レート制限に達していないか確認
3. ネットワーク接続を確認

## 今後の拡張案

- [ ] Webダッシュボード追加
- [ ] Slack/Discord連携
- [ ] 音声メッセージ対応
- [ ] マルチモーダル対応（画像、動画）
- [ ] カスタムツール呼び出し
- [ ] データベース統合

## ライセンス

MIT License

## 作者

Your Name

---

詳細な設計については `blueprint.md` を参照してください。
