---
title: "OpenClawでInstagram自動投稿ループを実運用したログ（失敗込み）"
emoji: "🐙"
type: "idea"
topics: ["instagram", "openclaw", "automation", "sora", "cron"]
published: true
---

## この記事について

初めまして！Taiyoです！

この記事は、Instagram自動投稿を**実運用で回しながら改善した記録**です。
「作って終わり」ではなく、実際に失敗して、直して、再発防止までやった内容を、できるだけわかりやすく共有します。

今回やったことは、次の4つです。

- Sora動画生成
- Instagram自動投稿
- 6h/12h/24h分析
- 障害時の切り分けと修正

読むとわかることはシンプルです。

- どの順番で組めば安定するか
- どこで失敗しやすいか
- 失敗した時に何を直せばいいか

---

## 前段階: Facebookページ作成〜Instagram API接続（ここを飛ばすと詰まる）

ここが実は一番ハマりやすいので、最初に手順を固定します。

### Step 0. 先に理解しておくこと

Instagram投稿APIは、**個人アカウント連携だけでは動きません**。
必要なのは次の3点です。

1. Instagramがプロアカウント（ビジネス/クリエイター）
2. FacebookページとInstagramが正しく紐づいている
3. Meta Appで必要権限を取ったトークンがある

この3つが1つでも欠けると、`me/accounts` で `instagram_business_account` が返らず、投稿が通りません。

### Step 1. Facebookページを作る

- URL: `https://www.facebook.com/pages/create`
- とりあえず仮ページ名でもOK（後で変更可能）

### Step 2. Instagram側をプロアカウント化

Instagramアプリで以下を確認:

- プロフィール → 設定とプライバシー → アカウントタイプ
- ビジネス（またはクリエイター）になっていること

### Step 3. FacebookページとInstagramをリンク

最短は以下のどちらか:

- Facebookページ設定 → リンク済みアカウント → Instagram
- Meta Business settings → アカウント → Instagramアカウント

重要なのは「接続済み表示」だけでなく、**対象ページにアセット割り当て済み**であること。

### Step 4. Meta App側のOAuth設定

- App ID / App Secret を取得
- Valid OAuth Redirect URI を設定（例: `https://localhost/callback`）
- 必要権限を付与
  - `instagram_basic`
  - `instagram_content_publish`
  - `pages_show_list`
  - `pages_read_engagement`

### Step 5. 認可コード交換で長期トークン取得

- 認可URLで同意 → `code` 取得
- `code` を交換して long-lived token を発行
- `IG_ACCESS_TOKEN` として保存

### Step 6. `IG_USER_ID` を取得して固定

Graph APIで以下を確認し、対象の `instagram_business_account.id` を取得:

- `/me/accounts?fields=id,name,instagram_business_account`

取得したIDを `IG_USER_ID` として保存。

### Step 7. ここまで終わったら初回投稿テスト

- 画像/動画URLを使って `container -> FINISHED -> publish` の順で投稿
- 投稿URLが返れば接続成功

### ここで実際に起きた詰まりポイント

- UI上は接続済みだが API に反映されない
- `business.facebook.com/out_of_scope_redirect`
- 古いトークンで確認していて接続状態が更新されない

対策は「再同意」「ページ割当確認」「トークン再発行」の3点セットでした。

---

## 先に結論（運用が安定した形）

まず結論です。安定したのは、次の5ステップを分離した構成でした。

1. **トレンド調査**（FireClaw + Virlo参照）
2. **Start**（新規生成。ただし未投稿completedがあれば再利用して新規生成スキップ）
3. **Finalize**（完了動画の投稿、重複防止）
4. **6h/12h/24h分析**（必要に応じて48h追補）
5. **次回最適化**（プロンプト・投稿時刻の調整）

ポイントは、生成と投稿を1ジョブにまとめないことです。ここを分けるだけで、タイムアウト事故と重複投稿がかなり減りました。

---

## 固定運用ルール（実装反映済み）

ここは「守るだけで事故が減る」重要ルールです。

- 2段階構成（Start/Finalize分離）
- 1日1投稿ガード（JST）
- 完了ジョブ複数時は最新1件のみ投稿、残りskipped
- 投稿前照合（`prompt_hash / video_id / source_url`）
- 投稿成功時はログより先に状態保存
- 画像・動画の再利用は**Instagram未投稿のみ**

特に、**状態保存より先に成功ログを出さない**のが大事です。ここを逆にすると「投稿できてないのに成功に見える」事故が起きます。

---

## 実際にハマったポイントと対処

### 1) Meta権限まわり（`out_of_scope_redirect` / Explorer入れない）

**現象**
- Graph API Explorerに入れない
- `business.facebook.com/out_of_scope_redirect` に飛ぶ

**原因**
- Business文脈で開いていて、開発者ツールのコンテキストと衝突

**対処**
- 個人プロフィール文脈で `developers.facebook.com` を開き直し
- 権限同意をやり直し

![Meta権限まわりの実画面](/images/openclaw-instagram-automation/71b1ccf1-f37a-43f4-a51a-640c95813429.png)

---

### 2) `me/accounts` が空 / `instagram_business_account` が返らない

**現象**
- `/me/accounts?fields=id,name,instagram_business_account` が空またはIG情報なし

**原因**
- ページ選択ミス（OpenClawページではない）
- 認可ダイアログで対象ページの許可漏れ

**対処**
- ページ選択を明示して再同意
- `/{page_id}?fields=instagram_business_account{id,username}` で最終確認

![ページ/権限選択の実画面](/images/openclaw-instagram-automation/5059ed6f-35a3-4631-9929-02a8d021c10a.png)

---

### 3) OpenAI動画生成が 400 で失敗

**現象**
- `POST /v1/videos` が 400

**原因（最終）**
- `billing_hard_limit_reached`

**対処**
- 課金上限・支払い設定の確認
- さらに、Start前に既存completed未投稿を拾って**無駄生成をスキップ**する実装に変更

![運用ログでの失敗確認画面](/images/openclaw-instagram-automation/99d26f33-fe82-4a7a-8bd3-2bb06bb8aa78.png)

---

### 4) 投稿ログの出し先がブレる

**現象**
- 固定スレと運用ログスレが混在

**対処**
- ログはフォーラム下の運用ログスレへ集約
- さらに要望に合わせて「投稿ごとに新規ログスレッド作成」方針へ更新

![運用ログスレッド管理画面](/images/openclaw-instagram-automation/c9a5f34d-16c0-407a-82d3-3266ca404430.png)

---

## Discord会話ベースの実録タイムライン（何で詰まったか）

ここは実際の会話ログをもとに、詰まり→解決の順番を時系列でまとめます。

最初の大きな詰まりは、UIではInstagramが接続済みに見えるのに、APIでは `instagram_business_account` が返らない状態でした。会話上では何度も「接続済みなのに取れない」という往復になりましたが、最終的には「個人アカウントのリンク」と「FacebookページとInstagramの業務連携」を混同していたのが主因でした。ページへのアセット割り当てと再認可を行い、長期トークンを再発行して解決しました。

![接続済みなのにAPIが返らない時の確認画面](/images/openclaw-instagram-automation/149fb5f9-dae2-43c7-b3c6-8749f97dd0cc.png)

次の詰まりは、投稿自体は成功しているのに、実行ジョブ側が文字コードエラーで落ちて「未投稿扱い」のまま再実行され、重複投稿が連鎖したことです。これは実際に運用中に発生し、同系リールが短時間で複数本出る事故につながりました。

![重複投稿が発生した時の運用ログ画面](/images/openclaw-instagram-automation/8062c8bd-db69-40c9-8084-08092e44d160.png)

この事故で得た教訓は、"投稿成功ログを出す前に状態保存を完了する" こと、そして "Finalizeで投稿対象を常に最新1件に限定する" ことです。実装上もこの順序に修正し、古いcompletedジョブは `skipped` に落とすように変えました。

さらに、Sora生成時間が長い時にタイムアウトする問題があったため、生成開始（Start）と投稿（Finalize）を分離しました。これにより、1回の長いジョブで全工程を抱えない構造にでき、復旧も容易になりました。

![Start/Finalize分離後の運用構成](/images/openclaw-instagram-automation/dfe1cf31-043c-42da-8921-1051b4100f26.png)

---

## インスタAPI取得手順（実運用で使った“詰まらない版”）

ここからは、箇条書きだけでなく、実際に通った順番で説明します。

まず、Meta Appを作る前にInstagram側をプロアカウント（ビジネスまたはクリエイター）にしておきます。ここが個人アカウントのままだと、後の権限同意が通っても投稿APIが使えません。

次にFacebookページを作ります。ページ名は仮でも問題ありませんが、このページが後で `IG_USER_ID` を引く時の起点になるので、どのページに紐づけるかを最初に決めておくと混乱しません。

その後、Meta for DevelopersのApp DashboardでInstagram API with Facebook Login（またはBusiness Login for Instagram）の設定に入り、OAuth Redirect URIを追加します。会話中に最も多かったエラーは、URIの不一致とコンテキスト不一致（Business文脈）でした。`https://localhost/callback` を設定し、同じ値で認可URLを組むのが安全でした。

認可同意が完了したら `code` を受け取り、短期トークン→長期トークンへ交換します。ここで長期トークンを保存し、次に `/me/accounts` でページ一覧と `instagram_business_account` を確認します。もし返らない場合は、ページ側のリンク済みアカウント設定とアセット割り当てを再確認し、トークンを再発行すると通るケースが多いです。

最後に取得した `instagram_business_account.id` を `IG_USER_ID` として固定し、`container -> FINISHED -> publish` の順でテスト投稿を実行します。これが通れば、投稿基盤は完成です。

参照（Firecrawlで確認した公式ドキュメント）:
- Business Login for Instagram: https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/business-login/
- Instagram API with Facebook Login: https://developers.facebook.com/docs/instagram-platform/instagram-api-with-facebook-login/business-login-for-instagram/

---

## 反映したコード改善

ここからは「どう直したか」を、実装ベースで説明します。

### Start改善（無駄生成防止）

`ig_pipeline_start.py` を更新し、開始時に `GET /v1/videos` 相当で

- completed
- かつ未投稿

があれば新規生成をスキップして再利用するように変更。

これで課金事故と重複生成を減らせます。

![Start周りの運用ログ](/images/openclaw-instagram-automation/b86e1b68-e6bf-4ef1-8922-0d400e498dc2.png)

### Finalize改善（重複/状態不整合防止）

- 投稿済みは除外
- 投稿候補は最新1件
- その他を `skipped`
- 状態保存をログより先に実行

![Finalize周りの運用ログ](/images/openclaw-instagram-automation/c9cb246c-c2ad-4c80-997f-4f745a121b6e.png)
---

## Cron構成（最終イメージ）

- Research（毎日）
- Start（22:30 JST）
- Finalize（15分ごと）
- KPI分析（15分判定で6h/12h/24h通知）
- Optimizer（日次）

![Cron/運用スレッドの実画面](/images/openclaw-instagram-automation/8c15859f-6cea-4676-9b65-b4d682cf5ab8.png)

---

## セキュリティで学んだこと

実運用中に以下が発生しました。

- トークン/シークレットをチャットへ貼ってしまう

そのため、運用では必ず以下を徹底すべきです。

- 漏れたトークンは即失効
- App Secret再生成
- 長期トークン再発行
- `.secrets/instagram.env` でのみ管理

---

---

## まとめ

最後に、これから同じ仕組みを作る人向けに一言でまとめます。

この運用で一番大事なのは、機能よりも順序でした。

- 先に認証と権限を固める
- 生成と投稿を分離する
- 状態保存を先にやる
- ログ出し先を固定する
- 例外時の挙動を先に決める

ここまで固めると、Instagram自動投稿は「動く」から「回る」に変わります。

もし今から着手するなら、まずは Start / Finalize の2段階分離だけでも先に入れておくのがおすすめです。

