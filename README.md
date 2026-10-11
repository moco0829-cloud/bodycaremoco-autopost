# Instagram 自動投稿（Body Care moco）

Body Care moco（@bodycaremoco_narita）の Instagram に、毎日 **昼12時半・夜19時** に、AIが作った画像と文章を自動投稿する仕組みです。
投稿は、横にスワイプして見るカルーセル形式（表紙・中身1〜3枚・締めの3〜5枚）です。
GitHub Actions（クラウド）で動くので、パソコンの電源が切れていても投稿されます。

Salon de moco（@salondemoco）の仕組み（`moco0829-cloud/instagram-autopost`）をもとに作った、別のリポジトリです。
こちらを変えても Salon de moco には影響しません。

```
GitHub Actions（毎日 昼12時半・夜19時の投稿に向けて起動）
  ├─ Claude が「テーマ・キャプション・各画像の文字・画像の指示文」を作る（brand.md と過去の投稿履歴を参照）
  ├─ OpenAI の画像生成AIが背景画像を作る（1080×1350 の縦長に整える）
  ├─ 同じ背景の上に、表紙・中身（番号付きの手順やチェックリスト）・締めの文字をゴシック体で重ねる
  ├─ 画像をリポジトリに保存（Instagram が読める公開URLにするため）
  ├─ Instagram API で投稿
  └─ 履歴を history.json に記録
```

## ファイル

| ファイル | 役割 |
| --- | --- |
| `brand.md` | **アカウントの発信方針。** 投稿の内容を変えたいときはここを書き換える |
| `autopost.py` | 文章・画像の生成と Instagram への投稿 |
| `history.json` | 投稿履歴（テーマの重複を避けるのに使う） |
| `posts/` | 生成した画像の保存先 |
| `.github/workflows/autopost.yml` | 毎日 昼12時半・夜19時の自動実行 |
| `.github/workflows/refresh-token.yml` | Instagram トークンの自動延長（週1回） |

## 色合い

青みの白（`#F3F8FD`）・ネイビー（`#1B2A4A`）・ブルー（`#2E6DB4`）に、太陽の黄色の飾り。最初の9投稿の画像に合わせています。
昼は明るい日差しと青空、夜は夕暮れ〜夜の青とやわらかな明かりの背景になります（`autopost.py` の `PALETTES`）。

## 費用の目安（1日2投稿・月60投稿）

| 項目 | 目安 |
| --- | --- |
| Claude API（文章） | 月 数十円程度 |
| OpenAI 画像生成（medium 品質） | 月 400〜600円程度 |
| GitHub Actions | 公開リポジトリなら無料 |
| Instagram API | 無料 |

Claude と OpenAI のキーは Salon de moco と同じものを使っているので、請求は Salon de moco の分とまとめて来ます。

---

## セットアップ手順

### 1. Instagram をプロアカウントに切り替える（無料）

1. Instagram アプリで @bodycaremoco_narita に切り替える →「プロフィール」→ 右上の「≡」→「アカウントの種類とツール」
2. 「プロアカウントに切り替える」→ カテゴリを選ぶ →「ビジネス」を選択

### 2. Meta のアプリでアクセストークンを取得する

1. [Meta for Developers](https://developers.facebook.com/) にログインし、「マイアプリ」を開く
2. 「アプリを作成」→ ユースケースで **「Instagram でメッセージやコンテンツを管理」** を選ぶ
3. アプリの「Instagram」→「Instagramログインによる API 設定」を開く
4. 「アクセストークンを生成」で @bodycaremoco_narita を追加し、ログインして許可する
   - 権限に `instagram_business_basic` と `instagram_business_content_publish` が含まれていることを確認
5. 表示された **アクセストークン**（長期トークン・60日有効）を控える
6. ブラウザで次のURLを開き、表示される `user_id` を控える（`トークン` の部分を置き換え）
   ```
   https://graph.instagram.com/v23.0/me?fields=user_id,username&access_token=トークン
   ```

### 3. GitHub に公開リポジトリを作って、このフォルダを置く

Instagram API はインターネット上で公開されている画像URLしか読めないため、**公開（Public）リポジトリ** にします。

### 4. GitHub の Secrets を登録する

リポジトリの「Settings」→「Secrets and variables」→「Actions」→「New repository secret」で次を登録します。

| 名前 | 中身 |
| --- | --- |
| `ANTHROPIC_API_KEY` | Claude の API キー（Salon de moco と同じでよい） |
| `OPENAI_API_KEY` | OpenAI の API キー（Salon de moco と同じでよい） |
| `IG_ACCESS_TOKEN` | 手順2-5 のアクセストークン（@bodycaremoco_narita のもの） |
| `IG_USER_ID` | 手順2-6 の `user_id` |
| `GH_PAT` | トークン自動延長用。下記参照 |

**`GH_PAT` の作り方**: [新しいトークンの作成画面（classic）](https://github.com/settings/tokens/new) を開き、
Note に `bodycaremoco-autopost-secrets`、Expiration に「No expiration」、Select scopes で「**repo**」にチェックを入れて作成します。
これがないと、Instagram のトークンが60日で切れて投稿が止まります。
（Fine-grained token では Secrets の許可を付けても書き込めないことがあったため、classic を使います。）

### 5. 試しに動かす

1. リポジトリの「Actions」タブ →「Instagram 自動投稿」→「Run workflow」
2. まずは **「dry_run」にチェック** して実行 → `posts/` に画像、`posts/latest.json` に文章ができるので確認
3. 内容に問題なければ、チェックを外して実行 → Instagram に実際に投稿されます

### 6. 自動投稿を始める

**最初の9投稿（`../files/` の画像）を手で投稿し終えてから** オンにします。

1. リポジトリの「Settings」→「Secrets and variables」→「Actions」→「**Variables**」タブ
2. 「New repository variable」で、名前 `AUTOPOST_ENABLED`、値 `true` を登録

以降は毎日 12:30 と 19:00 に自動で投稿されます。止めたいときは、この値を `false` にします。

---

## よくある調整

- **投稿内容の方向性を変えたい** → `brand.md` を書き換えて push
- **投稿時刻を変えたい** → `.github/workflows/autopost.yml` の `cron`（UTC表記。日本時間から9時間引く）と、`post_at, give_up` の行
- **画像の色を変えたい** → `autopost.py` の `PALETTES`
- **画像下のサロン名を変えたい** → workflow の生成ステップの `env` に `IMAGE_SIGNATURE` / `IMAGE_SIGNATURE_SUB` を追加
- **画像の品質・料金を変えたい** → workflow の生成ステップの `env` に `IMAGE_QUALITY: low` などを追加
- **Facebook ページにも投稿したくなった** → Salon de moco の `autopost.yml` にある「Facebook ページにも投稿」のステップを戻し、Secrets に `FB_PAGE_ID` と `FB_PAGE_TOKEN` を登録（プログラム側の機能は残してある）
- **一時停止したい** → Variables の `AUTOPOST_ENABLED` を `false` にする

## 注意点

- **このリポジトリは Public（公開）のままにしてください。** 非公開にすると次の2つの理由で投稿が止まります。
  1. Instagram は、ここに置いた画像を公開URL（raw.githubusercontent.com）から取りに来るため、非公開だと画像を取得できない
  2. GitHub Actions は公開リポジトリなら無料で使い放題だが、非公開だと月2,000分まで。投稿時刻までの待ち時間を含めて1日約300分使うので、すぐ上限に達する
  公開されるのはプログラム・brand.md・投稿の履歴・投稿した画像だけで、トークンやAPIキーは Secrets に入っていて外からは見えません。

- GitHub Actions の定期実行は混雑時に **数時間遅れる** ことがあります。そのため昼の枠は 9:23・10:23・11:07、夜の枠は 16:23・17:23・18:07 の3回ずつ仕掛けておき、
  最初に動いた回が画像と文章を作って **投稿時刻（12:30 / 19:00）まで待ってから** 投稿します。その枠の投稿が済んでいれば残りの回は何もせず、
  昼は14時、夜は21時を過ぎても始まらなかった日はその枠の投稿を見送ります（時間外に投稿しないため）。
- 毎回の投稿で Actions が画像と履歴をコミットするので、手元で変更するときは GitHub 側を pull してから編集・push してください。
- 投稿は API 経由で毎回自動公開されます。AIが作った内容がそのまま出るため、最初の数日は投稿をチェックし、
  気になる点があれば `brand.md` の「投稿しないこと」に追記してください。
