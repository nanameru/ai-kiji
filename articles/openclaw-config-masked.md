---
title: "OpenClaw設定ファイルを安全に共有する方法（マスク済み）"
emoji: "🦞"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["openclaw", "security", "config", "discord", "ai"]
published: false
---

## この記事の目的

OpenClaw の設定内容をチーム共有したいときに、**機密情報を漏らさず**に共有するための実践例です。
加えて、実運用で迷いやすい

- 全体ファイル構造（どこに何があるか）
- `SOUL.md` など特性ファイルを入れる理由
- それらが実行時にどう効くか

をまとめます。

対象ファイル例:

- `C:\Users\<user>\.openclaw\openclaw.json`

---

## 先に結論

設定ファイルには以下のような機密が含まれます。

- Discord Bot Token
- Gateway Token
- APIキー類
- 個人識別につながるID

そのため、外部共有時は必ず **マスク済みJSON** を使います。

---

## マスク済み設定サンプル

以下は実運用で使える「共有用テンプレート」です（値は伏せ済み）。

```json
{
  "meta": {
    "lastTouchedVersion": "2026.3.2"
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "openai-codex/gpt-5.3-codex"
      },
      "workspace": "C:\\Users\\<user>\\.openclaw\\workspace",
      "compaction": {
        "mode": "safeguard"
      }
    }
  },
  "tools": {
    "profile": "full",
    "exec": {
      "host": "gateway",
      "security": "full",
      "ask": "off"
    },
    "web": {
      "search": { "enabled": true },
      "fetch": { "enabled": true }
    }
  },
  "approvals": {
    "exec": {
      "enabled": false
    }
  },
  "channels": {
    "discord": {
      "enabled": true,
      "token": "***REDACTED***",
      "groupPolicy": "allowlist",
      "allowFrom": [
        "discord:<ALLOWED_USER_ID>"
      ],
      "guilds": {
        "<GUILD_ID>": {
          "requireMention": false
        }
      }
    }
  },
  "gateway": {
    "mode": "local",
    "auth": {
      "mode": "token",
      "token": "***REDACTED***"
    }
  },
  "plugins": {
    "entries": {
      "discord": { "enabled": true },
      "acpx": { "enabled": true }
    }
  }
}
```

---

## 共有前チェックリスト

- `token`, `api_key`, `secret` をすべて置換したか
- Discord/Slack等の実IDを `<PLACEHOLDER>` 化したか
- ローカルパスに個人名が入っていないか
- `published: false` の下書き状態で確認したか

---

## 補足

- 共有資料には「運用方針（承認ON/OFF、allowlist、権限境界）」を残し、
  **認証値そのものは残さない**のが安全です。
- 社外共有なら、最小構成だけ抜き出して別ファイルにするのがおすすめです。

---

## 全体ファイル構造（実運用で見やすい形）

以下は、`C:\Users\<user>\.openclaw\workspace` を「役割」で整理した構造です。

```text
workspace/
├─ AGENTS.md              # 運用ルール（毎セッションの動き方）
├─ SOUL.md                # エージェント人格・口調・境界
├─ USER.md                # ユーザー情報（呼称・プロジェクト文脈）
├─ MEMORY.md              # 長期記憶（主にメインセッション）
├─ memory/
│  └─ YYYY-MM-DD.md       # 日次ログ（短期記憶）
├─ HEARTBEAT.md           # 定期チェックの指示
├─ TOOLS.md               # ローカル環境メモ
├─ skills/
│  ├─ startups-rip-pathfinder/
│  └─ gemini-video-understanding/
├─ articles/              # Zenn原稿
├─ app, lib, components/  # 開発中プロダクト本体
└─ .env.local             # 機密値（絶対に外部共有しない）
```

---

## 特性ファイルを入れる理由と動作

### 1) `SOUL.md`（質問の `sol.md` は `SOUL.md` のこと）

**入れる理由**
- エージェントの口調・判断軸・境界を固定するため
- セッションをまたいでも「同じキャラ」で振る舞わせるため

**どう動作するか**
- セッション開始時に読み込まれ、返信トーン・優先順位に影響
- 例: 「端的に」「媚びない」「実務優先」などが出力方針になる

### 2) `AGENTS.md`

**入れる理由**
- 毎回の運用手順を機械的に再現するため
- 読む順序（SOUL/USER/memory）や安全ルールを固定するため

**どう動作するか**
- 起動時の行動テンプレートとして効く
- グループチャット時の発言抑制や外部送信時の確認ルールを適用

### 3) `USER.md`

**入れる理由**
- ユーザー固有の文脈（プロジェクト・呼称）を維持するため

**どう動作するか**
- 提案内容の優先度付け（例: Kirigami関連）に反映
- 呼び方や回答の方向性が安定

### 4) `MEMORY.md` と `memory/YYYY-MM-DD.md`

**入れる理由**
- 再起動で失われる文脈をファイルとして永続化するため

**どう動作するか**
- 日次メモから重要事項を長期記憶へ昇格
- 次回セッションで過去経緯を再利用

### 5) `skills/` 配下

**入れる理由**
- 何度も繰り返す手順を再利用可能にするため

**どう動作するか**
- ユーザー要求に一致する skill の `description` でトリガー
- `SKILL.md` 本文がロードされ、作業フローが強化される

**今回の例**
- `startups-rip-pathfinder`:
  - 企画前に類似失敗事例10件以上を調査し、失敗回避策を設計
- `gemini-video-understanding`:
  - 動画の転写/分割/Embedding/RAG回答の手順を標準化

---

## 共有時の推奨テンプレ（運用）

- 構造図は共有してOK
- `SOUL.md` / `AGENTS.md` の方針も共有してOK
- `openclaw.json` は**必ずマスク版のみ共有**
- `.env.local` / token / key は共有禁止

---

## まとめ

OpenClaw の運用は、

1. **構造を役割で分ける**（設定・記憶・スキル・実装）
2. **人格/運用ルールを明文化する**（SOUL/AGENTS）
3. **機密値は完全マスク**して共有する

この3点で、チームでも安全に再現できます。
