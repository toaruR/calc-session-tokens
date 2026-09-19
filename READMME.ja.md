# calc-session-tokens

AI コーディングエージェント（**Google Antigravity / AGY**、**OpenAI Codex**、**Claude Code**）のセッションにおけるトークン消費量を正確に計測・集計・可視化するための、ゼロ依存スキルハーネス＆スクリプト集です。

親セッションだけでなく、セッション内で再帰的・階層的に起動されたサブエージェント（Subagents）も自動追跡し、プロンプトキャッシュ（Prompt Caching）、新規入力（Uncached Input）、モデル出力（Output）、思考トークン（Reasoning / Thinking）を適切に分離して集計します。

---

[English](README.md) | [日本語](README.ja.md)

---

## 主な特徴

- **マルチエージェント対応**: **Antigravity (AGY)**、**OpenAI Codex**、**Claude Code** の各ログ形式に対応。
- **チャットから即座に実行**: IDE のチャット欄で `/session-tokens` を入力するだけで集計結果を Markdown テーブル形式で表示。
- **サブエージェントの再帰集計**: 親エージェントから起動された子エージェント群（`invoke_subagent`、`SubAgentActivity`、Agent ツール経由）のトークン消費を自動検出し、階層別に集計。
- **高精度なキャッシュ・請求トークン算出**:
  - キャッシュから読み出されたトークン（割引・無料枠対象）と新規入力トークンを区別。
  - 思考トークン（Reasoning / Thinking tokens）を出力トークンの内数として正確に扱い、二重加算を防止。
  - プロバイダー横断で費用感を統一比較できる指標として **実課金対象トークン合計（Billed Total: `Uncached Input + Output`）** を算出。
- **外部依存ゼロ (Zero Dependencies)**: Python 標準ライブラリのみで動作（AGY の SQLite 内 protobuf バイナリも独自ワイヤデコーダで解析するため `protobuf` ライブラリのインストール不要）。

---

## 使い方 (Usage)

### 基本的な使い方（チャット欄から実行）

各エディタ / IDE のチャット入力欄でスラッシュコマンドを入力するだけで、エージェントが自動的に現在のセッションと起動されたサブエージェントのトークン消費量を集計して報告します。

| エージェント / IDE | チャットコマンド | 動作内容 |
|---|---|---|
| **Google Antigravity (AGY IDE / CLI)** | `/session-tokens` | 現在アクティブなセッションと子サブエージェントの消費トークンを自動集計 |
| **Claude Code** | `/session-tokens` | 現在の Claude Code セッションおよび Agent ツールで起動したサブエージェントを集計 |
| **OpenAI Codex** | `/codex-session-tokens` | 現在の Codex セッション（スレッド）と子エージェントを集計 |

#### 実行例（チャット欄での表示）

チャット欄でコマンドを送信すると、以下のような集計レポートが出力されます：

```markdown
# AGY Session Token Report
- **Conversation ID**: `a320c2da-27fd-...`
- **Detected Model**: `gemini-3.8-flash`
- **Total LLM Steps**: `14` (Parent: 14, Subagents: 0)

## 📊 Token Usage Summary (総計)
| Metric (指標) | Tokens | 備考 |
|---|---|---|
| **Prompt Cached Tokens** | **145,210** | プロンプトキャッシュ読込 |
| **Prompt Uncached Tokens** | **12,480** | 新規入力トークン |
| **Output Tokens** | **3,120** | 生成（Thinking含む）トークン |
| **Total Billed Tokens** | **15,600** | 実課金対象合計 (Uncached + Output) |
```

---

### コマンドラインからの直接実行（オプション指定）

過去セッションの集計や JSON 形式での出力（自動化・スクリプト連携用）を行いたい場合は、ターミナルから直接スクリプトを実行することも可能です。

#### 1. Google Antigravity (AGY)
```pwsh
# 最新セッションを集計
python .agents/skills/session-tokens/scripts/calc_session_tokens.py

# 特定の会話ID (Conversation ID) を指定
python .agents/skills/session-tokens/scripts/calc_session_tokens.py --conversation-id "<CONVERSATION_ID>"

# JSON 形式で出力
python .agents/skills/session-tokens/scripts/calc_session_tokens.py --json
```

#### 2. Claude Code
```bash
# 最新セッションを集計 ($CLAUDE_CODE_SESSION_ID から自動解決)
python .claude/skills/session-tokens/scripts/calc_session_tokens.py

# 特定のセッションIDを指定
python .claude/skills/session-tokens/scripts/calc_session_tokens.py --session-id "<SESSION_ID>"

# JSON 形式で出力
python .claude/skills/session-tokens/scripts/calc_session_tokens.py --json
```

#### 3. OpenAI Codex
```pwsh
# 現在のセッションまたは最新セッションを集計
python -X utf8 .agents/skills/codex-session-tokens/scripts/calc_codex_session_tokens.py

# 特定のスレッドIDを指定
python -X utf8 .agents/skills/codex-session-tokens/scripts/calc_codex_session_tokens.py --session-id "<THREAD_ID>"

# 親セッションのみ集計（サブエージェントを除外）
python -X utf8 .agents/skills/codex-session-tokens/scripts/calc_codex_session_tokens.py --no-subagents

# JSON 形式で出力
python -X utf8 .agents/skills/codex-session-tokens/scripts/calc_codex_session_tokens.py --json
```

---

## トークン指標の定義

| 指標 (Metric) | 説明 |
|---|---|
| **Cached Input Tokens** | プロンプトキャッシュから読み出された入力トークン（通常は割引または無償枠）。 |
| **Uncached Input Tokens** | キャッシュにヒットせず、新規にモデルへ送信された入力トークン (`Input - Cached`)。 |
| **Output Tokens** | モデルが生成した応答トークン（Thinking / Reasoning トークンを含む）。 |
| **Total Billed Tokens** | 課金計算対象となる有効トークン合計 (`Uncached Input + Output`)。 |
| **All Processed Tokens** | コンテキストウィンドウ全体で処理された総トークン数 (`Input + Output`)。 |

---

## 対応エージェントと実装詳細

| エージェント / ハーネス | スキルディレクトリ | 対象ログ・DB 形式 |
|---|---|---|
| **Google Antigravity (AGY)** | `.agents/skills/session-tokens` | SQLite データベース (`conversations/*.db`) + `transcript.jsonl` |
| **Claude Code** | `.claude/skills/session-tokens` | プロジェクトログ (`~/.claude/projects/*/*.jsonl`) |
| **OpenAI Codex** | `.agents/skills/codex-session-tokens` | ロールアウトログ (`~/.codex/sessions/**/*.jsonl`) |

---

## プロジェクト構成

```
calc-session-tokens/
├── .agents/
│   └── skills/
│       ├── session-tokens/              # Antigravity (AGY) 向けスキル定義
│       │   ├── SKILL.md
│       │   └── scripts/
│       │       └── calc_session_tokens.py
│       └── codex-session-tokens/        # OpenAI Codex 向けスキル定義
│           ├── SKILL.md
│           └── scripts/
│               └── calc_codex_session_tokens.py
├── .claude/
│   └── skills/
│       └── session-tokens/              # Claude Code 向けスキル定義
│           ├── SKILL.md
│           └── scripts/
│               └── calc_session_tokens.py
├── LICENSE                              # MIT License
├── README.md                            # 英語版ドキュメント
└── README.ja.md                         # 日本語版ドキュメント
```

---

## 動作要件

- **Python 3.8+**
- 外部ライブラリのインストール不要（Python 標準ライブラリのみで動作）

---

## ライセンス

本プロジェクトは [MIT License](LICENSE) のもとで公開されています。
