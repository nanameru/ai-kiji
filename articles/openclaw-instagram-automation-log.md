---
title: "OpenClawでInstagram自動投稿基盤を作るまでの実戦ログ"
emoji: "🐙"
type: "idea"
topics: ["instagram", "openclaw", "automation", "sora", "api"]
published: false
---

## この記事について

この下書きは、Discord上での実運用ログを元に、

- Instagram API連携
- OpenAI動画生成（Sora系）
- 自動投稿
- 投稿後分析ループ（6h/12h/24h）

を実際に組んだ流れをまとめたものです。

---

## 目的

「動物系ショート動画を自動生成してInstagramへ投稿し、投稿後の分析を回して改善する」運用を作る。

---

## 実装したこと（要点）

### 1. Instagram投稿の自動化スクリプト作成

`~/.agents/skills/instagram-auto-post/` に以下を作成。

- `scripts/post_image.py`
- `scripts/post_reel.py`
- `scripts/oauth_helper.py`
- `scripts/refresh_token.py`

ポイント:

- `container -> status FINISHED -> media_publish` の正攻法フロー
- キャプションの改行崩れ対策（`\\n` を実改行へ変換）

### 2. Meta OAuth連携

- App ID / App Secret を設定
- OAuth Redirect URI設定
- 認可コード交換 -> long-lived token 取得
- `IG_ACCESS_TOKEN` / `IG_USER_ID` を環境変数化

### 3. 初回投稿（Reel）

- Sora系で「ドジ猫」動画を生成
- 公開URL化
- Instagram Reels 投稿

### 4. 分析ループの自動化

cronジョブで以下を設定。

- 2時間ごと分析
- 日次レビュー
- 投稿後6h/12h/24hの節目評価

分析観点:

- Views
- 平均視聴時間
- 完視聴率
- Sends per reach
- Saves per reach

---

## 実運用でハマったところ

### UI上は接続済みなのに APIで `instagram_business_account` が返らない

原因:

- ページ連携の最終反映待ち
- 古いトークンで確認していた
- 個人アカ連携とページ連携の混同

対処:

- ページ連携を再確認
- 再認可してトークン再発行
- 数分待って再確認

### キャプションの `/n` 問題

原因:

- 改行を文字列として渡していた

対処:

- 投稿スクリプト側で `\\n -> \n` 変換

---

## アルゴリズム対策（調査結果の要約）

海外・国内の公開情報を照合すると、短期で効くのは以下。

1. 視聴維持（watch time / 完視聴）
2. シェア（sends）
3. 反応率（likes per reach）

加えて必須条件:

- オリジナル性
- 他SNS透かしなし
- 低品質/転載っぽい素材回避

---

## 次のアクション

- 投稿テンプレ10本を作成（冒頭3秒フック固定）
- 週次で「勝ちパターン」更新
- 失敗投稿の共通要因をDBで管理

---

## 画像について

この下書きに、Discordで取得した実際の設定スクショを差し込む予定です。

例:

- Meta開発者設定画面
- Instagram接続確認画面
- 初回投稿の表示画面

> 画像は `images/` 配下に置いて、本文に埋め込みます。

---

## 参考リンク

- Instagram Creators: Recommendations and Originality
- Instagram Help: Protecting original content
- Meta Graph API Docs

（公開時に正式URLを整備）
