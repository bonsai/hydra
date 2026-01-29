# Blueprint: Email-based LLM Agent System

## 1. システムアーキテクチャ

### 1.1 全体構成

```
┌─────────────┐
│   ユーザー   │
└──────┬──────┘
       │ メール送信
       ▼
┌─────────────────────────┐
│   ローカルメールサーバー │
│   (SMTP/IMAP)           │
└──────┬──────────────────┘
       │ 5分ごとにチェック
       ▼
┌─────────────────────────┐
│   Pythonエージェント     │
│   - メール取得          │
│   - LLM API呼び出し     │
│   - 返信メール送信      │
└──────┬──────────────────┘
       │
       ▼
┌─────────────────────────┐
│   Anthropic Claude API  │
└─────────────────────────┘
```

### 1.2 ローカルメールサーバー仕様

**簡易セットアップ推奨**: ローカル開発用に軽量メールサーバーを使用

#### 推奨ツール
1. **MailHog** (開発用)
   - SMTP: localhost:1025
   - Web UI: http://localhost:8025
   - IMAP不要（API経由でメール取得）

2. **MailDev** (開発用)
   - SMTP: localhost:1025
   - Web UI: http://localhost:1080
   - REST API対応

3. **Postfix + Dovecot** (本番用)
   - SMTP: localhost:25
   - IMAP: localhost:143

#### 簡略化メールヘッダ仕様

ローカル環境では最小限のヘッダのみ使用:

```
From: user@localhost
To: llm@localhost
Subject: プログラミング質問
Date: 2026-01-29 23:00:00

ここに本文
```

**必須ヘッダのみ**:
- `From`: 送信元
- `To`: 宛先
- `Subject`: 件名
- `Date`: 日時

**オプション**（スレッド管理用）:
- `Message-ID`: メッセージ識別子
- `In-Reply-To`: 返信先ID

**不要なヘッダ**（ローカルでは省略可）:
- `Received`: 経路情報
- `DKIM-Signature`: 署名
- `SPF`: スパム対策
- `Return-Path`: 返信先
- `MIME-Version`: 簡易テキストのみなら不要

### 1.2 コンポーネント設計

#### A. メールクライアント (email_client.py)
- **役割**: IMAP/SMTPでメール送受信
- **機能**:
  - 未読メール取得
  - メール本文・添付ファイル解析
  - 返信メール送信
  - スレッド管理

#### B. LLMクライアント (llm_client.py)
- **役割**: Claude APIとの通信
- **機能**:
  - プロンプト構築
  - ストリーミング対応
  - エラーハンドリング
  - レート制限管理

#### C. エージェント (agent.py)
- **役割**: メールとLLMの橋渡し
- **機能**:
  - メール内容の解析
  - 会話履歴管理
  - コンテキスト構築
  - 応答生成

#### D. メインサーバー (main.py)
- **役割**: 全体の制御
- **機能**:
  - スケジューラー管理
  - エージェント起動
  - ログ管理
  - 設定読み込み

## 2. データフロー

### 2.1 メール受信フロー

```
1. スケジューラーが5分ごとに起動
   ↓
2. 各エージェントのメールボックスをチェック
   ↓
3. 未読メールを取得
   ↓
4. メール内容を解析
   - 差出人
   - 件名
   - 本文
   - 添付ファイル
   - スレッドID
   ↓
5. 会話履歴を取得（スレッド内の過去メール）
   ↓
6. LLMプロンプトを構築
   ↓
7. Claude APIを呼び出し
   ↓
8. 応答を生成
   ↓
9. 返信メールを送信
   ↓
10. メールを既読にマーク
```

### 2.2 プロンプト構築フロー

```python
# 疑似コード
def build_prompt(email, thread_history, agent_config):
    prompt = {
        "system": agent_config.system_prompt,
        "messages": [
            # スレッド履歴
            {"role": "user", "content": thread_history[0].body},
            {"role": "assistant", "content": thread_history[1].body},
            ...
            # 最新メール
            {"role": "user", "content": email.body}
        ]
    }
    return prompt
```

## 3. データベース設計（オプション）

### 3.1 テーブル構造

#### emails テーブル
```sql
CREATE TABLE emails (
    id INTEGER PRIMARY KEY,
    agent_email VARCHAR(255),
    sender VARCHAR(255),
    subject TEXT,
    body TEXT,
    thread_id VARCHAR(255),
    message_id VARCHAR(255),
    created_at TIMESTAMP,
    processed BOOLEAN DEFAULT FALSE
);
```

#### conversations テーブル
```sql
CREATE TABLE conversations (
    id INTEGER PRIMARY KEY,
    thread_id VARCHAR(255),
    agent_email VARCHAR(255),
    user_email VARCHAR(255),
    started_at TIMESTAMP,
    last_message_at TIMESTAMP,
    message_count INTEGER
);
```

#### llm_requests テーブル
```sql
CREATE TABLE llm_requests (
    id INTEGER PRIMARY KEY,
    email_id INTEGER,
    model VARCHAR(100),
    prompt_tokens INTEGER,
    completion_tokens INTEGER,
    cost DECIMAL(10, 6),
    latency_ms INTEGER,
    created_at TIMESTAMP,
    FOREIGN KEY (email_id) REFERENCES emails(id)
);
```

## 4. 詳細実装

### 4.1 main.py

```python
import logging
from apscheduler.schedulers.blocking import BlockingScheduler
from src.agent import Agent
from src.config import load_config

def main():
    # ロガー設定
    logging.basicConfig(
        level=logging.INFO,
        format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        handlers=[
            logging.FileHandler('logs/agent.log'),
            logging.StreamHandler()
        ]
    )
    
    # 設定読み込み
    config = load_config('config.yaml')
    
    # エージェント初期化
    agents = [Agent(agent_config) for agent_config in config['agents']]
    
    # スケジューラー設定
    scheduler = BlockingScheduler()
    
    def check_emails():
        for agent in agents:
            try:
                agent.process_emails()
            except Exception as e:
                logging.error(f"Agent {agent.name} error: {e}")
    
    # 5分ごとに実行
    scheduler.add_job(
        check_emails,
        'interval',
        seconds=config['scheduler']['check_interval']
    )
    
    logging.info("Email LLM Agent Server Started")
    scheduler.start()

if __name__ == "__main__":
    main()
```

### 4.2 email_client.py (ローカルメール用簡易版)

```python
import imaplib
import smtplib
from email.mime.text import MIMEText
from email.parser import BytesParser
from email import policy
from datetime import datetime

class EmailClient:
    def __init__(self, config):
        self.email = config['email']
        self.password = config.get('password', '')  # ローカルは認証不要の場合も
        self.imap_server = config.get('imap_server', 'localhost')
        self.imap_port = config.get('imap_port', 143)
        self.smtp_server = config.get('smtp_server', 'localhost')
        self.smtp_port = config.get('smtp_port', 1025)
        self.use_ssl = config.get('use_ssl', False)  # ローカルはSSL不要
    
    def fetch_unread_emails(self):
        """未読メールを取得（簡易版）"""
        # ローカル環境では認証なしの場合
        if self.use_ssl:
            imap = imaplib.IMAP4_SSL(self.imap_server, self.imap_port)
        else:
            imap = imaplib.IMAP4(self.imap_server, self.imap_port)
        
        # パスワードがある場合のみログイン
        if self.password:
            imap.login(self.email, self.password)
        
        imap.select('INBOX')
        
        # 未読メール検索
        status, messages = imap.search(None, 'UNSEEN')
        email_ids = messages[0].split()
        
        emails = []
        for email_id in email_ids:
            status, msg_data = imap.fetch(email_id, '(RFC822)')
            email_body = msg_data[0][1]
            message = BytesParser(policy=policy.default).parsebytes(email_body)
            
            emails.append({
                'id': email_id,
                'from': message.get('From', 'unknown@localhost'),
                'subject': message.get('Subject', '(no subject)'),
                'body': self._get_email_body(message),
                'message_id': message.get('Message-ID', f'<{email_id}@localhost>'),
                'in_reply_to': message.get('In-Reply-To'),
                'date': message.get('Date', datetime.now().isoformat())
            })
        
        imap.close()
        imap.logout()
        return emails
    
    def _get_email_body(self, message):
        """メール本文を取得（簡易版）"""
        # プレーンテキストのみ対応
        if message.is_multipart():
            for part in message.walk():
                if part.get_content_type() == 'text/plain':
                    return part.get_payload(decode=True).decode('utf-8', errors='ignore')
        else:
            payload = message.get_payload(decode=True)
            if payload:
                return payload.decode('utf-8', errors='ignore')
        return ""
    
    def send_reply(self, to_email, subject, body, in_reply_to=None):
        """返信メールを送信（簡易版）"""
        msg = MIMEText(body, 'plain', 'utf-8')
        msg['From'] = self.email
        msg['To'] = to_email
        msg['Subject'] = f"Re: {subject}" if not subject.startswith('Re:') else subject
        msg['Date'] = datetime.now().strftime('%a, %d %b %Y %H:%M:%S +0900')
        
        # スレッド管理用（オプション）
        if in_reply_to:
            msg['In-Reply-To'] = in_reply_to
            msg['References'] = in_reply_to
        
        # ローカルはSTARTTLS不要
        with smtplib.SMTP(self.smtp_server, self.smtp_port) as smtp:
            if self.password:
                smtp.login(self.email, self.password)
            smtp.send_message(msg)
    
    def mark_as_read(self, email_id):
        """メールを既読にする（簡易版）"""
        if self.use_ssl:
            imap = imaplib.IMAP4_SSL(self.imap_server, self.imap_port)
        else:
            imap = imaplib.IMAP4(self.imap_server, self.imap_port)
        
        if self.password:
            imap.login(self.email, self.password)
        
        imap.select('INBOX')
        imap.store(email_id, '+FLAGS', '\\Seen')
        imap.close()
        imap.logout()
```

### 4.2.1 MailHog用アダプター（IMAP不要版）

MailHogにはIMAPがないため、HTTP APIを使用:

```python
import requests
from email.parser import BytesParser
from email import policy

class MailHogClient:
    """MailHog用の簡易メールクライアント"""
    
    def __init__(self, config):
        self.email = config['email']
        self.api_url = config.get('api_url', 'http://localhost:8025/api/v2')
        self.smtp_server = config.get('smtp_server', 'localhost')
        self.smtp_port = config.get('smtp_port', 1025)
        self.processed_ids = set()
    
    def fetch_unread_emails(self):
        """メール取得（HTTP API経由）"""
        response = requests.get(f"{self.api_url}/messages")
        messages = response.json()
        
        emails = []
        for msg in messages.get('items', []):
            msg_id = msg['ID']
            
            # 既に処理済みならスキップ
            if msg_id in self.processed_ids:
                continue
            
            # 自分宛のメールのみ
            to_addresses = [to['Mailbox'] + '@' + to['Domain'] 
                          for to in msg['To']]
            if self.email not in to_addresses:
                continue
            
            emails.append({
                'id': msg_id,
                'from': f"{msg['From']['Mailbox']}@{msg['From']['Domain']}",
                'subject': msg['Content']['Headers'].get('Subject', [''])[0],
                'body': msg['Content']['Body'],
                'message_id': msg['Content']['Headers'].get('Message-ID', [f'<{msg_id}>'])[0],
                'in_reply_to': msg['Content']['Headers'].get('In-Reply-To', [None])[0],
                'date': msg['Created']
            })
        
        return emails
    
    def send_reply(self, to_email, subject, body, in_reply_to=None):
        """返信送信（SMTP）"""
        from email.mime.text import MIMEText
        import smtplib
        from datetime import datetime
        
        msg = MIMEText(body, 'plain', 'utf-8')
        msg['From'] = self.email
        msg['To'] = to_email
        msg['Subject'] = f"Re: {subject}" if not subject.startswith('Re:') else subject
        msg['Date'] = datetime.now().strftime('%a, %d %b %Y %H:%M:%S +0900')
        
        if in_reply_to:
            msg['In-Reply-To'] = in_reply_to
            msg['References'] = in_reply_to
        
        with smtplib.SMTP(self.smtp_server, self.smtp_port) as smtp:
            smtp.send_message(msg)
    
    def mark_as_read(self, email_id):
        """既読マーク（処理済みリストに追加）"""
        self.processed_ids.add(email_id)
        
        # オプション: MailHogからメール削除
        # requests.delete(f"{self.api_url}/messages/{email_id}")
```

### 4.2.2 ファイルベースクライアント（メールサーバー不要）

```python
import os
import glob
from datetime import datetime

class FilesystemClient:
    """ファイルベースの超簡易メールクライアント"""
    
    def __init__(self, config):
        self.email = config['email']
        self.inbox_dir = config['inbox_dir']
        self.outbox_dir = config['outbox_dir']
        
        # ディレクトリ作成
        os.makedirs(self.inbox_dir, exist_ok=True)
        os.makedirs(self.outbox_dir, exist_ok=True)
    
    def fetch_unread_emails(self):
        """受信メール取得（ファイル読み込み）"""
        emails = []
        
        # inbox内の未処理ファイル
        for filepath in glob.glob(f"{self.inbox_dir}/*.txt"):
            with open(filepath, 'r', encoding='utf-8') as f:
                content = f.read()
            
            # 簡易パース
            headers = {}
            body = ""
            lines = content.split('\n')
            
            # ヘッダ部分
            i = 0
            while i < len(lines) and lines[i].strip():
                if ':' in lines[i]:
                    key, value = lines[i].split(':', 1)
                    headers[key.strip()] = value.strip()
                i += 1
            
            # 本文部分
            body = '\n'.join(lines[i:]).strip()
            
            emails.append({
                'id': filepath,
                'from': headers.get('From', 'unknown@localhost'),
                'subject': headers.get('Subject', '(no subject)'),
                'body': body,
                'message_id': headers.get('Message-ID', f'<{os.path.basename(filepath)}>'),
                'date': headers.get('Date', datetime.now().isoformat())
            })
        
        return emails
    
    def send_reply(self, to_email, subject, body, in_reply_to=None):
        """返信メール送信（ファイル書き込み）"""
        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        filename = f"reply_{timestamp}.txt"
        filepath = os.path.join(self.outbox_dir, filename)
        
        with open(filepath, 'w', encoding='utf-8') as f:
            f.write(f"From: {self.email}\n")
            f.write(f"To: {to_email}\n")
            f.write(f"Subject: Re: {subject}\n")
            f.write(f"Date: {datetime.now().isoformat()}\n")
            if in_reply_to:
                f.write(f"In-Reply-To: {in_reply_to}\n")
            f.write("\n")
            f.write(body)
    
    def mark_as_read(self, email_id):
        """既読マーク（ファイル削除 or リネーム）"""
        # 処理済みディレクトリに移動
        processed_dir = os.path.join(os.path.dirname(self.inbox_dir), 'processed')
        os.makedirs(processed_dir, exist_ok=True)
        
        filename = os.path.basename(email_id)
        new_path = os.path.join(processed_dir, filename)
        os.rename(email_id, new_path)
```

### 4.3 llm_client.py

```python
import anthropic
from typing import List, Dict

class LLMClient:
    def __init__(self, api_key, config):
        self.client = anthropic.Anthropic(api_key=api_key)
        self.model = config.get('model', 'claude-sonnet-4-5-20250929')
        self.max_tokens = config.get('max_tokens', 4096)
        self.temperature = config.get('temperature', 0.7)
    
    def generate_response(self, system_prompt: str, messages: List[Dict]) -> str:
        """LLM応答を生成"""
        try:
            response = self.client.messages.create(
                model=self.model,
                max_tokens=self.max_tokens,
                temperature=self.temperature,
                system=system_prompt,
                messages=messages
            )
            
            return response.content[0].text
        
        except anthropic.APIError as e:
            raise Exception(f"LLM API Error: {e}")
```

### 4.4 agent.py

```python
import logging
from src.email_client import EmailClient
from src.llm_client import LLMClient

class Agent:
    def __init__(self, config):
        self.name = config['name']
        self.system_prompt = config['system_prompt']
        self.email_client = EmailClient(config)
        self.llm_client = LLMClient(config['api_key'], config)
        self.conversation_history = {}  # thread_id -> messages
        
        self.logger = logging.getLogger(f"Agent.{self.name}")
    
    def process_emails(self):
        """メールを処理"""
        self.logger.info(f"Checking emails for {self.name}")
        
        emails = self.email_client.fetch_unread_emails()
        self.logger.info(f"Found {len(emails)} unread emails")
        
        for email in emails:
            try:
                self._process_single_email(email)
            except Exception as e:
                self.logger.error(f"Error processing email: {e}")
    
    def _process_single_email(self, email):
        """1通のメールを処理"""
        self.logger.info(f"Processing email from {email['from']}")
        
        # スレッド履歴を取得
        thread_id = email['thread_id']
        if thread_id not in self.conversation_history:
            self.conversation_history[thread_id] = []
        
        # メッセージを追加
        self.conversation_history[thread_id].append({
            'role': 'user',
            'content': email['body']
        })
        
        # LLM応答生成
        response = self.llm_client.generate_response(
            self.system_prompt,
            self.conversation_history[thread_id]
        )
        
        # 履歴に追加
        self.conversation_history[thread_id].append({
            'role': 'assistant',
            'content': response
        })
        
        # 返信送信
        self.email_client.send_reply(
            to_email=email['from'],
            subject=email['subject'],
            body=response,
            in_reply_to=email['message_id']
        )
        
        # 既読にする
        self.email_client.mark_as_read(email['id'])
        
        self.logger.info(f"Replied to {email['from']}")
```

## 5. requirements.txt

```
anthropic>=0.18.0
PyYAML>=6.0
APScheduler>=3.10.0
python-dotenv>=1.0.0
```

## 6. config.yaml (ローカル環境用)

### 6.1 MailHog使用時

```yaml
# MailHog: 開発用軽量メールサーバー
# インストール: brew install mailhog (Mac) / apt install mailhog (Ubuntu)
# 起動: mailhog
# Web UI: http://localhost:8025

mail_backend: "mailhog"  # mailhog | imap | maildev

agents:
  - name: "General Assistant"
    email: "llm@localhost"
    smtp_server: "localhost"
    smtp_port: 1025
    api_url: "http://localhost:8025/api/v2"  # MailHog API
    api_key: "${ANTHROPIC_API_KEY}"
    model: "claude-sonnet-4-5-20250929"
    system_prompt: |
      あなたは親切で知識豊富なアシスタントです。
      ユーザーの質問に対して、丁寧で分かりやすい回答を提供してください。
    max_tokens: 4096
    temperature: 0.7

  - name: "Code Expert"
    email: "code@localhost"
    smtp_server: "localhost"
    smtp_port: 1025
    api_url: "http://localhost:8025/api/v2"
    api_key: "${ANTHROPIC_API_KEY}"
    model: "claude-sonnet-4-5-20250929"
    system_prompt: |
      あなたはプログラミングの専門家です。
      コードレビュー、デバッグ、最適化の提案を行います。
      回答にはコード例を含めてください。
    max_tokens: 8192
    temperature: 0.3

scheduler:
  check_interval: 300  # 5分
  max_retries: 3
  retry_delay: 60
```

### 6.2 IMAP使用時（Postfix + Dovecot）

```yaml
mail_backend: "imap"

agents:
  - name: "General Assistant"
    email: "llm@localhost"
    smtp_server: "localhost"
    smtp_port: 25
    imap_server: "localhost"
    imap_port: 143
    password: ""  # ローカルは認証不要の場合も
    use_ssl: false
    api_key: "${ANTHROPIC_API_KEY}"
    model: "claude-sonnet-4-5-20250929"
    system_prompt: "あなたは親切なアシスタントです。"
    max_tokens: 4096
    temperature: 0.7

scheduler:
  check_interval: 300
```

### 6.3 超簡易版（ファイルベース）

メールサーバー不要。ファイルシステムで疎結合:

```yaml
mail_backend: "filesystem"

agents:
  - name: "General Assistant"
    email: "llm@localhost"
    inbox_dir: "/tmp/mailbox/llm/inbox"
    outbox_dir: "/tmp/mailbox/llm/outbox"
    api_key: "${ANTHROPIC_API_KEY}"
    model: "claude-sonnet-4-5-20250929"
    system_prompt: "あなたは親切なアシスタントです。"

scheduler:
  check_interval: 300
```

ファイル形式:
```
/tmp/mailbox/llm/inbox/20260129_230000.txt
---
From: user@localhost
To: llm@localhost
Subject: 質問

ここに本文
---
```

## 7. 環境変数 (.env)

```
EMAIL_PASSWORD=your_email_password
ANTHROPIC_API_KEY=your_anthropic_api_key
```

## 8. デプロイメント

### 8.1 開発環境（ローカルメールサーバー）

#### ステップ1: MailHogセットアップ（推奨）

```bash
# Macの場合
brew install mailhog
mailhog

# Ubuntuの場合
sudo apt-get install mailhog
mailhog

# Dockerの場合
docker run -d -p 1025:1025 -p 8025:8025 mailhog/mailhog

# Web UIで確認
# http://localhost:8025
```

#### ステップ2: Pythonエージェントセットアップ

```bash
# 仮想環境作成
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 依存関係インストール
pip install -r requirements.txt

# 環境変数設定
cp .env.example .env
# .envを編集（ANTHROPIC_API_KEYのみ必要）

# 実行
python src/main.py
```

#### ステップ3: メール送信テスト

```bash
# Python経由で送信
python -c "
import smtplib
from email.mime.text import MIMEText

msg = MIMEText('PythonとLLMについて教えて')
msg['Subject'] = '質問'
msg['From'] = 'user@localhost'
msg['To'] = 'llm@localhost'

with smtplib.SMTP('localhost', 1025) as smtp:
    smtp.send_message(msg)
"

# curlで送信
curl -X POST http://localhost:8025/api/v1/messages \
  -H "Content-Type: application/json" \
  -d '{
    "from": "user@localhost",
    "to": "llm@localhost",
    "subject": "質問",
    "text": "PythonとLLMについて教えて"
  }'

# 5分以内に返信が届く（Web UIで確認）
```

#### 超簡易版: メールサーバー不要

```bash
# ファイルベース版を使用
mkdir -p /tmp/mailbox/llm/{inbox,outbox}

# メール送信（ファイル作成）
cat > /tmp/mailbox/llm/inbox/$(date +%Y%m%d_%H%M%S).txt << EOF
From: user@localhost
To: llm@localhost
Subject: 質問

PythonとLLMについて教えて
EOF

# エージェント起動
python src/main.py

# 5分以内に返信ファイルが生成される
ls /tmp/mailbox/llm/outbox/
```

### 8.2 本番環境 (systemd)

```ini
# /etc/systemd/system/email-llm-agent.service
[Unit]
Description=Email LLM Agent Service
After=network.target

[Service]
Type=simple
User=llm-agent
WorkingDirectory=/opt/email-llm-agent
ExecStart=/opt/email-llm-agent/venv/bin/python src/main.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
# サービス有効化
sudo systemctl enable email-llm-agent
sudo systemctl start email-llm-agent
sudo systemctl status email-llm-agent
```

## 9. 監視とメンテナンス

### 9.1 ログモニタリング

```bash
# リアルタイムログ
tail -f logs/agent.log

# エラーログ抽出
grep ERROR logs/agent.log

# 統計
grep "Replied to" logs/agent.log | wc -l
```

### 9.2 メトリクス

- メール処理数
- LLM API呼び出し回数
- 平均応答時間
- エラー率

## 10. セキュリティ考慮事項

### 10.1 認証
- メールパスワードは環境変数で管理
- API keyは暗号化して保存
- 2要素認証を推奨

### 10.2 検証
- 送信元メールアドレスの検証
- スパムフィルタリング
- レート制限実装

### 10.3 データ保護
- メール内容の暗号化
- ログから機密情報を除外
- 定期的なログローテーション

## 11. 拡張機能

### 11.1 添付ファイル処理
- 画像解析（Claude Vision）
- PDF解析
- コードファイル解析

### 11.2 ツール統合
- Web検索
- カレンダー連携
- タスク管理システム連携

### 11.3 マルチモーダル
- 音声メッセージ変換
- 画像生成
- グラフ・図表生成

## 12. パフォーマンス最適化

- メール取得のバッチ処理
- LLMレスポンスのキャッシング
- 非同期処理の導入
- データベースインデックス最適化

## 13. テスト戦略

```python
# tests/test_agent.py
import unittest
from src.agent import Agent

class TestAgent(unittest.TestCase):
    def setUp(self):
        self.config = {
            'name': 'Test Agent',
            'email': 'test@localhost',
            'system_prompt': 'Test prompt'
        }
        self.agent = Agent(self.config)
    
    def test_process_email(self):
        # テストコード
        pass
```

---

この設計書に基づいて実装を進めてください。
質問や不明点があればお気軽にお問い合わせください。
