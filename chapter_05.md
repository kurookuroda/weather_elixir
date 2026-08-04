# 第5章: LiveViewの骨組み（Elixir入門・深掘り編）

第4章までで、Elixirの世界における「データ変換」「通信」「状態保持」「統合」のパラダイムを学んできました。いよいよ**初めてWeb層（フロントエンド）に足を踏み入れます**。

しかし、ここでVueやReactのような「クライアントサイドJavaScript」の感覚でコードを書くと、必ず壁にぶつかります。なぜならPhoenix LiveViewは、見た目はSPAのように動きますが、その正体は**「サーバー上の軽量プロセスが、HTMLの差分をネットワーク経由で押し付けてくる仕組み」**だからです。

この章では、ただ「時計とカウントダウンを作る」だけでなく、**なぜLiveViewがそのような設計になっているのか、裏側でプロセスがどう動いているのか**を、Vueの `setInterval` や `ref` と比較しながら深く掘り下げていきますにゃ。

---

## この章でじっくり学ぶこと

- **LiveViewの2段階レンダリング**: なぜ最初に静的HTMLを配信し、後からWebSocketで差分を送るのか（SEOとパフォーマンスの両立）
- **`socket` の正体**: 魔法のオブジェクトではなく、「プロセスの状態を運ぶただのデータ構造」であること
- **`assign` の裏側**: Vueの `Proxy` によるリアクティブ化と、Elixirの「新しいマップを作る」アプローチの決定的な違い
- **Actorモデルにおけるタイマー**: なぜ `setInterval`（無限ループ）ではなく `Process.send_after`（1回限りのメッセージ送信）なのか
- **プロセスの透明性**: テストコードでなぜライブなプロセスに直接メッセージを送れるのか

---

## 移植対象: 元のVueコードの「見えない前提」

```typescript
// Vueでは「ブラウザの中」で全てが完結する
const currentTimeText = ref('LOADING...')
let countdown = 300

// ブラウザのAPI（Windowオブジェクト）がタイマーを管理する
tickInterval = setInterval(() => {
  countdown-- // メモリ上の変数を直接書き換える（ミュータブル）
  updateTimerText.value = `${m}:${String(s).padStart(2, "0")}`
}, 1000)
```

VueやReactの前提には**「ユーザーのブラウザ内にメモリがあり、そこで変数が書き換わる」**という世界があります。
しかしLiveViewでは、**ユーザーのブラウザには「状態」はありません。** 状態はすべて遠く離れたサーバー上のプロセスの中にあり、ブラウザは単に「今の状態を表すHTML」を受け取って描画しているだけです。

この「状態のある場所」の違いを意識しながら、実装に入りましょう。

---

## Step 0: ルーティングの仕組み

`lib/weather_ex_web/router.ex` を開き、ルーティングを追加します。

```elixir
scope "/", WeatherExWeb do
  pipe_through :browser

  get "/", PageController, :home        # ← 既存（従来のController）
  live "/weather", WeatherLive          # ← 追加（LiveView）
end
```

**深掘りポイント**: `get "/"` は「リクエストが来たらHTMLを生成して返す」という1回きりの処理です。一方 `live "/weather"` は「接続を確立し、継続的な通信路（WebSocket）を維持する」という宣言になります。Phoenixのルーターは、この「1回きりか、継続的か」をシームレスに扱うことができますにゃ。

---

## Step 1: `mount/3` と2段階レンダリングの深掘り

`lib/weather_ex_web/live/weather_live.ex` を新規作成します。

```elixir
defmodule WeatherExWeb.WeatherLive do
  use WeatherExWeb, :live_view

  alias WeatherEx.Weather

  @update_interval 300  # 5分 = 300秒
  @tick_interval 1_000  # 1秒（ミリ秒）

  @impl true
  def mount(_params, _session, socket) do
    if connected?(socket) do
      # --- パターンB: WebSocket接続後 ---
      schedule_tick()
      {:ok, schedule_update(socket)}
    else
      # --- パターンA: 初回HTTPリクエスト時（静的レンダリング） ---
      {:ok, assign(socket,
        current_time: format_time(DateTime.utc_now()),
        countdown: @update_interval,
        records: [],
        loading: true
      )}
    end
  end
end
```

### なぜ `mount` は2回呼ばれるのか？

Vueの `onMounted` は、ブラウザでDOMが構築された時に**1回だけ**呼ばれます。
しかしLiveViewの `mount/3` は、以下の2回呼ばれる設計になっています。

1. **初回のHTTPリクエスト時 (`connected?` が `false`)**
   - サーバーはWebSocketを張る前の「素のHTML」を返す必要があります。これはSEO（Googleのクローラーが読める）ためであり、またWebSocketが繋がるまでの「ローディング画面」として機能します。ここで重いAPI通信やタイマーを走らせると、サーバーのリソースを無駄に消費してしまいます。
2. **WebSocket接続が確立した直後 (`connected?` が `true`)**
   - ここで初めて「このユーザー専用のプロセス」が誕生し、状態管理やタイマーが開始されます。

**SPA（Vue/React）のデメリットを解消するための設計**です。SPAは初期ロード時に巨大なJSをダウンロードし、その後JSが動いてデータを fetched するため、初回表示が遅く、SEOも弱いです。LiveViewの2段階レンダリングは、伝統的なサーバーサイドレンダリングの「速さとSEO」を保ちつつ、接続後はSPAのようなインタラクティブ性を得るという、両者のいいとこ取りをしているのですにゃ。

---

## Step 2: `Process.send_after` とActorモデルのタイマー

時計を動かす本体です。

```elixir
  @impl true
  def handle_info(:tick, socket) do
    new_countdown = socket.assigns.countdown - 1

    socket =
      if new_countdown <= 0 do
        socket
        |> fetch_weather()
        |> assign(countdown: @update_interval)
      else
        assign(socket, countdown: new_countdown)
      end

    # ★ 次の1回をスケジュール
    schedule_tick()

    {:noreply, assign(socket, current_time: format_time(DateTime.utc_now()))}
  end

  defp schedule_tick do
    Process.send_after(self(), :tick, @tick_interval)
  end
```

### なぜ `setInterval`（無限ループ）ではなく `send_after`（1回限り）なのか？

ここがElixirの**Actorモデル**の根幹に関わる考え方です。

JavaScriptの `setInterval` は、ブラウザ（グローバルな実行環境）の裏側で「時計のプロセス」を別に立ち上げ、それが勝手にあなたの関数を定期的に呼び出します。あなたはその「時計」を直接コントロールしておらず、`clearInterval` で破壊するしかありません。

Elixirの世界では、**「自分以外の誰も、自分の状態を勝手に変えることはできない」**という絶対のルールがあります。
もし `setInterval` のような「外部から勝手に実行されるループ」があったら、それはこのルールに反します。

だからElixirでは、「1秒後に**自分自身（`self()`）**に `:tick` というメッセージを送ってくれ」とお願いします（`Process.send_after`）。
メッセージは自分のプロセスの**メールボックス**にポストされ、順番に処理されます。処理が終わったら、また「1秒後にメッセージを送ってくれ」とお願いする（`schedule_tick()`）。

この**「自分で自分にメッセージを送り続ける明示的なループ」**によって、タイマーのライフサイクルを完全に自分のプロセス内でコントロールしています。ユーザーがブラウザを閉じればプロセスが死に、勝手に動いているタイマーも消滅するため、メモリリークが起きないのですにゃ。

---

## Step 3: データ取得と `assign` の正体

```elixir
  defp fetch_weather(socket) do
    case Weather.fetch_pressure_diffs() do
      {:ok, records} ->
        assign(socket, records: records, loading: false, error: nil)
      {:error, reason} ->
        assign(socket, error: reason, loading: false)
    end
  end

  defp format_time(%DateTime{} = dt) do
    dow = ["日", "月", "火", "水", "木", "金", "土"] |> Enum.at(Date.day_of_week(dt) - 1)
    Calendar.strftime(dt, "%Y年%m月%d日(#{dow}) %H:%M:%S")
  end

  defp format_countdown(seconds) do
    m = div(seconds, 60)
    s = rem(seconds, 60)
    "#{m}:#{String.pad_leading("#{s}", 2, "0")}"
  end
```

### `socket` と `assign` の正体

Vueでは、状態を `ref` や `reactive` で包むと、裏側で `Proxy` という仕組みが働き、値が変わったことをフレームワークが検知（トラッキング）してくれます。

LiveViewの `socket` は、魔法のトラッキング機能を持ったオブジェクトではありません。
`socket` は単なる**Map（データ構造）**です。その中の `assigns` というキーに、現在の画面の状態が入っているだけです。

```elixir
%Phoenix.LiveView.Socket{
  assigns: %{
    current_time: "...",
    countdown: 300,
    records: []
  },
  ...
}
```

`assign(socket, countdown: 299)` を呼ぶと、Elixirは**「古い socket の `assigns` にある `countdown` を `299` に差し替えた、新しい Map」**を作って返します（第3章のGenServerの状態更新と全く同じです）。

では、何が「変更を検知」しているのでしょうか？
実はLiveViewは、`handle_info` が `{:noreply, new_socket}` を返した瞬間に、「あ、新しいsocketが返ってきたから、前のsocketと比較（Diff）して、変わってたらブラウザに送ろう」と動きます。**データの追跡をフレームワークの裏側の仕組みに依存するのではなく、関数の入出力（古いsocketと新しいsocket）の差分として変更を捉える**のが、Elixirのデータ駆動的なアプローチですにゃ。

> **注意**: Elixirの `Calendar.strftime/2` には、JSの `toLocaleDateString` にあるような `%-m`（ゼロパディングなし）という修飾子は存在しません。Erlangの標準ライブラリに依存しているため、利用可能な書式は公式ドキュメントを確認する必要がありますにゃ。

---

## Step 4: HEExテンプレート（テキスト版）

`lib/weather_ex_web/live/weather_live.html.heex` を新規作成します。

```heex
<div class="p-8 max-w-2xl mx-auto font-sans">
  <h1 class="text-2xl font-black text-slate-800 mb-4 flex items-center gap-2">
    <span>🌡️</span> 気圧変化アラート
  </h1>

  <div class="bg-white rounded-2xl shadow-lg border border-slate-200 p-6 mb-6">
    <div class="text-blue-600 font-mono font-bold text-xl tracking-tight">
      <%= @current_time %>
    </div>
    <div class="text-slate-400 font-mono text-xs font-bold tracking-widest uppercase mt-2">
      UPDATE IN: <%= format_countdown(@countdown) %>
    </div>
  </div>

  <%= if @loading do %>
    <div class="text-slate-500 text-sm animate-pulse">データを取得中...</div>
  <% end %>

  <%= if @error do %>
    <div class="bg-red-50 text-red-600 p-4 rounded-xl text-sm">
      エラー: <%= inspect(@error) %>
    </div>
  <% end %>

  <div class="space-y-2">
    <%= for record <- @records do %>
      <div class="bg-white rounded-xl shadow border border-slate-100 p-4 flex justify-between items-center">
        <div>
          <div class="font-bold text-slate-800"><%= record.name %></div>
          <div class="text-2xl leading-none mt-1"><%= record.weather %></div>
        </div>
        <div class="text-right font-mono">
          <div class="text-2xl font-black text-slate-800">
            <%= record.slp %><span class="text-xs font-bold text-slate-500 ml-1">hPa</span>
          </div>
          <%# class={[...]} は Phoenix 1.7+ の機能。条件が true のクラスだけがマージされる（Vueの :class に近い） %>
          <div class={[
            "text-sm font-black",
            record.diff < 0 && "text-red-500",
            record.diff >= 0.5 && "text-blue-500",
            record.diff == nil && "text-slate-400"
          ]}>
            <%= if record.diff != nil && record.diff > 0, do: "+", else: "" %><%= record.diff %>
          </div>
        </div>
      </div>
    <% end %>
  </div>

  <div class="mt-6 text-xs text-slate-400 text-center">
    <%= length(@records) %> 地点のデータを表示中
  </div>
</div>
```

### HEExの深掘り：ただの文字列補間ではない

Vueのテンプレート（`.vue` ファイル）は、最終的にJavaScriptの `render` 関数にコンパイルされ、ブラウザ上でVirtual DOM（VNode）を生成します。

HEEx（`.heex` ファイル）は、**コンパイル時（サーバー起動時）にElixirの関数に変換されます。**
しかも単なる文字列連結ではなく、「どの `assign` の値が、どのHTMLノードに対応するか」という**ツリー構造のマッピング情報**を内部に持ちます。
これにより、`handle_info` で新しいsocketが返ってきた時、LiveViewは「前回のツリー」と「今回のツリー」を比較し、変更があったDOMノード**だけ**をJavaScript経由でブラウザに送信します。
`<%= for record <- @records do %>` で回している部分でも、レコードの順番が変わったり1件追加されただけなら、リスト全体を送り直すのではなく、最小限の差分（例えば `<li>` の追加）だけがネットワークを流れますにゃ。

---

## 動作確認

```bash
mix phx.server
```

ブラウザで `http://localhost:4000/weather` を開きます。

1. **最初に静的HTMLが表示される**（`mount`の`connected?`が`false`の時の状態）
2. **WebSocket接続後、時計が動き出す**（`:tick`メッセージが回り始める）
3. **カウントダウンが1秒ごとに減少**し、0になるとデータが更新される

開発者ツールのNetworkタブでWebSocket（`ws://`）の通信を覗くと、時刻更新のたびに**「 diffs 」というごく小さなJSONデータ（差分だけ）が送られている**のが見えるはずです。これがLiveViewの「サーバーサイドレンダリング + 差分更新」の仕組みにゃ。

---

## テストコード：プロセスの透明性を体験する

`test/weather_ex_web/live/weather_live_test.exs`:

```elixir
defmodule WeatherExWeb.WeatherLiveTest do
  use WeatherExWeb.ConnCase, async: true
  import Phoenix.LiveViewTest

  test "マウント時に時刻とカウントダウンが表示される", %{conn: conn} do
    {:ok, _view, html} = live(conn, "/weather")

    assert html =~ "気圧変化アラート"
    assert html =~ "UPDATE IN:"
  end

  test ":tickメッセージでカウントダウンが減少する", %{conn: conn} do
    {:ok, view, _html} = live(conn, "/weather")

    # 初期カウントダウンは300
    assert render(view) =~ "UPDATE IN: 5:00"

    # 直接:tickメッセージを送信（1秒待たずにテスト）
    send(view.pid, :tick)
    html = render(view)

    # 299秒 = 4:59
    assert html =~ "UPDATE IN: 4:59"
  end

  test "カウントダウン0でデータ更新がトリガーされる", %{conn: conn} do
    {:ok, view, _html} = live(conn, "/weather")

    # assignを直接書き換えてカウントダウンを1にする（テスト用ショートカット）
    :sys.replace_state(view.pid, fn state ->
      %{state | assigns: %{state.assigns | countdown: 1}}
    end)

    # tickを送ると0になり、Weather.fetch_pressure_diffs/0 が呼ばれる
    send(view.pid, :tick)
    html = render(view)

    # 実際にAPIからデータが取れていれば気圧値が含まれる
    assert html =~ "hPa"
  end
end
```

### なぜテストで `send(view.pid, :tick)` できるのか？

Vueのテストで「コンポーネントの内部タイマーを1秒進める」のは非常に難しく、jestの `vi.useFakeTimers()` のようなモック魔法を使う必要があります。

しかしElixirでは、LiveViewがただの**「メッセージを受け取るGenServerプロセス」**であるという事実が、すべてをシンプルにします。
`view.pid` はそのLiveViewプロセスのID（PID）です。テストコードから直接 `send( PID, :メッセージ )` と送れば、あたかも1秒経過したかのように `handle_info` が即座に呼ばれます。

さらに `:sys.replace_state/2` は、テスト専用の裏口からプロセスの中身（socketの状態）を直接書き換える関数です。「カウントダウンを1にするために299回メッセージを送る」という非現実的なテストを避け、意図通りにロジックを検証できます。
**「プロセスは暗箱ではなく、メッセージで通信し、状態を覗くことができる」**というBEAM VMの透明性が、テストの容易性に直結しているのですにゃ。

> **補足**: 第4章でContextに依存性注入（DI）を仕込んだけど、LiveViewモジュール内では `Weather.fetch_pressure_diffs()` とハードコードしているため、テスト時にDIを流し込むにはMoxライブラリやProcess dictionaryの活用が必要になる。第5章のスコープを超えるので、ここでは「カウントダウン0でAPIコールが走る」ことだけを検証するにゃ。本番相当のテストは第6章以降で別途整備する。

---

## つまずきやすい点

- **`mount/3`が2回呼ばれる**: 最初のHTTPリクエスト時（`connected?`が`false`）と、WebSocket接続後（`true`）の2回。タイマー開始や重い処理は`connected?`が`true`の時だけ行うのを徹底しないと、無駄な処理が走ったり競合したりするにゃ
- **`Process.send_after`は1回だけ**: `setInterval`のように勝手に繰り返してくれない。`handle_info`の最後に必ず`schedule_tick()`を呼んで再予約する。忘れると1回だけ更新されて止まってしまうにゃ
- **タイムゾーン**: `DateTime.utc_now()`はUTC。日本時間（JST）に変換したい場合は`DateTime.shift_zone!/2`を使うか、表示フォーマットで調整する。気象庁APIの時刻もJST前提なので、統一しておくと後の章で楽になるにゃ
- **`assign/3`はマージする**: 同じキーを`assign`し直すと上書きされるが、存在しないキーは消えない。Vueの`reactive`オブジェクトとは挙動が異なるので注意にゃ
- **ブラウザのリロード**: LiveViewはWebSocket接続が切れると自動的に再接続を試みる（`phx-disconnected`クラスが付く）。再接続後も`mount/3`が呼ばれるので、状態の初期化はそこで完結させるにゃ
- **`Calendar.strftime`の修飾子**: JSの`toLocaleDateString`とは異なり、`%-m`（ゼロパディングなし）のような修飾子は使えない。Elixirの`Calendar.strftime`はErlangの`strftime`実装に依存しているので、利用可能な書式は[Elixir公式ドキュメント](https://hexdocs.pm/elixir/Calendar.html#strftime/3)を確認するにゃ

---

## 章末チェックリスト

- [ ] 「状態はブラウザになく、サーバーのプロセスにある」というLiveViewの前提を理解した
- [ ] `lib/weather_ex_web/live/weather_live.ex` を作成（`mount`/`handle_info`/`assign`）
- [ ] なぜ `mount` が2回呼ばれ、`connected?` で分岐するのかを理解した
- [ ] `Process.send_after` が「自分自身へのメッセージ送信」であることを理解した
- [ ] `lib/weather_ex_web/live/weather_live.html.heex` を作成（テキスト表示版）
- [ ] `lib/weather_ex_web/router.ex` に `live "/weather", WeatherLive` を追加
- [ ] `test/weather_ex_web/live/weather_live_test.exs` を作成し、`send` による直接メッセージ送信を体験した
- [ ] `mix test` が通過
- [ ] `mix phx.server` でブラウザ確認（時計が動き、カウントダウンが減る）
- [ ] カウントダウン0で `Weather.fetch_pressure_diffs/0` が呼ばれデータが更新されることを確認

---

前章: [第4章 — Context層で統合(weather.ex)](./chapter04_weather_context.md)
次章: [第6章 — 中間デプロイチェックポイント①](./chapter06_deploy_check.md)
