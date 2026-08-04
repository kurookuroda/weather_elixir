# 第6章: 中間デプロイチェックポイント①（本番で「ロジック層」を検証する）

第0章で作った「空っぽのPhoenixアプリがDokployで動く」という土台の上に、第1〜5章で組み上げてきた「気圧変化を計算してLiveViewに表示する」機能を初めて載せます。

ここでいきなり第7章のLeaflet統合に進みたくなる気持ちを抑えて、**あえて機能が地味な段階でデプロイを挟む**のがこのチュートリアルの肝にゃ。理由は単純で、「ローカルでは動くが本番では動かない」原因を、JS連携という複雑な変数が増える前に潰しておきたいからです。ここを飛ばすと、第9章で山ほどの原因（Elixirの設定？Dokployの設定？それともJS Hookのバグ？）が絡み合って収拾がつかなくなりますにゃ。

---

## この章でじっくり学ぶこと

- **`mix release` とは何か**: Nuxtの`npm run build`が「静的アセット＋SSRサーバーのコード」を作るのに対し、Elixirの`mix release`は何を作るのか
- **`runtime.exs` の役割**: コンパイル時の設定(`config.exs`)と、起動時に環境変数から読む設定(`runtime.exs`)がなぜ分かれているのか
- **`SECRET_KEY_BASE` の必然性**: LiveViewのWebSocket接続がなぜこの値に依存するのか
- **外部API疎通という「見えない依存」**: ローカルでは動くのに本番で動かない典型パターン
- **`check_origin`とリバースプロキシ越しのWebSocket**: HTTPは通るのにWebSocketだけ拒否される、という本番特有の罠
- **コンテナ内のタイムゾーン**: ローカルではJST、本番ではUTCという9時間ズレの罠
- **Dokployのビルド〜デプロイフロー**: Dockerfileベースのビルドで何が起きているか

---

## 移植対象と「見えない前提」の違い

Nuxtアプリをホスティングサービス(Vercel等)にデプロイする場合、多くはビルド時にほぼ全ての設定が完了し、実行時に読む環境変数は最小限（APIキー程度）で済みます。

Elixirのリリースはこれと発想が異なります。

```elixir
# config/runtime.exs（起動時に評価される）
if config_env() == :prod do
  secret_key_base =
    System.get_env("SECRET_KEY_BASE") ||
      raise "SECRET_KEY_BASE が設定されていません"

  host = System.get_env("PHX_HOST") || "example.com"

  config :weather_ex, WeatherExWeb.Endpoint,
    url: [host: host, port: 443, scheme: "https"],
    # WebSocket接続を許可するオリジン。未設定/不一致だと本番でWSだけ拒否される
    check_origin: ["https://#{host}"],
    http: [port: String.to_integer(System.get_env("PORT") || "4000")],
    secret_key_base: secret_key_base,
    server: true
end
```

**ここで見落としがちな設定が`check_origin`です。** Dokploy内部のリバースプロキシ(Traefik)越しにWebSocketが動くため、LiveViewはリクエストの`Origin`ヘッダーが自分のドメインと一致するかを検証します。この設定が抜けていると、**HTTPの初回アクセス（静的HTML）は表示されるのにWebSocketだけ拒否され、「時計が動かない＝データが更新されない」という一見謎の障害**になります。ブラウザの開発者ツールでWebSocketリクエストが`403`になっていたら、まずこれを疑うにゃ。

Vueの`import.meta.env.VITE_XXX`はビルド時に値がJSバンドルへ**焼き込まれ**ます。しかし`runtime.exs`は、**コンテナが起動した瞬間**に環境変数を読みます。同じビルド成果物（Dockerイメージ）を、環境変数だけ差し替えてステージングにも本番にも使い回せるのはこのためです。逆に言うと、**ビルドは通ってもコンテナ起動時に環境変数が足りずクラッシュする**という、Nuxtではあまり馴染みのない失敗パターンがElixirリリースには存在しますにゃ。

---

## Step 0: これまでの純粋ElixirコードをPhoenixに統合する

第1〜4章では `mix new weather_ex --sup` という**純粋Elixirプロジェクト**として進めてきました。しかし第5章のLiveViewを動かすにはPhoenixが必要です。「いつPhoenixになったのか」を明確にしておきます。

- **第0章を実施済みの場合**: 第0章で作った`WeatherExWeb`（Phoenixプロジェクト）の`lib/weather_ex/`以下に、第1〜4章で作った各モジュール（`Observation`, `JmaClient`, `StationCache`, `Weather`）をそのまま配置します
- **第0章を飛ばした場合**: ここで`mix phx.new weather_ex`を実行し、`lib/weather_ex/`以下に同様にモジュールを配置してください

いずれの場合も、`mix.exs`の依存関係に`{:req, "~> 0.5"}`が含まれていることを確認します。Nuxtで言えば「別々に作っていたユーティリティ関数群を、Nuxtプロジェクトの`server/utils/`に統合する」ような作業にゃ。

---

## Step 1: リリース関連ファイルの確認

第0章で`mix phx.gen.release --docker`を実行済みのはずなので、以下のファイルが揃っているか確認します。

```bash
ls Dockerfile rel/overlays/bin/server config/runtime.exs
```

`config/runtime.exs`を開き、第0章のひな型から変わっていないことを確認します（今回は新しい環境変数を増やしません。JMAへのHTTP通信は`Req`ライブラリが直接アウトバウンド接続するだけで、APIキーの類が不要なため）。

**ここが今回のポイント**: 環境変数を増やさなくても、「機能が増えただけ」でクラッシュする可能性があります。次のStepで確認します。

---

## Step 2: 本番ビルドをローカルで模擬する

Dokployに投げる前に、ローカルで本番相当のビルドを試すと事故を減らせます。

```bash
MIX_ENV=prod mix release --overwrite
PHX_HOST=localhost SECRET_KEY_BASE=$(mix phx.gen.secret) PORT=4000 \
  _build/prod/rel/weather_ex/bin/server
```

ブラウザで`http://localhost:4000/weather`を開き、第5章までの状態（時計・カウントダウン・気圧データ）が表示されれば、少なくとも「Elixir側のprodビルド」は健全です。

> **補足**: `PHX_HOST=localhost`のままでも今回のビルド検証としては十分ですが、より本番に近い状態で試したい場合は本番と同じドメイン（例: `PHX_HOST=weather-app.example.com`）で起動しても構いません。必須ではないので、まずは`localhost`で先へ進んで大丈夫です。

**深掘りポイント**: この`_build/prod/rel/weather_ex/bin/server`は、Erlang VM（BEAM）ごと固めた自己完結型の実行ファイル群です。Nuxtの`node .output/server/index.mjs`のように**別途ランタイム（Node.js）をインストールする必要がありません**。Dockerイメージが軽量になりやすいのはこのためですにゃ。

> **補足**: `mix assets.deploy`のステップで一瞬Node.jsが動きます。これはTailwind CSSやLiveViewのJSコードをコンパイルするためです。**最終的なリリース（`bin/server`）にはNode.jsは含まれない**ので、Nuxtのように実行環境にNode.jsを入れ続ける必要はないにゃ。

---

## Step 3: Dokployへデプロイ

第0章で連携済みのDokployプロジェクトに戻り、GitHubへのpushをトリガーに再ビルドさせます。

```bash
git add .
git commit -m "第6章: LiveViewで気圧データを表示する機能を追加"
git push origin main
```

Dokployのダッシュボードでビルドログを確認します。ここで見るべきポイントは3つです。

1. **Dockerビルドが成功しているか**（`mix deps.get`〜`mix assets.deploy`〜`mix release`の各ステップ）
2. **コンテナが起動直後にクラッシュしていないか**（`runtime.exs`の環境変数チェックに引っかかっていないか）
3. **ヘルスチェックが通っているか**（Dokployは指定したパスへの応答でコンテナの健全性を判定します）

環境変数はDokployの「Environment」タブで、第0章時点の`SECRET_KEY_BASE` / `PHX_HOST` / `PORT`がそのまま使えます。今回は追加不要です。

---

## Step 4: 本番での動作確認

デプロイ完了後、本番URLの`/weather`にアクセスします。

- [ ] 静的HTML（`mount`の`connected?`が`false`の状態）がまず表示される
- [ ] 数秒後、時計とカウントダウンが動き出す（＝WebSocketが繋がっている）
- [ ] 気圧データの一覧が表示される（＝本番コンテナからJMAのAPIへ外向き通信が通っている）

もし3番目だけ失敗する場合、原因はほぼ確実に**VPS側のアウトバウンド通信規制**です。ローカル開発機やCIとは異なり、本番VPSはファイアウォールでアウトバウンド通信を絞っていることがあり、`www.jma.go.jp`への接続がブロックされているケースがあります。

```bash
# Dokployのダッシュボード → 該当サービス → 「Terminal」タブを開いてから以下を実行
curl -I https://www.jma.go.jp/bosai/amedas/data/latest_time.txt
```

これが失敗するなら、VPSのファイアウォール設定（`ufw`等）でHTTPS(443)のアウトバウンドを許可します。**Nuxtアプリをサーバーレス環境にデプロイしていた頃は意識しなかった層の問題**が、自前のVPS運用では顔を出しますにゃ。

> **補足**: `curl`がタイムアウトするが、IP指定（`curl -I https://210.145.168.56/...`等）だと繋がる場合、Dockerネットワーク内の**DNS解決**に問題がある可能性があります。以下を確認してください。
> ```bash
> nslookup www.jma.go.jp
> ```
> 名前解決ができない場合、DokployのDockerネットワーク設定（`docker-compose`の`dns`指定など）を見直す必要がありますにゃ。

---

## つまずきやすい点

- **`SECRET_KEY_BASE`未設定でコンテナが即クラッシュ**: `runtime.exs`の`raise`がそのまま起動失敗として現れます。Dokployの**「Logs（実行ログ）」タブ**（ビルドログではない方）で「Could not start」の直前に出るエラーメッセージを必ず確認するにゃ
- **ビルドは通るが起動しない**: `mix release`はコンパイルエラーがなければ成功してしまうため、「ビルド成功＝本番で動く」ではないことに注意。Step 2のローカル模擬ビルドを省略しないこと
- **VPSのアウトバウンド制限**: 気象庁APIへの疎通がVPS側のファイアウォールで塞がれていないか確認する（意外と見落としがち）
- **DNS解決の失敗**: `curl`で疎通確認する際、名前解決（`nslookup`）も併せて確認する。Dockerネットワーク内でDNSが機能しないケースがあるにゃ
- **WebSocketは実はこの章ではまだ「完全には」検証していない**: 時計が動いていれば基本的な疎通はできていますが、Traefik(Dokployが内部で使うリバースプロキシ)経由のWebSocketヘッダー周りの本格的な最終確認は第9章で改めて行います。ここでは「動いているように見える」レベルでOKにゃ
- **`mix phx.gen.secret`は毎回違う値を生成する**: ローカル確認用と本番用で同じ値である必要はありませんが、本番の値は一度決めたら変更しないこと（変更するとセッションやCSRFトークンが無効になります）
- **`check_origin`未設定によるWebSocket拒否**: 上記Step 1で追加した設定を忘れると、HTTPは通るのにWebSocketだけ`403`で切断される。本番ドメインでだけ再現する厄介な障害なので要注意にゃ
- **コンテナ内のタイムゾーンがUTCになっている**: ローカルではJSTで動いていた時計やJMAの観測時刻が、本番では9時間ズレて表示されることがあります。Dokployの環境変数に `TZ=Asia/Tokyo` を追加し、コンテナ起動時に読み込ませるようにするにゃ
- **ヘルスチェックのパス**: Dokployはデフォルトでルートパス`/`をヘルスチェック対象にします。第5章時点では`/`はPhoenixデフォルトの静的ページのままなので今回は問題ありませんが、将来的に`/`を差し替える場合はDokploy側のヘルスチェックパスを`/weather`などに変更するのを忘れないこと

---

## 章末チェックリスト

- [ ] `MIX_ENV=prod mix release`でローカル本番ビルドが動くことを確認した
- [ ] `runtime.exs`が「コンパイル時」ではなく「起動時」に評価される設定であることを理解した
- [ ] Dokployへpush→再ビルド→デプロイの一連の流れを実行した
- [ ] 本番URLで静的HTML→WebSocket接続→気圧データ表示、の3段階が確認できた
- [ ] （該当する場合）VPSのアウトバウンド通信規制を解除し、JMA APIへの疎通を確認した
- [ ] （該当する場合）`TZ=Asia/Tokyo`環境変数を設定し、時刻のズレを解消した
- [ ] 「ロジック層は本番で動く」という前提を得た状態で、次章（Leaflet統合）に進める

---

前章: [第5章 — LiveViewの骨組み（Elixir入門・深掘り編）](./chapter_05.md)  
次章: 第7章 — Leaflet JS Hookで地図統合

---

準備できたら第7章に進むにゃ。第6章で「本番で動く」という自信を得た上で、いよいよVueのLeafletコードをLiveView+JS Hookに移植する核心章に突入するにゃ。
