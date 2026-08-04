# 第8章: Geolocation往復（`pushEvent`と`push_event`の双方向通信）

第7章では「サーバー→ブラウザ」という一方通行の`push_event`だけを扱いました。今回はいよいよ**双方向**です。「現在地を特定」ボタンを押すと、ブラウザの位置情報APIを叩き、取れた座標で地図を`flyTo`する——元のVueコードで`locateUser`という1つの関数に収まっていた処理を、LiveViewでは**サーバーとブラウザを何度か往復するメッセージのやり取り**として設計し直します。

「たかがボタン1つになぜそこまで往復させるのか」と思うかもしれませんが、これは**LiveViewというアーキテクチャそのものの縮図**にゃ。ボタンの見た目(ローディング表示)を管理するのはサーバー、位置情報の取得と地図操作を担うのはブラウザ——**「状態はどこにあるべきか」を1つの機能の中で仕分ける**、この章がその総仕上げになります。

---

## この章でじっくり学ぶこと

- **`phx-click`と`handle_event`**: ボタン操作をサーバーに伝える、LiveViewの最も基本的な双方向通信
- **`this.pushEvent`**: JS Hook側からサーバーへイベントを送る、第7章の`push_event`の逆方向
- **`pushEvent`のコールバック引数**: サーバーからの応答を受け取る、擬似的な「往復リクエスト」の作り方
- **状態の置き場所の使い分け**: 「ボタンの見た目」はサーバー側の`assign`、「地図の操作」はブラウザ側のHook、と役割を分担する設計判断
- **ブラウザAPIのエラーハンドリング**: 位置情報の許可拒否・タイムアウトを、サーバー側の状態にどう反映するか

---

## 移植対象: 元のVueコードの「見えない前提」

```typescript
const isLocating = ref(false)
let userLocationMarker: any = null

async function locateUser() {
  isLocating.value = true
  locateBtnText.value = "取得中..."

  navigator.geolocation.getCurrentPosition(
    (pos) => {
      const { latitude, longitude } = pos.coords
      map.flyTo([latitude, longitude], 10)

      if (userLocationMarker) map.removeLayer(userLocationMarker)
      userLocationMarker = L.circleMarker([latitude, longitude], { color: "blue" }).addTo(map)

      isLocating.value = false
      locateBtnText.value = "現在地を特定"
    },
    (err) => {
      alert("位置情報の取得に失敗しました")
      isLocating.value = false
      locateBtnText.value = "現在地を特定"
    }
  )
}
```

Vueのこの関数は、**「ボタンの状態」も「位置情報の取得」も「地図の操作」も、全部同じ関数の中で完結**しています。`isLocating`というただのブール値の`ref`を切り替えるだけで、ボタンの見た目もdisabled状態も自動的に追従します。

LiveViewでは、この「全部同じ場所」という前提が崩れます。**ボタンの見た目(テキストやdisabled属性)を決めるHTMLはサーバーが生成する**一方、**位置情報の取得(`navigator.geolocation`)と地図の操作(`flyTo`)は、ブラウザでしか実行できません**。1つの関数を、サーバー側とブラウザ側の2つの「引き出し」に仕分ける作業が必要になります。

---

## Step 1: ボタンをLiveView管理下に置く

まず地図とは切り離して、**ボタン単体をサーバー管理のUIとして作ります**。`weather_live.html.heex`にボタンとエラーメッセージ表示を追加します。

```heex
<div class="p-4 max-w-7xl mx-auto font-sans space-y-4">
  <%--- テキストUI（上部） ---%>
  <div class="bg-white rounded-2xl shadow-lg border border-slate-200 p-6">
    <h1 class="text-2xl font-black text-slate-800 mb-2 flex items-center gap-2">
      <span>🌡️</span> 気圧変化アラート
    </h1>
    <div class="text-blue-600 font-mono font-bold text-xl tracking-tight">
      <%= @current_time %>
    </div>
    <div class="text-slate-400 font-mono text-xs font-bold tracking-widest uppercase mt-1">
      UPDATE IN: <%= format_countdown(@countdown) %>
    </div>

    <%--- 現在地ボタン（第8章で追加） ---%>
    <button
      phx-click="locate_user"
      disabled={@locating}
      class="mt-3 w-full bg-blue-50/50 hover:bg-blue-100 text-slate-500 text-xs font-bold py-2 px-3 rounded-xl border border-slate-200 shadow-sm transition-all disabled:opacity-50 disabled:cursor-not-allowed"
    >
      <%= if @locating, do: "取得中...", else: "📍 現在地を特定" %>
    </button>

    <%--- 位置情報エラー表示（第8章で追加） ---%>
    <%= if @location_error do %>
      <div class="mt-2 text-xs text-red-500 font-bold bg-red-50 p-2 rounded-lg">
        <%= @location_error %>
      </div>
    <% end %>
  </div>

  <%--- 地図（第7章と同じ） ---%>
  <div
    id="weather-map"
    phx-hook="WeatherMap"
    phx-update="ignore"
    class="h-[70vh] w-full rounded-2xl border border-slate-200"
  ></div>
</div>
```

`weather_live.ex`の`mount/3`に初期値を追加し、`handle_event/3`でクリックを受け取ります。

```elixir
def mount(_params, _session, socket) do
  if connected?(socket) do
    schedule_tick()
    {:ok, socket |> assign(locating: false, location_error: nil) |> schedule_update()}
  else
    {:ok, assign(socket,
      current_time: format_time(DateTime.utc_now()),
      countdown: @update_interval,
      records: [],
      loading: true,
      locating: false,
      location_error: nil
    )}
  end
end

@impl true
def handle_event("locate_user", _params, socket) do
  {:noreply,
   socket
   |> assign(locating: true, location_error: nil)
   |> push_event("start_geolocation", %{})}
end
```

**深掘りポイント**: `phx-click="locate_user"`は、ボタンがクリックされた瞬間にサーバーへ`"locate_user"`という名前のメッセージを送ります。これはWebSocket経由の通常のLiveViewイベントで、第7章までの`push_event`/`handleEvent`とは**向きが逆**であることに注意してくださいにゃ。Vueで言えば`@click="locateUser"`とほぼ同じ書き味ですが、実行される場所がブラウザではなくサーバー側の`handle_event/3`だという点が違います。

`assign(locating: true, location_error: nil)`でボタンのdisabled状態とテキストを即座に切り替えつつ、同じsocketに`push_event("start_geolocation", %{})`を積んで「ブラウザさん、位置情報を取ってきて」と依頼を投げています。

---

## Step 2: Hookで位置情報を取得し、サーバーへ送り返す

`assets/js/hooks/weather_map.js`の`mounted()`に、依頼を受け取るハンドラを追加します。

```javascript
mounted() {
  this.map = L.map(this.el).setView([35.5, 137.5], 6)

  L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
    attribution: "&copy; OpenStreetMap contributors",
  }).addTo(this.map)

  this.markersLayer = L.layerGroup().addTo(this.map)
  this.userLocationMarker = null

  this.handleEvent("update_markers", ({ records }) => {
    this.renderMarkers(records)
  })

  // サーバーからの「取得して」依頼を受け取る
  this.handleEvent("start_geolocation", () => {
    this.locateUser()
  })
},

locateUser() {
  if (!navigator.geolocation) {
    this.pushEvent("location_error", { reason: "unsupported" })
    return
  }

  navigator.geolocation.getCurrentPosition(
    (pos) => {
      const { latitude, longitude } = pos.coords
      // サーバーへ座標を送り返す(JS → LiveView)
      this.pushEvent("location_found", { lat: latitude, lon: longitude })
    },
    (err) => {
      let reason = "timeout"
      switch (err.code) {
        case err.PERMISSION_DENIED:
          reason = "denied"
          break
        case err.POSITION_UNAVAILABLE:
          reason = "unavailable"
          break
        case err.TIMEOUT:
          reason = "timeout"
          break
      }
      this.pushEvent("location_error", { reason })
    },
    { enableHighAccuracy: true, timeout: 10_000, maximumAge: 0 }
  )
},
```

### `this.pushEvent`の書式

```javascript
this.pushEvent(eventName, payload, callback)
```

第3引数の`callback`は省略可能です。サーバー側で`handle_event/3`が`{:reply, map, socket}`を返すと、その`map`がこの`callback`に渡されます。VueやJSの`fetch`のような「投げて、返事を待つ」構文に近い書き味を、Hookでも実現できる仕組みです。今回はサーバー側からの応答を**別イベント(`push_event`)として非同期に送り返す**設計にしているため、`callback`は使わずシンプルな片道送信×2として組んでいます(理由はStep 4の深掘りで説明します)。

---

## Step 3: サーバーで座標を受け取り、地図へ`flyTo`を指示する

`weather_live.ex`に、Hookから送られてくる2種類のイベントを受け取る`handle_event/3`を追加します。

```elixir
@impl true
def handle_event("location_found", %{"lat" => lat, "lon" => lon}, socket) do
  {:noreply,
   socket
   |> assign(locating: false)
   |> push_event("center_map", %{lat: lat, lon: lon})}
end

@impl true
def handle_event("location_error", %{"reason" => reason}, socket) do
  message =
    case reason do
      "denied" -> "位置情報の利用が許可されていません。ブラウザの設定を確認してください。"
      "unavailable" -> "現在地を取得できませんでした。電波状況の良い場所で再度お試しください。"
      "unsupported" -> "お使いのブラウザは位置情報に対応していません。"
      _ -> "位置情報の取得がタイムアウトしました。再度お試しください。"
    end

  {:noreply, socket |> assign(locating: false, location_error: message)}
end
```

`assign(locating: false)`でボタンを元の状態に戻しつつ、成功時は`push_event("center_map", ...)`で「今度はあなた(ブラウザ)の番、この座標に飛んでください」とお願いを投げ返しています。

---

## Step 4: Hookで地図をflyToし、現在地マーカーを置く

`weather_map.js`の`mounted()`に、最後の受け取り役を追加します。

```javascript
this.handleEvent("center_map", ({ lat, lon }) => {
  this.map.flyTo([lat, lon], 10, { animate: true, duration: 1.5 })

  if (this.userLocationMarker) {
    this.map.removeLayer(this.userLocationMarker)
  }
  this.userLocationMarker = L.circleMarker([lat, lon], {
    radius: 8,
    color: "#2563eb", // blue-600
    fillOpacity: 1,
  }).addTo(this.map)
})
```

これで一連の往復が完成しました。全体の流れを図にすると次のようになりますにゃ。

```
[ブラウザ]                          [サーバー(LiveViewプロセス)]
  ボタンclick
   └─ phx-click="locate_user" ──────▶ handle_event("locate_user")
                                        assign(locating: true)
   ◀── push_event("start_geolocation") ┘
  navigator.geolocation 実行
   └─ pushEvent("location_found") ──▶ handle_event("location_found")
                                        assign(locating: false)
   ◀── push_event("center_map") ───────┘
  map.flyTo() 実行
```

### なぜ「1往復のリクエスト/レスポンス」ではなく「2つの片道メッセージ」を2セット使うのか

Step 2で触れた`this.pushEvent`の第3引数`callback`を使えば、「`location_found`を送ったら`center_map`相当の返事を直接受け取る」という1往復にまとめることもできます。しかし今回あえて「ボタンのクリック」と「位置情報の結果」を**別々の独立したメッセージ**として設計しました。理由は、**サーバー側のLiveViewプロセスが、常に「今の正しい状態」を把握している場所であるべき**だからです。

もし`callback`だけで完結させると、「ボタンが押された」という事実そのものがサーバーの`assign`に記録されず、**ブラウザ側だけが知っている一時的な状態**になってしまいます。例えば「現在地取得中に別の場所からこのページを開いた人には、ボタンがローディング中に見えない」といった不整合の原因になります。`assign(locating: true/false)`をサーバー側に必ず経由させることで、**「今、位置情報を取得中かどうか」という事実がサーバープロセスの中に一元化**され、複数のブラウザタブや将来のマルチユーザー機能拡張にも耐えられる設計になっていますにゃ。

---

## Step 5: 動作確認

```bash
mix phx.server
```

`http://localhost:4000/weather`をブラウザで開き、以下を確認します。

1. 「📍 現在地を特定」ボタンを押すと、即座に「取得中...」に変わりボタンが押せなくなる
2. ブラウザの位置情報許可ダイアログが出る(初回のみ)
3. 許可すると、地図が現在地へ`flyTo`し、青い丸マーカーが表示される
4. ボタンが「📍 現在地を特定」に戻る
5. 位置情報の許可を「ブロック」に設定した状態で試し、エラーメッセージが赤いバナーで表示され、ボタンがちゃんと元に戻ることも確認する

開発者ツールのNetworkタブでWebSocketフレームを見ると、`"locate_user"`(クリック) → `"start_geolocation"`(依頼) → `"location_found"`(結果) → `"center_map"`(指示)という4つのメッセージが順番に流れているのが見えるはずですにゃ。

---

## つまずきやすい点

- **本番(HTTPS)でないとGeolocation APIが使えない**: `navigator.geolocation`はセキュリティ上の理由で`https://`または`localhost`でしか動作しません。第6章で`check_origin`をきちんと設定し本番がHTTPSで動いていることを確認済みなので今回は問題ありませんが、素のHTTPで自前デプロイしていると「ボタンを押しても何も起きない」という原因になります
- **`handle_event`のパターンマッチはstring keyであること**: `%{"lat" => lat, "lon" => lon}`のようにキーは**文字列**です。第4〜5章のElixir内部の`atom`キーの構造体に慣れていると、`%{lat: lat}`とatomで書いてマッチせず詰まりがちなので注意するにゃ(JSから来るJSONはPhoenixの内部でstring keyのmapとしてデコードされます)
- **連打対策**: `disabled={@locating}`があるので通常のクリックでは連打できませんが、ネットワークの遅延で「クリック→ボタンが無効化されるまでのわずかな間」に2回目のクリックが飛ぶ可能性はゼロではありません。今回は許容していますが、厳密にやるなら`handle_event("locate_user", _, %{assigns: %{locating: true}} = socket)`のガード節で早期リターンする設計に発展できます
- **`userLocationMarker`の掃除忘れ**: `center_map`のたびに前回のマーカーを`removeLayer`しないと、位置情報を取得するたびに青い丸が増え続けてしまいます。第7章の`markersLayer.clearLayers()`と同じ「描き直す前に消す」パターンを思い出すとよいにゃ
- **タイムアウトの扱い**: `getCurrentPosition`の第3引数`timeout`を設定しないと、電波状況の悪い環境で「取得中...」のままボタンが永遠に固まって見えることがあります。今回のように`timeout: 10_000`(10秒)を必ず設定しておくと安心にゃ
- **エラーメッセージの表示忘れ**: `@location_error`を`assign`しただけでは画面に出ません。テンプレート側で`<%= if @location_error do %>`の条件分岐を書くのを忘れないこと（Step 1のテンプレートに含まれています）

---

## 章末チェックリスト

- [ ] `phx-click="locate_user"`のボタンを作り、`handle_event/3`で`assign(locating: true)`できることを確認した
- [ ] `push_event("start_geolocation", ...)`でサーバーからブラウザへ「取得して」の依頼を送れた
- [ ] Hookの`handleEvent("start_geolocation", ...)`で`navigator.geolocation.getCurrentPosition`を呼べた
- [ ] `this.pushEvent("location_found", ...)`でブラウザからサーバーへ座標を送り返せた
- [ ] サーバーが`push_event("center_map", ...)`でブラウザへ`flyTo`の指示を送り返せた
- [ ] `flyTo`と現在地マーカーの表示、前回マーカーの掃除ができている
- [ ] 位置情報を拒否した場合のエラーハンドリング(`location_error`)を実装し、ボタンが正しく元に戻ることを確認した
- [ ] `@location_error`をテンプレートで表示するUIを書いた
- [ ] なぜ「ボタンの状態」をサーバー側の`assign`に一元化するのかを、Vueの`ref`との違いとして説明できる

---

前章: [第7章 — Leaflet JS Hookで地図統合](./chapter_07.md)  
次章: 第9章 — 本番デプロイ最終調整(WebSocketの生命線を本番で確定させる)
