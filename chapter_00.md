# 第0章: Walking Skeleton — 環境構築からDokployへの初回デプロイ

このチュートリアルは、Nuxt/Vue製の気圧変化マップアプリをElixir/Phoenixに移植しながらElixirを学ぶシリーズです。第1〜9章はすでに存在していますが、実は一番最初に済ませておくべき「土台作り」が抜けていました。それがこの第0章です。

第1章を開くといきなり`mix new weather_ex --sup`という、Phoenixを使わない**素のElixirプロジェクト**を作るところから始まります。ところが第5章では`live "/weather", WeatherLive`というルーティングが登場し、第6章では「第0章で作った`WeatherExWeb`(Phoenixプロジェクト)」という記述が当然のように出てきます。この章は、その間を埋めるために書かれました。

## この章でじっくり学ぶこと

- **なぜ「機能ゼロ」の状態を最初にデプロイするのか**(Walking Skeletonという考え方)
- `mix phx.new`のオプション(`--no-ecto`など)が何を意味するか
- Phoenixのディレクトリ構成とNuxtのディレクトリ構成の対応関係
- `mix phx.gen.release --docker`が何を生成するか
- DokployというセルフホストPaaSの基本操作(VPSセットアップ〜GitHub連携〜初回デプロイ)
- 「本番URLで何かが表示される」という**最小の成功体験**を最初に得ることの意味

---

## Step 0: なぜ機能ゼロの状態でデプロイするのか

ソフトウェア開発には**Walking Skeleton**(歩く骨格)という考え方があります。機能は何もないが、システムの端から端まで(今回で言えば「ブラウザ→インターネット→VPS上のPhoenixコンテナ」)が、とにかく一度繋がって動く状態を最初に作る、というプラクティスです。

なぜ先に地図やLiveViewを作らないのか。理由は第6章・第9章を読むと分かりますが、先取りして説明すると:

- 「ローカルでは動くが本番では動かない」という問題には、**Elixir側の設定ミス**・**Dokploy側の設定ミス**・**インフラ(VPS/ネットワーク)側の問題**という3つの原因が絡み合います
- 機能を全部作ってから初めてデプロイを試すと、この3つが同時に襲いかかってきて、どこに原因があるのか切り分けるだけで一苦労します
- 逆に「何もない状態」で先にデプロイを済ませておけば、後の章でエラーが起きたときに「機能追加で何かを壊した」と即座に分かります

Nuxt/Vercelの世界では、`git push`すればほぼ自動的にこの「土台」が用意されるため、あまり意識する機会がありませんでした。Elixir + 自前VPS(Dokploy)構成では、この土台自体を自分の手で組み立てる必要があります。これが今回のスタート地点です。

---

## Step 1: ローカル環境の確認

まずElixir/Erlangが入っているか確認します。

```bash
elixir -v
```

Phoenix 1.8系を使うには、**Elixir 1.17以上・Erlang/OTP 25以上**が必要です。入っていない場合は[Elixir公式のインストールページ](https://elixir-lang.org/install.html)を参照してください(`asdf`や`mise`のようなバージョン管理ツールを使うと、後々プロジェクトごとにバージョンを固定しやすくなります)。

次にPhoenixのプロジェクトジェネレータをインストールします。

```bash
mix archive.install hex phx_new
```

---

## Step 2: `mix phx.new`でプロジェクトを作る

```bash
mix phx.new weather_ex --no-ecto
```

**なぜ`--no-ecto`を付けるのか**: `Ecto`はElixir版のORM(データベースアクセス層)です。Phoenixはデフォルトで「PostgreSQLを使う」前提のプロジェクトを生成しますが、このアプリには永続化すべきデータがありません。観測地点マスタは第3章で作った`StationCache`(GenServerのメモリ上キャッシュ)で十分であり、そもそも元のNuxtアプリにもデータベースは存在しませんでした。**要らないものは最初から生成しない**、という判断です。

> **補足**: `--live`フラグを見たことがある人もいるかもしれませんが、Phoenix 1.6以降は**LiveViewがデフォルトで有効**になっているため、明示的に付ける必要はありません。

```bash
cd weather_ex
```

インストール確認のプロンプトが出たら`Y`を選び、依存関係の取得を待ちます。

### ディレクトリ構成: Nuxtとの対応

生成されたディレクトリを見て、Nuxtの構成と対応付けてみましょう。

| Nuxt | Phoenix | 役割 |
|---|---|---|
| `pages/` | `lib/weather_ex_web/live/` | ページ(今回はLiveView) |
| `server/api/` | `lib/weather_ex_web/controllers/` | APIエンドポイント |
| `server/utils/` | `lib/weather_ex/` | ビジネスロジック(第1〜4章の`Observation`/`JmaClient`/`StationCache`/`Weather`はここに置く) |
| `public/` | `priv/static/` | 静的ファイル |
| `assets/`(自分で作る) | `assets/` | CSS/JS(こちらはPhoenix側も同じ名前) |
| `nuxt.config.ts` | `config/config.exs`, `config/runtime.exs` | 設定ファイル |
| `.output/` | `_build/`, `rel/` | ビルド成果物 |

`lib/weather_ex/`(Webを知らないビジネスロジック層)と`lib/weather_ex_web/`(Web層)が明確に分かれているのがPhoenixの特徴です。第1〜4章のコードは前者に、第5章以降のLiveViewコードは後者に置かれます。

### ローカルで起動確認

```bash
mix phx.server
```

`http://localhost:4000` を開き、Phoenixのデフォルトのウェルカムページが表示されれば成功です。この時点ではまだ地図もLiveViewの`/weather`ルートもありません。それでいいのです — Walking Skeletonの精神に従い、**まず「何かが動く」ことだけを確認します**。

---

## Step 3: Gitリポジトリとして初期化する

```bash
git init
git add .
git commit -m "第0章: mix phx.new でプロジェクトの骨格を作成"
```

GitHubで新しいリポジトリ(例: `weather_ex`)を作り、pushします。

```bash
git remote add origin https://github.com/<あなたのアカウント>/weather_ex.git
git push -u origin main
```

> **注意**: このチュートリアル自体(`weather_elixir`リポジトリ)は解説文書だけを置くドキュメント用リポジトリです。実際に手を動かすElixirのプログラム本体は、**別のリポジトリ**(ここでは`weather_ex`)として管理します。Dokployにも、この`weather_ex`リポジトリの方を接続します。

---

## Step 4: `mix phx.gen.release --docker`でDocker関連ファイルを生成する

```bash
mix phx.gen.release --docker
```

これにより以下が生成されます。

- `Dockerfile` — 本番用のマルチステージビルド定義
- `rel/overlays/bin/server`, `rel/overlays/bin/server.bat` — リリース起動スクリプト
- `config/runtime.exs`の追記 — 起動時に環境変数を読む設定の雛形

`config/runtime.exs`を開いて、`if config_env() == :prod do ... end`のブロックがあることを確認してください。中身の詳しい意味は第6章でじっくり扱いますが、ここでは「**コンパイル時ではなく、コンテナが起動した瞬間に環境変数を読む場所**」という位置づけだけ覚えておけば十分です。

```bash
git add .
git commit -m "第0章: mix phx.gen.release --docker でDocker関連ファイルを生成"
git push
```

---

## Step 5: VPSにDokployをインストールする

さくらのVPS・Hetzner・ConoHaなど、SSH接続できるLinux(Ubuntu 22.04+/Debian 11+推奨)のVPSを1台用意します。最小構成の目安は**RAM 2GB以上、空き容量10GB以上**です。

SSHでVPSに接続し、rootまたはsudo権限で以下を実行します。

```bash
curl -sSL https://dokploy.com/install.sh | sh
```

このスクリプトが自動的にDockerのインストールも含めてDokployのセットアップを行います。数分で完了し、最後に管理画面のURL(`http://<VPSのIP>:3000`)が表示されます。

> **ファイアウォールの確認**: ポート`80`(HTTP)・`443`(HTTPS)・`3000`(Dokploy管理画面、後で塞いでも構いません)がVPS側で開いていることを確認してください。`ufw`を使っている場合は`ufw allow 80/tcp`, `ufw allow 443/tcp`のように許可します。
>
> **補足**: 一部のVPSでは`ufw`が初期状態で**インストールされていない・無効になっている**ことがあります。その場合は`ufw status`で状態を確認し、無効ならば有効化するか、**プロバイダ側のセキュリティグループ（クラウドファイアウォール）**設定を確認してくださいにゃ。例えばAWS EC2やGCP Compute Engineでは、OS側の`ufw`とは別に、コンソール側でポートを開放する必要があります。

ブラウザで管理画面を開き、初回アクセス時に管理者アカウントを作成します。

---

## Step 6: DokployでGitHub連携し、初回デプロイする

1. Dokploy管理画面で **「Create Project」** → 適当なプロジェクト名(例: `weather-map`)を入力
2. プロジェクト内で **「Create Service」** → **「Application」** を選択
3. **Source**として GitHub を選び、Step 3で作った`weather_ex`リポジトリを連携(初回は GitHub App のインストール許可が必要です)
4. **Build Type**を`Dockerfile`に設定(Step 4で生成した`Dockerfile`がリポジトリルートにあることを確認)
5. **Environment**タブで、最低限必要な環境変数を設定します:

```bash
SECRET_KEY_BASE=<mix phx.gen.secret の出力をコピー>
PHX_HOST=<Dokployが割り当てる一時ドメイン、または独自ドメイン>
PORT=4000
```

`SECRET_KEY_BASE`は手元で以下のコマンドを実行して生成した値を使います。

```bash
mix phx.gen.secret
```

> **補足**: この`SECRET_KEY_BASE`は本番でも使う値なので、安全な場所に控えておくとよいにゃ。ローカルでの模擬ビルド（第6章参照）でも同じ値を使えますが、ローカル用と本番用で別の値を生成しても構いません。

6. **Domains**タブで、Dokployが自動発行する一時ドメイン(`xxxx.traefik.me`のような形式)を確認するか、独自ドメインをDNS設定込みで登録します
7. **Deploy**ボタンを押して初回デプロイを実行します

---

## Step 7: 本番での動作確認

ビルドログ(Dokployの「Deployments」タブ)を確認し、以下の3点をチェックします。

1. **Dockerビルドが成功しているか**(`mix deps.get`〜`mix assets.deploy`〜`mix release`が一通り走っているか)
2. **コンテナが起動直後にクラッシュしていないか**(「Logs」タブで確認。`SECRET_KEY_BASE`が未設定だとここでクラッシュします)
3. **割り当てられたURLにアクセスして、Phoenixのデフォルトウェルカムページが表示されるか**

これが表示されれば、Walking Skeletonの完成です。**まだ機能は何もありません**が、「ローカルのコード変更 → Git push → Dokployで自動ビルド → 本番URLで確認」という開発サイクルの土台が整いました。

---

## 補足: 第1章の記述との整合性について

ここで一つ、率直に注記しておきます。第1章には次の記述があります。

> まずはElixirのプロジェクトを作ります。(中略)
> ```bash
> mix new weather_ex --sup
> cd weather_ex
> ```

この章(第0章)を先に実施した場合、**このコマンドは実行しないでください**。第0章の`mix phx.new weather_ex --no-ecto`で作ったプロジェクトが、そのまま第1章以降で使うプロジェクトです。`--sup`オプション(Supervisorツリーを持つアプリケーションにする)は、`mix phx.new`が生成するプロジェクトにも標準で含まれているため、機能的な過不足はありません。

これは第1〜4章が「第0章が存在しない状態」で先に書かれたために生じた重複です。実際、第6章の冒頭にも「第0章を実施済みの場合/飛ばした場合」という分岐が用意されており、この食い違いは執筆側でも認識した上で許容されています。第1章を読む際は、`mix new`のくだりは読み飛ばし、`lib/weather_ex/weather/observation.ex`の作成から始めてください。

---

## つまずきやすい点

- **`--no-ecto`を忘れると`DATABASE_URL`が無くて本番でクラッシュする**: `mix phx.new weather_ex`だけ実行してしまった場合、`config/runtime.exs`にPostgres接続の設定が生成され、本番起動時に`DATABASE_URL`が無いとエラーになります。今回のアプリにDBは不要なので、作り直すか、該当のEcto関連設定を手動で削除してください
- **`SECRET_KEY_BASE`未設定によるクラッシュ**: Dokployの「Environment」タブへの設定を忘れると、コンテナは起動直後にクラッシュします。「Logs」タブ(ビルドログではなく実行ログ)でエラーメッセージを確認する癖をつけてください
- **Dockerfileのビルドコンテキスト**: DokployのBuild設定で、リポジトリのルート(`Dockerfile`が置いてある場所)を正しく指定できているか確認してください。モノレポ構成にする場合は特に注意が必要です
- **ポート番号の不一致**: `mix phx.gen.release --docker`が生成する`runtime.exs`はデフォルトで`PORT`環境変数(未設定なら`4000`)を読みます。Dokploy側の「Port」設定もこれに合わせてください
- **DokployのGitHub App権限**: プライベートリポジトリを連携する場合、GitHub App側でそのリポジトリへのアクセス権限を明示的に許可する必要があります。連携直後にリポジトリの一覧に出てこない場合はここを確認してください
- **VPSのファイアウォールが多層構造になっている**: `ufw`だけでなく、プロバイダ側のセキュリティグループ（AWSのSecurity Group、GCPのFirewall Rules、さくらのクラウドファイアウォールなど）でもポート開放が必要なケースがあります。Dokployの管理画面(`:3000`)にすらアクセスできない場合は、まずプロバイダ側の設定を疑うにゃ

---

## 章末チェックリスト

- [ ] `elixir -v`でElixir 1.17以上・Erlang/OTP 25以上を確認した
- [ ] `mix phx.new weather_ex --no-ecto`でプロジェクトを作成した
- [ ] `mix phx.server`でローカル起動し、`http://localhost:4000`でウェルカムページを確認した
- [ ] `weather_ex`をGitHubの別リポジトリとしてpushした(このチュートリアル文書のリポジトリとは別)
- [ ] `mix phx.gen.release --docker`でDocker関連ファイルを生成した
- [ ] VPSにDokployをインストールし、管理画面にアクセスできた
- [ ] DokployでGitHub連携し、`SECRET_KEY_BASE`/`PHX_HOST`/`PORT`を設定した
- [ ] 初回デプロイに成功し、本番URLでPhoenixのウェルカムページが表示されることを確認した
- [ ] 第1章を読む際、`mix new weather_ex --sup`の行は実行不要であることを理解した

---

次章: [第1章 — 純粋関数でロジック移植](./chapter_01.md)

---

準備できたら第1章に進むにゃ。ここからは`lib/weather_ex/weather/`以下にビジネスロジックを積み上げていく、Elixir入門の本編が始まるにゃ。
