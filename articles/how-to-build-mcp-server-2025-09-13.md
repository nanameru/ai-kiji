---
title: "MCPの作り方：最小サーバーから実運用まで"
emoji: "🛠️"
type: "tech"
topics: ["mcp", "model-context-protocol", "nodejs", "typescript", "server"]
published: false
---

## 概要
- Model Context Protocol (MCP) サーバーをゼロから実装する手順を、最小構成→拡張→運用の順で解説します。
- まずはローカルで動く最小サーバーを作り、ツール定義・バリデーション・ログ設計・配布方法までを俯瞰します。

## 背景・前提
- 対象: LLM から外部システム/API/DB を安全に呼び出すためのインターフェースを実装したい開発者
- 前提: Node.js 18+ / TypeScript の基礎、npm/pnpm を扱えること
- 参考: 仕様サイト（Model Context Protocol）: https://modelcontextprotocol.io/

## 最小構成のMCPサーバー
以下は HTTP で `ping` ツールを1個だけ提供する最小サンプルです。

```bash
mkdir mcp-hello && cd mcp-hello
npm init -y
npm i express zod
npm i -D typescript ts-node @types/node @types/express
npx tsc --init
```

`src/server.ts` を作成：

```ts
import express from 'express';
import { z } from 'zod';

const app = express();
app.use(express.json());

// ツール: ping（入力: text、出力: text）
const PingInput = z.object({ text: z.string().min(1) });

app.post('/mcp/tools/ping', (req, res) => {
  const parsed = PingInput.safeParse(req.body?.params);
  if (!parsed.success) {
    return res.status(400).json({ error: 'Invalid params', issues: parsed.error.issues });
  }
  const { text } = parsed.data;
  return res.json({ result: { text: `pong: ${text}` } });
});

app.get('/mcp/manifest', (_req, res) => {
  res.json({
    name: 'mcp-hello',
    version: '0.1.0',
    tools: [
      {
        name: 'ping',
        description: 'Echo ping result',
        inputSchema: {
          type: 'object',
          properties: { text: { type: 'string' } },
          required: ['text']
        }
      }
    ]
  });
});

const port = process.env.PORT || 4000;
app.listen(port, () => console.log(`[MCP] listening on http://localhost:${port}`));
```

実行：
```bash
npx ts-node src/server.ts
```

疎通テスト：
```bash
curl -s http://localhost:4000/mcp/manifest | jq '.name,.tools[0].name'
curl -s -X POST http://localhost:4000/mcp/tools/ping -H 'Content-Type: application/json' -d '{"params":{"text":"hello"}}'
```

## ツール設計のポイント（安全・一貫性）
- 入力スキーマ: zod 等で厳格にバリデーション。エラーは機械可読に返す
- 出力フォーマット: 可能な限り安定した JSON 形; LLM が扱いやすい簡潔さに
- 冪等性: 同じ入力は同じ出力。副作用がある場合は明示のフラグを要求
- タイムアウト/リトライ: 外部 API 呼び出しには上限・再試行戦略を定義
- レート制限/認可: API キー必須化、IP/トークン単位のレート制御

## Manifest（メタデータ）の拡張
- `description` にユースケース例を含めると LLM が選択しやすい
- `inputSchema.examples` 等があれば積極的に提示
- 互換性: 将来の変更は後方互換を意識し、new tool 追加で拡張

## ロギングと観測
- 収集: リクエストID・ツール名・入力サマリ・処理時間・結果サイズ
- 失敗時: 例外種別・外部APIの応答・リトライ回数を保存
- 可視化: ローカル開発ではコンソール、運用では構造化ログ + 集約

```ts
const start = Date.now();
try {
  // ...handler
} finally {
  const ms = Date.now() - start;
  console.log(JSON.stringify({ tool: 'ping', ms }));
}
```

## 実運用に向けた拡張
- 認証: Bearer トークン/署名ベース。ツールごとに権限スコープ
- 監査: 重要操作（書き込み/削除）は監査ログに記録
- スキーマ進化: バージョンフィールドを持たせ段階的移行を可能に
- デプロイ: Cloud Run/Fly.io/Vercel Functions などの無停止更新

## クライアント（MCP対応エージェント）からの利用
- Manfiest URL を登録し、`tools.ping` などを直接呼び出す
- 「契約としてのプロンプト」を併用（入力・制約・期待出力の明文化）

## よくある落とし穴
- 曖昧な出力: LLM が解釈に失敗 → JSON 形式を固定し、テキストは補助
- 入力の過不足: 必須/任意を明確化し、バリデーションメッセージを丁寧に
- 無限リトライ: 上限とヒューズ（サーキットブレーカー）を導入

## 参考・出典
- MCP 公式: https://modelcontextprotocol.io/
- Express: https://expressjs.com/
- Zod: https://zod.dev/

## まとめ・次アクション
- 最小サーバーを写経→`ping` をツール化→ログとバリデーションを追加
- 次に運用要件（認証・監査・レート制限）を段階的に導入
- 新規ツールは後方互換を保ちつつ追加し、Manifest を育てましょう。
