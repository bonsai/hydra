# Architecture Decision Record (ADR)

## Thunderbird + Ollama LLM Agent System

**日付**: 2026-01-29  
**ステータス**: 提案  
**決定者**: Development Team

---

## 1. コンテキスト

メールインターフェースでLLMエージェントと対話できるシステムを構築する。
画像のようにGmailではなく、Thunderbirdをメールクライアントとして使用し、ローカルでOllamaサーバーを実行する。

### 要件
- メールベースのLLM対話
- 5分間隔での自動チェック・返信
- プライバシー重視（ローカル実行）
- 複数エージェント対応
- プロンプト管理の容易性

### 制約
- インターネット接続不要で動作
- APIコスト削減
- データプライバシー保護
- 既存メールワークフローとの統合

---

## 2. 検討した選択肢

### 選択肢A: PythonスクリプトでIMAPポーリング（当初案）
- **メリット**: シンプル、柔軟
- **デメリット**: Thunderbirdと別プロセス、同期の問題

### 選択肢B: Thunderbirdアドオン開発
- **メリット**: ネイティブ統合、リアルタイム処理、UI統合
- **デメリット**: 開発コスト高、WebExtension API制約

### 選択肢C: Thunderbirdフォーク（カスタムビルド）
- **メリット**: 完全なカスタマイズ、制約なし
- **デメリット**: メンテナンスコスト大、アップデート追従が困難

### 選択肢D: ハイブリッドアプローチ（アドオン + バックエンドサービス）
- **メリット**: 柔軟性と統合の両立
- **デメリット**: 複雑性増加

---

## 3. 決定事項

### 主要決定

#### 3.1 アーキテクチャ: **ハイブリッドアプローチ（選択肢D）**

```
┌─────────────────────────────────────┐
│      Thunderbird クライアント       │
│  ┌───────────────────────────────┐  │
│  │  Thunderbirdアドオン          │  │
│  │  - メール監視                 │  │
│  │  - UI統合                     │  │
│  │  - 設定管理                   │  │
│  └───────────┬───────────────────┘  │
└──────────────┼───────────────────────┘
               │ WebSocket/HTTP
               ▼
┌─────────────────────────────────────┐
│    ローカルバックエンドサービス     │
│  ┌───────────────────────────────┐  │
│  │  Python/Node.js サーバー      │  │
│  │  - メール処理ロジック         │  │
│  │  - 会話履歴管理               │  │
│  │  - エージェント管理           │  │
│  └───────────┬───────────────────┘  │
└──────────────┼───────────────────────┘
               │ HTTP API
               ▼
┌─────────────────────────────────────┐
│       Ollama ローカルサーバー       │
│  - llama3.2, codellama, mistral... │
│  - API: http://localhost:11434     │
└─────────────────────────────────────┘
```

**理由**:
- アドオンでUI統合とリアルタイム監視
- バックエンドで複雑なロジック処理
- 各コンポーネントの独立性維持
- 将来的な拡張性確保

#### 3.2 LLMバックエンド: **Ollama**

**理由**:
- ✅ ローカル実行（プライバシー保護）
- ✅ APIコストゼロ
- ✅ オフライン動作
- ✅ 複数モデル対応
- ✅ OpenAI互換API
- ✅ 簡単なセットアップ

**代替案との比較**:
| | Ollama | Claude API | OpenAI API | LM Studio |
|---|---|---|---|---|
| コスト | 無料 | 従量課金 | 従量課金 | 無料 |
| プライバシー | ◎ | △ | △ | ◎ |
| オフライン | ◎ | × | × | ◎ |
| セットアップ | 簡単 | 簡単 | 簡単 | 中程度 |
| モデル選択 | 豊富 | 固定 | 固定 | 豊富 |

#### 3.3 メールクライアント: **Thunderbird**

**理由**:
- ✅ オープンソース
- ✅ 拡張性（アドオンAPI）
- ✅ ローカルメール管理
- ✅ クロスプラットフォーム
- ✅ 既存ワークフローとの統合

#### 3.4 アドオン vs フォーク: **まずアドオン、必要に応じてフォーク**

**フェーズ1: アドオン開発**
- WebExtension APIで実装
- 80%のユースケースをカバー
- リリース・配布が容易

**フェーズ2: フォーク検討（オプション）**
- アドオンAPIの制約に直面した場合
- より深い統合が必要な場合
- カスタムビルドとして配布

---

## 4. 詳細設計

### 4.1 Thunderbirdアドオン仕様

#### 技術スタック
- **API**: MailExtensions (WebExtensions for Thunderbird)
- **言語**: JavaScript/TypeScript
- **UI**: HTML/CSS (Thunderbird組み込み)
- **通信**: WebSocket / REST API

#### 主要機能

```javascript
// manifest.json
{
  "manifest_version": 2,
  "name": "Ollama LLM Agent",
  "version": "1.0.0",
  "description": "メールでローカルLLMと対話",
  "permissions": [
    "messagesRead",
    "messagesUpdate", 
    "compose",
    "storage",
    "accountsRead"
  ],
  "background": {
    "scripts": ["background.js"]
  },
  "options_ui": {
    "page": "options.html"
  }
}
```

**コア機能**:
1. **メール監視**: 特定アドレス宛メールを検知
2. **自動返信**: LLM応答を返信メールとして送信
3. **エージェント切替**: 件名プレフィックスでエージェント選択
4. **設定UI**: Ollamaサーバー接続、エージェント設定
5. **履歴管理**: スレッド単位で会話コンテキスト保持

### 4.2 バックエンドサービス仕様

#### 技術スタック（オプション1: Python）
```python
# FastAPI + Ollama Python SDK
from fastapi import FastAPI
from ollama import Client

app = FastAPI()
ollama_client = Client(host='http://localhost:11434')

@app.post("/chat")
async def chat(message: str, model: str = "llama3.2"):
    response = ollama_client.chat(
        model=model,
        messages=[{"role": "user", "content": message}]
    )
    return response
```

#### 技術スタック（オプション2: Node.js）
```javascript
// Express + Ollama API
const express = require('express');
const axios = require('axios');

const app = express();
const OLLAMA_API = 'http://localhost:11434';

app.post('/chat', async (req, res) => {
  const response = await axios.post(`${OLLAMA_API}/api/chat`, {
    model: req.body.model || 'llama3.2',
    messages: req.body.messages
  });
  res.json(response.data);
});
```

**選択**: Node.js（理由: Thunderbirdアドオンとの親和性）

### 4.3 Ollama設定

#### 推奨モデル
```bash
# 汎用アシスタント
ollama pull llama3.2

# コード専門
ollama pull codellama

# 日本語特化
ollama pull llama3.2-ja

# 軽量高速
ollama pull phi3
```

#### モデル選択基準
| モデル | サイズ | 用途 | メモリ |
|---|---|---|---|
| llama3.2:3b | 2GB | 汎用・高速 | 8GB |
| llama3.2:7b | 4GB | 汎用・高品質 | 16GB |
| codellama:7b | 4GB | コーディング | 16GB |
| mistral:7b | 4GB | 汎用・高性能 | 16GB |

### 4.4 データフロー

```
1. ユーザーがllm@localhostにメール送信
   ↓
2. Thunderbirdアドオンがメール受信を検知
   ↓
3. メール内容を抽出（件名、本文、添付ファイル）
   ↓
4. バックエンドサービスにリクエスト送信
   WebSocket: ws://localhost:8080/chat
   または
   REST: POST http://localhost:8080/chat
   ↓
5. バックエンドが会話履歴を取得
   ↓
6. Ollama APIを呼び出し
   POST http://localhost:11434/api/chat
   ↓
7. LLM応答を受信（ストリーミング可）
   ↓
8. バックエンドが応答を整形
   ↓
9. アドオンが返信メールを作成・送信
   ↓
10. メールを既読にマーク
```

### 4.5 エージェント管理

#### 設定ファイル形式
```json
{
  "agents": [
    {
      "id": "general",
      "email": "llm@localhost",
      "model": "llama3.2:7b",
      "system_prompt": "あなたは親切なアシスタントです。",
      "temperature": 0.7,
      "max_tokens": 4096
    },
    {
      "id": "code",
      "email": "code@localhost",
      "model": "codellama:7b",
      "system_prompt": "あなたはプログラミング専門家です。",
      "temperature": 0.3,
      "max_tokens": 8192
    },
    {
      "id": "japanese",
      "email": "ja@localhost",
      "model": "llama3.2-ja:7b",
      "system_prompt": "日本語で丁寧に回答してください。",
      "temperature": 0.7,
      "max_tokens": 4096
    }
  ],
  "ollama_endpoint": "http://localhost:11434",
  "check_interval": 300
}
```

---

## 5. 実装計画

### Phase 1: MVP（2-3週間）
- [ ] Ollama環境セットアップ
- [ ] 基本的なThunderbirdアドオン
  - [ ] メール監視機能
  - [ ] 単純な返信機能
- [ ] シンプルなバックエンド（Node.js）
- [ ] 1つのエージェント対応

### Phase 2: 基本機能拡張（2-3週間）
- [ ] 複数エージェント対応
- [ ] スレッド管理・会話履歴
- [ ] 設定UI
- [ ] エラーハンドリング

### Phase 3: 高度な機能（3-4週間）
- [ ] 添付ファイル処理（画像、PDF）
- [ ] コードブロック整形
- [ ] ストリーミング応答
- [ ] パフォーマンス最適化

### Phase 4: フォーク検討（オプション）
- [ ] アドオンAPIの限界評価
- [ ] フォークの必要性判断
- [ ] カスタムビルド作成

---

## 6. ディレクトリ構造

```
thunderbird-ollama-agent/
├── addon/                      # Thunderbirdアドオン
│   ├── manifest.json
│   ├── background.js          # バックグラウンドスクリプト
│   ├── options.html           # 設定UI
│   ├── options.js
│   ├── popup.html             # ポップアップUI（オプション）
│   └── icons/
│       ├── icon-32.png
│       └── icon-64.png
├── backend/                    # バックエンドサービス
│   ├── package.json
│   ├── server.js              # Express サーバー
│   ├── ollama-client.js       # Ollama API クライアント
│   ├── agent-manager.js       # エージェント管理
│   ├── conversation.js        # 会話履歴管理
│   └── config.json            # エージェント設定
├── docs/
│   ├── ADR.md                 # このファイル
│   ├── README.md
│   └── SETUP.md               # セットアップガイド
├── tests/
│   ├── addon.test.js
│   └── backend.test.js
└── scripts/
    ├── build-addon.sh         # アドオンビルド
    ├── install-ollama.sh      # Ollama セットアップ
    └── dev-setup.sh           # 開発環境セットアップ
```

---

## 7. リスクと緩和策

### リスク1: Ollama性能不足
**影響**: 応答が遅い、品質が低い  
**緩和策**: 
- 軽量モデル（3B）から開始
- 必要に応じてクラウドAPIへフォールバック
- モデルの段階的アップグレード

### リスク2: メモリ不足
**影響**: システムが不安定  
**緩和策**:
- モデルサイズに応じた推奨スペック明記
- スワップ設定の最適化
- メモリ監視機能

### リスク3: Thunderbirdアドオン API制約
**影響**: 必要な機能が実装できない  
**緩和策**:
- Phase 1で技術検証
- 代替アプローチの検討
- フォークへのピボット準備

### リスク4: メール処理の遅延
**影響**: リアルタイム性の低下  
**緩和策**:
- 非同期処理の徹底
- ストリーミング応答対応
- タイムアウト設定

---

## 8. 成功指標

### 定量指標
- [ ] 応答時間: メール受信から返信まで < 30秒
- [ ] メモリ使用量: < 8GB（3Bモデル）
- [ ] CPU使用率: 推論時 < 80%
- [ ] エラー率: < 1%

### 定性指標
- [ ] セットアップが30分以内で完了
- [ ] 既存メールワークフローを阻害しない
- [ ] LLM応答品質がユーザー満足度を満たす
- [ ] プライバシーが完全に保護される

---

## 9. 代替案との比較まとめ

| 観点 | Thunderbird + Ollama | Gmail + Claude API | Python IMAP Script |
|---|---|---|---|
| プライバシー | ◎ ローカル完結 | △ クラウド送信 | ◎ ローカル完結 |
| コスト | ◎ 無料 | △ 従量課金 | △ API課金 |
| セットアップ | ○ やや複雑 | ◎ 簡単 | ○ 中程度 |
| UI統合 | ◎ ネイティブ | ◎ ブラウザ | × 別アプリ |
| オフライン | ◎ 可能 | × 不可 | × 不可 |
| 拡張性 | ◎ 高い | △ API制限 | ◎ 高い |
| メンテナンス | ○ 中程度 | ◎ 低い | ○ 中程度 |

**総合評価**: Thunderbird + Ollamaがプライバシー重視、コスト重視のユースケースに最適

---

## 10. 次のステップ

### 即座に実行
1. Ollamaインストール・モデルダウンロード
2. Thunderbird開発環境セットアップ
3. 簡単なPoC（Proof of Concept）アドオン作成

### 短期（1-2週間）
1. MVPアドオン実装
2. バックエンドサービス実装
3. 基本的な動作確認

### 中期（1-2ヶ月）
1. 機能拡張
2. ユーザーテスト
3. ドキュメント整備

### 長期（3-6ヶ月）
1. フォーク必要性の評価
2. コミュニティフィードバック収集
3. 公開リリース準備

---

## 11. 参考資料

### Thunderbird開発
- [MailExtensions API](https://webextension-api.thunderbird.net/)
- [Thunderbird Add-on Developer Guide](https://developer.thunderbird.net/)

### Ollama
- [Ollama Documentation](https://ollama.ai/docs)
- [Ollama API Reference](https://github.com/ollama/ollama/blob/main/docs/api.md)

### 類似プロジェクト
- Mailpilot (AI email assistant)
- SaneBox (email management)
- Various Thunderbird AI addons

---

## 承認

- [ ] 技術リード承認
- [ ] プロダクトオーナー承認
- [ ] セキュリティレビュー完了

---

**改訂履歴**:
- 2026-01-29: 初版作成
