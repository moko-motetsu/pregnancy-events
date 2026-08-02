# マタニティおでかけイベントガイド

現在地から100km圏内のおでかけイベントを一覧化するWebアプリ。カテゴリで絞り込みができ、有料イベントは購入サイトへのリンクを表示。妊娠週数に応じたおすすめイベントも紹介します。

`zukan-seal-collection` と同じく **`index.html` 1ファイルで完結する静的Webアプリ**です。ビルドやサーバーは不要ですが、`events.json` を読み込むため `file://` で直接開くとブラウザのセキュリティ制限で読み込みに失敗します（後述のフォールバックでアプリ自体は起動しますが、サンプルデータのみの表示になります）。ローカルサーバー経由か、GitHub Pages 等で公開して使ってください。

## 家族限定の入場制限について

公開サイトは合言葉によるアクセスゲートで保護しています（`GATE_HASH` にSHA-256ハッシュのみ埋め込み、平文はソースに含めない）。加えて `<meta name="robots" content="noindex, nofollow">` と `robots.txt` で検索エンジンには載らないようにしています。

これはGitHub Pagesの無料枠に本格的な認証機能が無いためのクライアント側の簡易対策であり、ページのソースやネットワーク通信を見れば技術的には突破できます。「不特定多数に公開しない」という意図を示すための軽い障壁であり、強固なセキュリティではありません。合言葉は家族にのみ共有してください。合言葉を変更する場合は、`index.html` 内の `GATE_HASH` を新しい合言葉のSHA-256ハッシュに置き換えてください（例: `python3 -c "import hashlib; print(hashlib.sha256('新しい合言葉'.encode()).hexdigest())"`）。

## 主な機能

- **現在地から100km圏内の絞り込み** — ブラウザの位置情報(Geolocation API)で現在地を取得し、`events.json` に登録した各イベントの緯度経度との距離（Haversine公式）でフィルタ。GPSが使えない場合は設定画面から都道府県を選んで代用可能。
- **カテゴリ絞り込み**（複数選択可）
- **有料イベントの購入サイトへのリンク**
- **妊娠週数に応じたオススメ表示** — 出産予定日を設定すると自動で現在の週数を計算し（`nutrition/pregnancy.py` と同じ「予定日から逆算・40週0日」ロジックをJSで再実装）、各イベントに設定した対象週数レンジに合致するものをおすすめセクションと専用バッジで表示。
- **季節のオススメ表示** — 現在の月から春夏秋冬を判定し、海水浴・ハイキングなど季節に合うイベント（`season`）をおすすめセクションと専用バッジで表示。
- **地点の指定** — GPSでの現在地取得に加え、設定画面から「よく使うスポット」（有明ガーデンなど）をワンタップで選択、または緯度経度を直接入力して任意の地点周辺のイベントを確認可能。現在地にいなくても、行き先候補のスポット周辺を先に調べられる。
- **近くの定期イベント会場** — 有明ガーデンシアター・有明アリーナ・有明GYM-EXのように、コンサートや興行が週替わりで開催される会場をディレクトリとして持つ（`venues`）。個別の公演情報は `pregnancy-events/scripts/fetch_venue_schedules.mjs` が各施設の公式サイトから自動取得し（GitHub Actions `watch_ariake_venues.yml`）、`venue_schedules.json` に保存する。取得できていない施設は公式スケジュールページへの直リンクにフォールバックする。
- **検索** — イベント名・会場名・説明文に加えて、カテゴリ名とタグ（`tags`）も検索対象。
- **花火大会シーズンの案内** — 花火大会は夏の間だけで関東全域で毎週100件近く開催され、手動データで全件を網羅するのは非現実的なため、代表例を数件`events`に掲載しつつ、カテゴリ「祭り・花火」を選択または「花火」で検索すると、全件を検索できる外部の花火大会一覧サイトへの案内バナーを表示する。
- **開催日の自動確認** — `events`内で`officialUrl`を持つ全イベントについて、`pregnancy-events/scripts/fetch_event_confirmations.mjs`が各イベントの公式サイト（自治体・観光協会・実行委員会・施設運営者など一次情報源のみ。まとめサイトは利用規約上の懸念があるため対象外）を確認し、`event_confirmations.json`に確認結果を保存する（GitHub Actions `watch_event_confirmations.yml`）。荒天順延や日程変更があった場合、確認が取れ次第自動で反映される。カテゴリは問わない（当初は花火大会限定だったが全カテゴリに拡張）。
- **公式情報の表示** — 各イベントに情報源(`source`)と最終更新日(`updatedAt`)を明記し、詳細画面に「必ず公式サイトでご確認ください」の注記を表示。
- **口コミ・渋滞回避・お得情報** — イベント詳細モーダルに、口コミ（評点・コメント・出典）、渋滞回避／アクセス情報（車・電車・駐車場・混雑時間帯）、お得情報（割引券など）を表示。

## ローカルで動かす

```bash
cd pregnancy-events
python3 -m http.server 8080
# ブラウザで http://localhost:8080/ を開く
```

`events.json` の取得に失敗した場合（`file://` で直接開いた場合など）は、画面上部に警告バナーが出てサンプルデータ1件のみで動作します。

## テスト

```bash
node --test pregnancy-events/test/*.test.mjs
```

Node標準の `node:test` のみを使用しており、`npm install` は不要です（Node 18以降）。対象は `scripts/` 内のパーサー関数（JSON-LD抽出、各会場の個別パーサー、和暦日付パースなど）と、`index.html` 内の純粋関数（距離計算・日付処理・ICS生成など）です。後者は `index.html` からビルドステップなしで動かす方針（zukan-seal-collectionと同じ規約）を崩さないよう、`test/app-context.mjs` がインラインの `<script>` をNodeの `vm` モジュール上で評価してテスト可能にしています（DOM操作を伴う描画系の関数はテスト対象外）。

`pregnancy-events/**` を変更するPR・pushでは GitHub Actions（`.github/workflows/pregnancy_events_test.yml`）が自動でこのテストを実行します。

## 公開する（GitHub Pages）

`zukan-seal-collection` と同じ構成で、公開用リポジトリ `moko-motetsu/pregnancy-events` へ自動デプロイしています（`https://moko-motetsu.github.io/pregnancy-events/`）。

- `.github/workflows/deploy_pregnancy_events.yml` — `pregnancy-events/**` が `main` に push されたら `index.html`・`README.md`・`events.json`・`venue_schedules.json` を公開用リポジトリへ反映
- `.github/workflows/watch_ariake_venues.yml` — 有明会場スケジュールの日次更新は `[skip ci]` でコミットするため上記の push トリガーが発火しない。そのためこのワークフロー自身が直接デプロイまで行う

どちらも公開用リポジトリへの書き込み権限を持つ Fine-grained PAT を、`test` リポジトリのシークレット **`PREGNANCY_EVENTS_DEPLOY_TOKEN`** として必要とする。未設定の場合、`deploy_pregnancy_events.yml` はエラーで失敗し、`watch_ariake_venues.yml` 側は反映だけスキップして取得自体は続行する。

PATの発行手順:
1. GitHubの `https://github.com/settings/personal-access-tokens/new` でFine-grained PATを発行
2. Repository access で `moko-motetsu/pregnancy-events` のみを選択
3. Permissions の Contents を **Read and write** に設定
4. 発行したトークンを `test` リポジトリの Settings > Secrets and variables > Actions で `PREGNANCY_EVENTS_DEPLOY_TOKEN` として登録

## `events.json` の書き方

イベント情報はこのファイルを直接編集して手動でメンテナンスします（`zukan-watch` のようなAPI自動収集は未実装。将来的に無料の公開イベントAPI等に置き換え可能な構造にしてあります）。

```jsonc
{
  "categories": ["祭り・花火", "マルシェ・フリマ", ...],  // カテゴリチップに表示される一覧
  "events": [
    {
      "id": "一意なID（英数字とハイフン）",
      "name": "イベント名",
      "category": "categoriesに含まれるいずれか",
      "description": "説明文",
      "venue": "会場名",
      "address": "住所",
      "lat": 35.6895,   // 緯度。Googleマップで会場を右クリック→表示された数値をコピー
      "lng": 139.6917,  // 経度
      "startDate": "YYYY-MM-DD",
      "endDate": "YYYY-MM-DD",     // 単日開催ならstartDateと同じ値でOK
      "startTime": "10:00",        // 任意
      "endTime": "17:00",          // 任意
      "price": {
        "type": "free または paid",
        "note": "料金の補足（任意）",
        "purchaseUrl": "有料の場合の購入サイトURL（任意）"
      },
      "officialUrl": "公式サイトURL",
      "image": "イベント画像のURL（任意。公式サイト上の画像を直リンクで表示。読み込み失敗時はカテゴリアイコンに自動フォールバック）",
      "source": "情報源（例: ○○実行委員会 公式サイト）",
      "updatedAt": "YYYY-MM-DD（この情報を確認・更新した日）",
      "pregnancyRecommend": {          // 任意。妊娠週数のおすすめレンジ
        "weekMin": 14, "weekMax": 27, "note": "おすすめ理由"
      },
      "season": ["夏"],                 // 任意。該当する季節（春/夏/秋/冬、複数可）。通年ならフィールド自体を省略
      "seasonNote": "海開きシーズンの定番。",  // 任意。季節のおすすめ理由
      "tags": ["屋外", "ベビーカーOK"],  // 任意
      "reviews": [                      // 任意。口コミ
        { "source": "Googleマップ", "rating": 4.3, "comment": "...", "url": "..." }
      ],
      "trafficTips": {                  // 任意。渋滞回避・アクセス情報
        "summary": "一言まとめ", "car": "車での行き方・渋滞情報",
        "train": "電車での行き方", "parking": "駐車場情報", "congestion": "混雑時間帯"
      },
      "deals": [                        // 任意。お得情報
        { "title": "前売り割引", "description": "...", "url": "..." }
      ]
    }
  ]
}
```

新しいイベントを追加したら `id` の重複がないことを確認し、公式サイトの情報を必ず一次情報として `officialUrl` と `source` に記載してください。

### `venues`（週替わりで公演が変わる会場）

コンサート会場やアリーナのように、公演・興行が週替わりで入れ替わる場所は `events` ではなく `venues` に登録します。個別の公演名や日付は掲載せず、地点との距離だけで近い会場を検出し、公式サイトのスケジュールページへリンクします（手動メンテナンスでは公演の入れ替わりに追従できず、誤った公演情報を「公式情報」として出してしまうことを避けるため）。

```jsonc
{
  "id": "一意なID",
  "name": "会場名",
  "category": "categoriesに含まれるいずれか",
  "lat": 35.6423, "lng": 139.7936,
  "address": "住所",
  "officialUrl": "会場の公式サイトURL",
  "scheduleUrl": "公演スケジュール・イベント一覧ページのURL（無ければofficialUrlと同じでOK）",
  "source": "情報源",
  "updatedAt": "YYYY-MM-DD",
  "note": "会場についての補足（例: 週替わりで公演が入れ替わる旨）"
}
```

新しい会場を追加する場合も同様に `venues` 配列へ追記してください。

### 会場スケジュールの自動取得（`venue_schedules.json`）

`venues` に登録した施設については、`pregnancy-events/scripts/fetch_venue_schedules.mjs` が各施設の `scheduleUrl` からJSON-LD（`application/ld+json`、schema.org の `Event`）の構造化データを探し、直近のイベントを `venue_schedules.json` に保存します。このファイルは自動生成物なので手動編集しないでください。

- 実行方法: GitHub Actions の「有明会場スケジュール取得」ワークフロー（`.github/workflows/watch_ariake_venues.yml`）。毎日6:15 JSTに自動実行。
- あるサイトでJSON-LDが見つからない場合、そのサイトのデータだけ前回の内容を維持し、他の施設の結果には影響しない（1施設の取得失敗が全体を壊さない設計）。JSON-LDが無いサイト向けに、有明アリーナ・有明GYM-EX・有明ガーデンシアターはそれぞれサイト構造に合わせた個別パーサーを実装済み。
- 新しい会場を `venues` に追加すれば、次回の取得から自動的に対象に含まれる（JSON-LDが無いサイトの場合は個別パーサーの追加が必要）。
- 取得できた各イベントの画像URL（サムネイル等）も `image` フィールドとして保存され、アプリ側では「近くの定期イベント会場」欄に小さいサムネイルとして表示されます（画像は公式サイト上のものを直リンクで表示。読み込みに失敗した場合は自動的に非表示になります）。

### イベント開催日の自動確認（`event_confirmations.json`）

`events` 内で `officialUrl` を持つ全イベント（カテゴリ不問）について、`pregnancy-events/scripts/fetch_event_confirmations.mjs` が `officialUrl` のページからJSON-LD（見つからない場合は `og:description` 内の和暦表記）で開催日を確認し、`event_confirmations.json` に保存します。`events.json` 自体は書き換えず、確認が取れたイベントだけアプリ側で表示時に上書きする仕組みです。このファイルも自動生成物なので手動編集しないでください。

もともとは「祭り・花火」カテゴリのみを対象にしていましたが、開催日・料金が変わりうるのはどのカテゴリのイベントも同じなので、`officialUrl` を持つ全イベントに対象を広げています。

- 実行方法: GitHub Actions の「イベント開催日確認」ワークフロー（`.github/workflows/watch_event_confirmations.yml`）。毎日6:30 JSTに自動実行（有明会場ワークフローと同様の頻度）。
- **対象は自治体・観光協会・実行委員会・施設運営者などの一次情報源のみ**。ウォーカープラス等のまとめサイトは利用規約で無断転載を禁止しているため、対象に含めないこと。新しいイベントを追加する際は、必ず主催者・自治体・施設の公式サイトを `officialUrl` に設定する。
- 開催日が確認できると、詳細画面の情報源欄に「開催日を公式サイトで自動確認済み」の注記が表示される。
- JSON-LD・和暦フォールバックのいずれでも確認できないサイト（多くはJS描画やカスタムCMS）は、`events.json` の手動入力値がそのまま使われる。取得失敗が続く場合はワークフローのログに診断用HTMLが出力されるので、そこからパーサーを追加できる。

## 注意事項

- 掲載しているイベント情報・座標・料金・口コミ・渋滞情報はすべて手動入力のサンプル/参考情報です。実運用前に必ず公式サイトで最新情報を確認し、正確なデータに更新してください。
- 位置情報はブラウザの `localStorage` にのみ保存され、外部送信は行いません。
