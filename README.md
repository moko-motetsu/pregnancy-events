# マタニティおでかけイベントガイド

現在地から100km圏内のおでかけイベントを一覧化するWebアプリ。カテゴリで絞り込みができ、有料イベントは購入サイトへのリンクを表示。妊娠週数に応じたおすすめイベントも紹介します。

`zukan-seal-collection` と同じく **`index.html` 1ファイルで完結する静的Webアプリ**です。ビルドやサーバーは不要ですが、`events.json` を読み込むため `file://` で直接開くとブラウザのセキュリティ制限で読み込みに失敗します（後述のフォールバックでアプリ自体は起動しますが、サンプルデータのみの表示になります）。ローカルサーバー経由か、GitHub Pages 等で公開して使ってください。

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
- **公式情報の表示** — 各イベントに情報源(`source`)と最終更新日(`updatedAt`)を明記し、詳細画面に「必ず公式サイトでご確認ください」の注記を表示。
- **口コミ・渋滞回避・お得情報** — イベント詳細モーダルに、口コミ（評点・コメント・出典）、渋滞回避／アクセス情報（車・電車・駐車場・混雑時間帯）、お得情報（割引券など）を表示。

## ローカルで動かす

```bash
cd pregnancy-events
python3 -m http.server 8080
# ブラウザで http://localhost:8080/ を開く
```

`events.json` の取得に失敗した場合（`file://` で直接開いた場合など）は、画面上部に警告バナーが出てサンプルデータ1件のみで動作します。

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

- 実行方法: GitHub Actions の「有明会場スケジュール取得」ワークフロー（`.github/workflows/watch_ariake_venues.yml`）。実サイトでの動作検証が済むまでは `workflow_dispatch`（Actionsタブから手動実行）のみが有効。検証後、ワークフロー内のコメントを外せば定期実行に切り替えられる。
- あるサイトでJSON-LDが見つからない場合、そのサイトのデータだけ前回の内容を維持し、他の施設の結果には影響しない（1施設の取得失敗が全体を壊さない設計）。
- 新しい会場を `venues` に追加すれば、次回の取得から自動的に対象に含まれる（スクリプト側の追加変更は不要）。

## 注意事項

- 掲載しているイベント情報・座標・料金・口コミ・渋滞情報はすべて手動入力のサンプル/参考情報です。実運用前に必ず公式サイトで最新情報を確認し、正確なデータに更新してください。
- 位置情報はブラウザの `localStorage` にのみ保存され、外部送信は行いません。
