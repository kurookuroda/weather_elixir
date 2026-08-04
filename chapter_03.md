# 第3章: GenServerで観測地点マスタをキャッシュ（Elixir入門編）

第1章では「データを変換する純粋関数」を、第2章では「外部APIと通信し、`{:ok, _}` / `{:error, _}` で結果を返す関数」を作りました。どちらも**「呼ばれるたびに同じ処理をして、結果を返すだけ」**の関数でした。

しかし、実際のアプリケーションには「状態」が必要な場面があります。
元のTypeScriptコードにある `let stationsMaster = null` がまさにそれです。「一度取得したデータをメモリに保持し、次回からは取得処理をスキップする」という、いわゆる**キャッシュ**の仕組みです。

この章では、Elixirにおける「状態を持つ」という概念の根本から、**GenServer**というOTPの核心に迫ります。ここが、Node.js/TypeScriptとElixirの設計思想が最も大きく分かれる場所にゃ。

## この章でじっくり学ぶこと

- なぜElixirは「変数」ではなく「プロセス」で状態を持つのか
- **BEAM VMにおけるプロセス**の正体（OSのプロセスとは違う！）
- **GenServer**の基本構造（Client API と Server callbacks の分離）
- 状態の「代入」ではなく「新しい状態の返却」というイミュータブルな状態管理
- **依存性注入**によるテスト容易性の確保（モックライブラリに頼らない手法）
- **Supervision Tree**：アプリケーションを自動復帰する構造にする仕組み

---

## 移植対象: 元のTypeScriptコード

```typescript
// 観測地点マスタは毎回取得ではなく、サーバー起動時に一度だけ取得してキャッシュ
let stationsMaster: any = null

async function getStationsMaster() {
  if (!stationsMaster) {
    stationsMaster = await $fetch("https://www.jma.go.jp/bosai/amedas/const/amedastable.json")
  }
  return stationsMaster
}
```

Node.jsは基本的にシングルプロセスで動きます。だから、モジュールスコープに変数 `stationsMaster` を置いておけば、サーバーが動いている限りずっとメモリ上に残り、どのリクエストからも参照できます。

### なぜElixirで同じことをしてはいけないのか？

もしElixirで同じように「モジュール内の変数に状態を保持する」ような書き方ができたとします（実際にはElixirの変数は再代入できないためできませんが、もしもできたとしたら）。
Elixir/BEAM VMは、リクエストごとに**別々の軽量プロセス**を立ち上げて並行処理を行います。あるプロセスが `stationsMaster` に値を書き込んでも、別のプロセスからはそれが見えないか、あるいは同時に書き込んでデータが破壊される（競合状態）可能性があります。

**Elixirの答え**:
状態を持たせたいなら、**「状態専用の独立したプロセス」を1つ立ち上げ、その中に閉じ込める**のです。そして他のプロセスは、その「状態プロセス」にメッセージを送ってデータを要求します。これがGenServerの正体です。

---

## 実装: `lib/weather_ex/weather/station_cache.ex`

```elixir
defmodule WeatherEx.Weather.StationCache do
  @moduledoc """
  AMeDAS観測地点マスタをプロセス内にキャッシュするGenServer。

  JSのモジュールスコープ変数(let stationsMaster)が持っていた
  「初回だけ取得してあとはメモリに保持する」という役割を、
  ElixirではGenServerの状態として持たせる。
  """

  # GenServerの機能をこのモジュールに注入する
  use GenServer

  # ==========================================
  # --- Client API（呼び出し側が使う関数） ---
  # ==========================================

  @doc """
  StationCacheプロセスを起動する。

  ## Options
    * `:name` - プロセス登録名。デフォルトは `__MODULE__`。
    * `:fetch_fun` - データを取得する関数。デフォルトは本物のJMAクライアント。
  """
  def start_link(opts \\ []) do
    # opts から :name を取り出し（無ければモジュール名）、残りを opts に戻す
    {name, opts} = Keyword.pop(opts, :name, __MODULE__)

    # opts から :fetch_fun を取り出し（無ければデフォルトの関数を設定）
    # &Module.function/arity は「関数そのもの」を変数として渡す記法
    fetch_fun =
      Keyword.get(opts, :fetch_fun, &WeatherEx.Weather.JmaClient.fetch_stations_master/0)

    # name が nil でなければ [name: name] を、nil なら [] を gen_opts にする
    gen_opts = if name, do: [name: name], else: []
    
    # GenServerを起動。初期状態として %{stations: nil, fetch_fun: fetch_fun} を渡す
    GenServer.start_link(__MODULE__, %{stations: nil, fetch_fun: fetch_fun}, gen_opts)
  end

  @doc """
  観測地点マスタ全体を取得する。
  """
  def get_stations(server \\ __MODULE__) do
    # 別プロセス(GenServer)に :get_stations というメッセージを送り、返信を待つ
    GenServer.call(server, :get_stations)
  end

  @doc """
  指定した観測地点IDのマスタ情報を取得する。
  """
  def get_station(server \\ __MODULE__, station_id) do
    case get_stations(server) do
      # Map.fetch はキーがあれば {:ok, value}、なければ :error を返す
      {:ok, stations} -> Map.fetch(stations, station_id)
      {:error, _reason} = error -> error
    end
  end

  # ==============================================
  # --- Server Callbacks（プロセス側の処理） ---
  # ==============================================

  @impl true
  # プロセス起動時に呼ばれる。初期状態を受け取り、そのまま返す
  def init(state) do
    {:ok, state}
  end

  @impl true
  # :get_stations メッセージを受け取った時の処理（パターン1：未取得の場合）
  # state から stations が nil と fetch_fun を取り出す（パターンマッチ）
  def handle_call(:get_stations, _from, %{stations: nil, fetch_fun: fetch_fun} = state) do
    # 注入された関数を実行する（.() は変数に入った関数を実行する構文）
    case fetch_fun.() do
      {:ok, stations} ->
        # 成功したら、state の stations にデータを入れた「新しい状態」を作って返す
        {:reply, {:ok, stations}, %{state | stations: stations}}

      {:error, reason} ->
        # 失敗したら、state は変更せずそのまま返す（エラーをキャッシュしない）
        {:reply, {:error, reason}, state}
    end
  end

  @impl true
  # :get_stations メッセージを受け取った時の処理（パターン2：取得済みの場合）
  def handle_call(:get_stations, _from, %{stations: stations} = state) do
    # そのままキャッシュ済みの stations を返す
    {:reply, {:ok, stations}, state}
  end
end
```

## `jma_client.ex`に取得関数を追加

第2章のファイルに、観測地点マスタ取得用の関数を1つ追加します。

```elixir
@doc """
観測地点マスタ(amedastable.json)を取得する。StationCacheから呼ばれる。
"""
def fetch_stations_master do
  case Req.get("#{@base_url}/const/amedastable.json") do
    {:ok, %Req.Response{status: 200, body: body}} -> {:ok, body}
    {:ok, %Req.Response{status: status}} -> {:error, {:unexpected_status, status}}
    {:error, reason} -> {:error, reason}
  end
end
```

## Supervision Treeへの登録: `lib/weather_ex/application.ex`

作成したGenServerを、アプリケーション起動時に自動で立ち上げるようにします。

```elixir
defmodule WeatherEx.Application do
  @moduledoc false
  use Application

  @impl true
  def start(_type, _args) do
    children = [
      # ここにモジュール名を書くと、自動的に start_link([]) が呼ばれる
      WeatherEx.Weather.StationCache
    ]

    opts = [strategy: :one_for_one, name: WeatherEx.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

---

## Elixir的な着眼点：じっくり解説

### 1. プロセスの正体：OSのプロセスではなく「超軽量スレッド」

「プロセス」と聞くと、OSのプロセス（Chromeのタブとか）を思い浮かべるかもしれませんが、BEAM VMのプロセスは全く違います。
メモリ消費が数KB〜数十KBで、生成・破棄がマイクロ秒単位で行える**「超軽量な独立した世界」**です。
一つのアプリケーションの中に、何十万というプロセスが同時に存在することも珍しくありません。
GenServerは「状態を持つデータを、こうした独立した小さな箱の中に隔離する」ための仕組みです。

### 2. Client API と Server Callbacks の分離

GenServerのコードは、明確に上と下に分かれます。
- **Client API** (`start_link`, `get_stations`): 普通の関数です。呼び出すと裏側で `GenServer.call` が発動し、対象のプロセスにメッセージ（`:get_stations`）を送信します。
- **Server Callbacks** (`init`, `handle_call`): プロセスの箱の中で、送られてきたメッセージを受け取って処理する関数です。

なぜ分けるのでしょうか？ **呼び出し側に「相手がプロセスであること」を隠蔽するため**です。
呼び出し側は `StationCache.get_stations()` と書くだけで、裏でメッセージパッシングが起きていることや、プロセスが存在することを意識する必要がありません。これが「並行処理を簡単にする」OTPの強力な設計思想です。

### 3. 状態は「代入」ではなく「新しい状態を返す」で更新する

ここが一番のパラダイムシフトです。
JSなら `stationsMaster = data` と**代入（破壊的変更）**しますが、Elixirではできません。

```elixir
{:reply, {:ok, stations}, %{state | stations: stations}}
```

`handle_call` は `{:reply, クライアントへの返事, プロセスの新しい状態}` というタプルを返します。
`%{state | stations: stations}` というのはマップの更新構文で、「古い `state` の `stations` キーの中身だけを新しい `stations` に差し替えた、**新しいマップ**」を作っています。
プロセスの状態は、あなたが代わりに書き換えるのではなく、「次の状態はこれです」とGenServerに教えることで、VMが裏側でスッと差し替えてくれます。

### 4. `handle_call` の複数定義で `if` を消す

第1章で関数の引数によるパターンマッチを学びましたが、それを**プロセスの状態**に対して適用しています。

```elixir
# 状態が %{stations: nil, ...} にマッチしたらこっち
def handle_call(:get_stations, _from, %{stations: nil, fetch_fun: fetch_fun} = state) do
# ...
# 状態が %{stations: stations} にマッチしたら（nilじゃなければ）こっち
def handle_call(:get_stations, _from, %{stations: stations} = state) do
```
状態が `nil` かどうかの `if` 文を書かなくて済むのが、パターンマッチの強力なところです。
※順番が重要です。下の節を上に書くと、`nil` の時もそちらにマッチしてしまいます。

### 5. 依存性注入：関数を引数で渡す `&Mod.fun/arity`

テストの章で大活躍するこの仕組み、じっくり解説します。

```elixir
fetch_fun = Keyword.get(opts, :fetch_fun, &WeatherEx.Weather.JmaClient.fetch_stations_master/0)
```

`&WeatherEx.Weather.JmaClient.fetch_stations_master/0` という奇妙な記法は、**「関数そのものを変数としてキャプチャする」**記法です。
JSのアロー関数 `() => JmaClient.fetch_stations_master()` に近いですが、もっと直接的です。

もし `()` をつけて `JmaClient.fetch_stations_master()` と書いてしまうと、「関数を**実行した結果**（つまり `{:ok, %{...}}` というデータ）」が渡されてしまいます。
`&.../0` と書くと、「まだ実行していない関数そのもの」を渡せます。

これにより、本番環境では本物のネットワーク通信関数を渡し、テスト環境では「常に `{:ok,ダミーデータ}` を返すだけの無名関数」を渡すことができます。これが**依存性注入（DI）**です。Elixirはクラスがないので、インターフェースではなく「関数」を直接注入して差し替えるのがエレガントなのです。

### 6. Application と Supervisor：自動復帰するシステム

```elixir
children = [WeatherEx.Weather.StationCache]
opts = [strategy: :one_for_one, name: WeatherEx.Supervisor]
Supervisor.start_link(children, opts)
```

`children` にモジュールを書くだけで、アプリ起動時に `start_link/1` が呼ばれます。
そして `:one_for_one` という戦略が、OTPの真骨頂です。
もし `StationCache` プロセスの中でバグが起きてプロセスがクラッシュ（死）したとします。Node.jsならサーバー全体が落ちて再起動が必要になる場面でも、ElixirのSupervisorは**「あ、子どもが死んだ。一人だけ生き返らせよう」**と自動的に `start_link` を再実行し、システムを継続させます。これがElixirアプリケーションの圧倒的な耐障害性の源です。

---

## 動作確認: iexで叩く

```bash
iex -S mix
```

`iex -S mix` を立ち上げた時点で、裏側で `StationCache` プロセスが既に動いています。

```elixir
iex> alias WeatherEx.Weather.StationCache

# 1回目：ネットワーク通信が発生するため少し待つ
iex> StationCache.get_stations()
{:ok, %{"47759" => %{...}, "47646" => %{...}, ...}}

# 2回目：キャッシュから返るため、瞬時に返るのが体感できる
iex> StationCache.get_stations()
{:ok, %{"47759" => %{...}, "47646" => %{...}, ...}}

iex> StationCache.get_station("47759")
{:ok, %{"kjName" => "東京", ...}}

# 存在しないID
iex> StationCache.get_station("00000")
:error
```

---

## テストコード: `test/weather_ex/weather/station_cache_test.exs`

ここで、第5の着眼点で説明した「依存性注入」が本領を発揮します。
**実際にネットワーク通信を発生させずに、GenServerのロジックだけをテスト**します。

```elixir
defmodule WeatherEx.Weather.StationCacheTest do
  use ExUnit.Case, async: true
  alias WeatherEx.Weather.StationCache

  # テスト用のヘルパー関数：ダミーのfetch_funを注入して起動する
  defp start_cache(fetch_fun) do
    # name: nil にすることで、テストごとに独立した「無名プロセス」として起動する
    {:ok, pid} = StationCache.start_link(name: nil, fetch_fun: fetch_fun)
    pid
  end

  test "初回のget_stationsでfetch_funを実行し、結果を返す" do
    # ダミーデータを返すだけの関数を注入
    fetch_fun = fn -> {:ok, %{"47759" => %{"kjName" => "東京"}}} end
    pid = start_cache(fetch_fun)

    # 注入した pid を指定して呼び出す
    assert {:ok, %{"47759" => %{"kjName" => "東京"}}} = StationCache.get_stations(pid)
  end

  test "2回目以降はfetch_funを再実行せずキャッシュを返す" do
    # 呼ばれた回数を数えるための「状態を持つ小さなプロセス(Agent)」を準備
    {:ok, counter} = Agent.start_link(fn -> 0 end)

    fetch_fun = fn ->
      # 呼ばれるたびにカウントを増やす
      Agent.update(counter, &(&1 + 1))
      {:ok, %{"47759" => %{"kjName" => "東京"}}}
    end

    pid = start_cache(fetch_fun)

    # 3回呼び出す
    StationCache.get_stations(pid)
    StationCache.get_stations(pid)
    StationCache.get_stations(pid)

    # カウンタが1のままである＝fetch_funは1回しか呼ばれていない（キャッシュされている）
    assert Agent.get(counter, & &1) == 1
  end

  test "fetch_funが失敗を返す場合、エラーを返しキャッシュしない" do
    {:ok, counter} = Agent.start_link(fn -> 0 end)

    fetch_fun = fn ->
      Agent.update(counter, &(&1 + 1))
      {:error, :network_error}
    end

    pid = start_cache(fetch_fun)

    assert {:error, :network_error} = StationCache.get_stations(pid)
    assert {:error, :network_error} = StationCache.get_stations(pid)
    
    # 失敗時はキャッシュされないので、毎回fetch_funが呼ばれてカウントが2になる
    assert Agent.get(counter, & &1) == 2
  end

  test "get_station/2で存在しないIDは:errorを返す(Map.fetchの挙動)" do
    fetch_fun = fn -> {:ok, %{"47759" => %{"kjName" => "東京"}}} end
    pid = start_cache(fetch_fun)

    assert :error = StationCache.get_station(pid, "99999")
  end
end
```

### なぜ `name: nil` なのか？

`StationCache.start_link()` のデフォルトでは、プロセスに `WeatherEx.Weather.StationCache` という名前がつきます。
しかしテストでは `async: true` によって複数のテストが同時に走ります。もし同名のプロセスを2つ起動しようとすると、`{:error, {:already_started, pid}}` というエラーになってしまいます。
`name: nil` で起動すると「名前を持たない無名プロセス」となり、戻り値の `pid`（プロセスID）を使ってしかアクセスできなくなるため、テスト同士が衝突しなくなります。

---

## つまずきやすい点

- **GenServerの状態は直接書き換えられない**。`state.stations = data` のような代入文はElixirには存在しません。必ず `%{state | key: new_value}` で新しいマップを作り、返り値の3つ目の要素として返します。
- **`handle_call` の複数節の順番**。`nil` チェックの節を下に書くと、上の「なんでも受け取る節」にマッチしてしまい、永遠にデータを取得しに行かないバグになります。
- **`&Mod.fun/0` の `/0` を忘れないこと**。これは「引数が0個の関数」というアリティ（引数の数）の指定です。これがないとコンパイルエラーになります。
- **テストでカウンタに `Agent` を使うのはElixir流**。モックライブラリ（Mox等）を使わなくても、標準機能の `Agent` と「関数を渡す」仕組みだけで、副作用の検証が美しく書けます。

---

## 章末チェックリスト

- [ ] 「なぜElixirは変数ではなくプロセスで状態を持つのか」を理解した
- [ ] `lib/weather_ex/weather/station_cache.ex` を作成し、Client API と Server Callbacks が分かれていることを確認した
- [ ] `jma_client.ex` に `fetch_stations_master/0` を追加した
- [ ] `lib/weather_ex/application.ex` の `children` に `StationCache` を登録した
- [ ] `handle_call` で `%{state | stations: stations}` を使って新しい状態を返していることを理解した
- [ ] `test/weather_ex/weather/station_cache_test.exs` を作成し、`name: nil` とダミー関数の注入でテストが通ることを確認した
- [ ] `iex -S mix` で2回連続で `get_stations()` を叩き、2回目が瞬時に返る（キャッシュされている）ことを体感した

---

前章: [第2章 — 外部API連携(jma_client.ex)](./chapter02_jma_client.md)
次章: 第4章 — Context層で統合(weather.ex)
