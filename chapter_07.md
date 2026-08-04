# 第7章: Leaflet JS Hookで地図統合（LiveViewとJSの境界線）

第6章で「ロジック層は本番で動く」という自信を得ました。いよいよ元のVueアプリの主役だった**地図**を統合します。

ここが今回のチュートリアルで最もLiveViewらしい章にゃ。Vueでは`onMounted`の中で`L.map(...)`を呼べば地図が生まれ、`ref`が変わるたびにVueのリアクティビティが勝手にDOMを更新してくれました。しかしLiveViewの世界では、**HTMLはサーバーが生成しブラウザに送りつけるもの**という大原則があります。Leafletのように「自分でDOMを直接操作するJSライブラリ」と、この大原則はそのままでは相性が悪いのです。

この章では、**「LiveViewが管理しない領域」を明示的に区切り、そこにJS Hookという橋を架ける**という設計を、小さく刻みながら体験します。いきなり全部（地図＋マーカー＋現在地ボタン）を作らず、まず「マーカー無しの地図が表示される」ことだけを確認してから次に進みますにゃ。

---

## この章でじっくり学ぶこと

- **`phx-update="ignore"`**: LiveViewに「このDOM領域には絶対に手を出すな」と宣言する仕組み
- **JS Hookのライフサイクル**: `mounted` / `updated` / `destroyed`は、Vueのどのライフサイクルフックと似ていて、何が根本的に違うのか
- **`push_event`**: サーバー(Elixirプロセス)からブラウザへ、一方通行でイベントを送る仕組み
- **`this.handleEvent`**: Hook側でサーバーからのイベントを受け取る仕組み
- **なぜマーカーは`assign`ではなく`push_event`で送るのか**: LiveViewの差分レンダリングとLeafletの内部状態が衝突する理由

---

## 移植対象: 元のVueコードの「見えない前提」

```typescript
// Vueでは地図もマーカーも「同じ場所」の変数として存在する
let map: any
let markers: any

onMounted(() => {
  map = L.map(mapContainer.value).setView([35.5, 137.5], 6)
  markers = L.layerGroup().addTo(map)
})

// データが更新されるたびに、markersを直接書き換える
watch(weatherData, (records) => {
  markers.clearLayers()
  records.forEach(r => { /* L.circleMarker(...).addTo(markers) */ })
})
```

Vueでは「地図オブジェクト」も「データ」も同じJSのメモリ空間に同居しているため、`watch`で両者をつなぐのは自然な発想です。

LiveViewではそうはいきません。**地図オブジェクト(`map`, `markers`)はブラウザの中にしか存在せず**、**データ(`records`)はサーバーのプロセスの中にしか存在しません**。この2つの世界をつなぐパイプが、これから作るJS Hookです。

---

## Step 1: アセットパイプラインにLeafletを追加する

Phoenixのフロントエンドビルドは`esbuild`が担当します。Nuxtの`vite.config.ts`に相当するのが`assets/`ディレクトリと`config/config.exs`のesbuild設定です。

```bash
cd assets
npm install leaflet
```

`assets/css/app.css`にLeafletの基本CSSを読み込みます。

```css
@import "leaflet/dist/leaflet.css";
```

**深掘りポイント**: NuxtではCDN経由でLeafletを読み込むケースも多いですが、Phoenixのesbuildはnpmパッケージをバンドルに含める前提の構成です。`node_modules`からの解決はesbuildが自動でやってくれるので、Nuxtで`nuxt.config.ts`にモジュール設定を書く感覚に近いにゃ。

---

## Step 2: マーカー無しの地図だけを表示する（あえて機能を絞る）

いきなりマーカー描画まで作らず、**まず地図タイルが表示されるだけの状態**を目指します。

### HEExテンプレート側

`lib/weather_ex_web/live/weather_live.html.heex`を、第5章のテキストUIと地図を上下に並べる形に書き換えます。

```heex
<div class="p-4 max-w-7xl mx-auto font-sans space-y-4">
  <%--- 第5章までのテキストUI（上部に固定） ---%>
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
  </div>

  <%--- 第7章で追加する地図（下部に配置） ---%>
  <div
    id="weather-map"
    phx-hook="WeatherMap"
    phx-update="ignore"
    class="h-[70vh] w-full rounded-2xl border border-slate-200"
  ></div>
</div>
```

**ここが今回で一番重要な1行です。** `phx-update="ignore"`です。

LiveViewは通常、サーバーから新しいHTMLの差分が来るたびに、対象のDOMを書き換えます。しかしLeafletは`div#weather-map`の中身を**Leaflet自身が直接DOM操作**して管理します（タイル画像の追加、パンやズームでのDOM移動など）。もしLiveViewがこの領域を「自分の管理下」だと思って差分パッチを当てようとすると、**LeafletとLiveViewが同じDOMを取り合って壊し合う**という悲惨な状態になります。

`phx-update="ignore"`は、「このdiv配下は、初回描画された後は二度とLiveViewが手を出さない」という宣言です。VueでいうところのReactの`dangerouslySetInnerHTML`に近い「ここから先はフレームワークの管理外」という境界線を引く行為だと思ってくださいにゃ。

### JS Hook側

`assets/js/hooks/weather_map.js`を新規作成します。

```javascript
import L from "leaflet"

const WeatherMap = {
  mounted() {
    this.map = L.map(this.el).setView([35.5, 137.5], 6)

    L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
      attribution: "&copy; OpenStreetMap contributors",
    }).addTo(this.map)

    this.markersLayer = L.layerGroup().addTo(this.map)
  },

  destroyed() {
    if (this.map) {
      this.map.remove()
    }
  },
}

export default WeatherMap
```

`assets/js/app.js`にHookを登録します。

```javascript
import WeatherMap from "./hooks/weather_map"

let liveSocket = new LiveSocket("/live", Socket, {
  hooks: { WeatherMap },
  params: { _csrf_token: csrfToken },
})
```

### VueのライフサイクルとJS Hookの対応

| Vue | JS Hook | 違い |
|---|---|---|
| `onMounted` | `mounted()` | ほぼ同じ。DOM要素(`this.el`)が使える状態で呼ばれる |
| `onUnmounted` | `destroyed()` | ほぼ同じ。LiveViewがこの要素をDOMから取り除く直前に呼ばれる |
| `watch(props)` | `updated()` | **決定的に違う**。Vueは特定のrefの変化を検知するが、`updated()`は「このHookがついたDOM要素がLiveViewによって更新された時」全般に呼ばれる、もっと粗い粒度のフック |

今回は`phx-update="ignore"`をつけているため、`updated()`はほぼ呼ばれません（LiveViewがこの要素を更新しようとしないため）。マーカーの更新は次のStepで`push_event`経由の別ルートを使います。

---

## Step 3: 動作確認（マーカー無し）

```bash
mix phx.server
```

`http://localhost:4000/weather`を開き、**地図タイルだけが表示される**ことを確認します。まだマーカーは出ません。パン・ズーム操作ができ、5分ごとのカウントダウン更新（第5,6章の機能）が地図の表示を壊さずに動き続けることも確認してくださいにゃ。ここまでで「LiveViewとLeafletが共存できる」という土台ができました。

---

## Step 4: `push_event`でマーカーデータを送る

`lib/weather_ex_web/live/weather_live.ex`の`fetch_weather/1`を拡張し、データ取得のたびにマーカー用のイベントをブラウザへpushします。

```elixir
defp fetch_weather(socket) do
  case Weather.fetch_pressure_diffs() do
    {:ok, records} ->
      socket
      |> assign(records: records, loading: false, error: nil)
      |> push_event("update_markers", %{records: Enum.map(records, &marker_payload/1)})

    {:error, reason} ->
      assign(socket, error: reason, loading: false)
  end
end

defp marker_payload(record) do
  %{
    name: record.name,
    lat: record.lat,
    lon: record.lon,
    diff: record.diff,
    slp: record.slp
  }
end
```

**深掘りポイント**: `assign`はテンプレート(`heex`)を差分レンダリングするためのものですが、`push_event`は**テンプレートを一切経由せず**、指定した名前のイベントをブラウザのJavaScriptへ直接投げつけます。第5章で学んだ「`assign`は新しいsocketを作るだけ」という理屈をそのまま当てはめると、`push_event`はその新しいsocketに「このイベントも一緒に配達してね」という配達伝票を貼り付けるイメージにゃ。

`record.lat`や`record.diff`は第1章で作った`Observation`のレコード（構造体）に入っている値をそのまま使っています。JSON化される際にElixirの構造体はキーがそのままJSのオブジェクトキーになります。

---

## Step 5: Hookでイベントを受け取り、マーカーを描画する

`assets/js/hooks/weather_map.js`に`handleEvent`を追加します。

```javascript
const WeatherMap = {
  mounted() {
    this.map = L.map(this.el).setView([35.5, 137.5], 6)

    L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
      attribution: "&copy; OpenStreetMap contributors",
    }).addTo(this.map)

    this.markersLayer = L.layerGroup().addTo(this.map)

    // サーバーからの "update_markers" イベントを購読する
    this.handleEvent("update_markers", ({ records }) => {
      this.renderMarkers(records)
    })
  },

  renderMarkers(records) {
    this.markersLayer.clearLayers()

    records.forEach((r) => {
      const marker = this.buildMarker(r)
      if (marker) marker.addTo(this.markersLayer)
    })
  },

  buildMarker(r) {
    if (r.diff === null) {
      return L.circleMarker([r.lat, r.lon], {
        radius: 5,
        color: "#cbd5e1", // 安定 = slate-300
        fillOpacity: 0.8,
      }).bindTooltip(`${r.name}: ${r.slp ?? "-"}hPa`)
    }

    if (r.diff <= -1.0) {
      return L.circleMarker([r.lat, r.lon], {
        radius: 7,
        color: "#a855f7", // 急落 = purple-500
        fillOpacity: 0.9,
      }).bindTooltip(`${r.name}: ${r.diff}hPa (急落)`)
    }

    if (r.diff <= -0.5) {
      return L.circleMarker([r.lat, r.lon], {
        radius: 6,
        color: "#ef4444", // 低下 = red-500
        fillOpacity: 0.85,
      }).bindTooltip(`${r.name}: ${r.diff}hPa (低下)`)
    }

    if (r.diff >= 0.5) {
      // 上昇は元のVue版と同じく三角アイコンで区別する
      return L.marker([r.lat, r.lon], {
        icon: L.divIcon({
          className: "rising-triangle-icon",
          html: '<div class="rising-triangle"></div>',
          iconSize: [12, 12],
        }),
      }).bindTooltip(`${r.name}: +${r.diff}hPa (上昇)`)
    }

    return L.circleMarker([r.lat, r.lon], {
      radius: 5,
      color: "#cbd5e1", // 安定
      fillOpacity: 0.8,
    }).bindTooltip(`${r.name}: ${r.diff}hPa`)
  },

  destroyed() {
    if (this.map) {
      this.map.remove()
    }
  },
}

export default WeatherMap
```

`assets/css/app.css`に上昇マーカー用の三角形CSSを追加します。

```css
/* Leaflet divIcon のデフォルト背景・枠線を消す */
.rising-triangle-icon {
  background: transparent !important;
  border: none !important;
}

.rising-triangle {
  width: 0;
  height: 0;
  border-left: 6px solid transparent;
  border-right: 6px solid transparent;
  border-bottom: 12px solid #3b82f6; /* blue-500 */
}
```

> **補足**: 元のVue版では `bindPopup` でHTMLテンプレート（天気絵文字・気温・標高など）を表示していましたが、本章ではLiveView→JSのデータ流れに集中するため、一旦 `bindTooltip` で簡易表示にしています。リッチなポップアップは、必要に応じて `marker_payload` にフィールドを追加し、Hook側でHTML文字列を組み立てて `bindPopup` に渡す形で実装できます。

### なぜ`this.handleEvent`は関数の中に「素直に」書けるのか

Vueで似たようなことをやろうとすると、WebSocketの生イベントを`onMounted`内で購読し、`onUnmounted`で確実に解除する、という定型処理が必要になります。忘れると、コンポーネントが破棄された後もイベントリスナーが残り続けるメモリリークの原因になります。

`this.handleEvent`は、Hookが`destroyed()`される際に**LiveView側が自動的に購読解除まで面倒を見てくれます**。第5章で見た「プロセスのライフサイクルが明示的」という思想がここにも表れていて、Hookという「JSの世界の小さなプロセスもどき」のライフサイクルに、購読の生死がきちんと紐づいているのですにゃ。

---

## Step 6: 動作確認（マーカーあり）

```bash
mix phx.server
```

`http://localhost:4000/weather`を開き、以下を確認します。

1. カウントダウンが0になった瞬間（または`iex`から`send(view.pid, :tick)`を送った瞬間）、地図上にマーカーが現れる
2. 気圧が下がっている地点は赤〜紫の丸、上がっている地点は青い三角で表示される
3. マーカーをクリック(タップ)すると地点名と数値がツールチップで見える
4. 5分ごとの再取得のたびに、マーカーが**一度全部消えて描き直される**ことも確認する(`clearLayers()`の挙動)

開発者ツールのNetworkタブでWebSocketフレームを見ると、`"update_markers"`というイベント名と、地点データのJSON配列が流れているのが確認できるはずです。これはStep 4で見た「差分レンダリングを経由しない、直接のメッセージ配達」の証拠にゃ。

---

## つまずきやすい点

- **`phx-update="ignore"`を忘れる**: これを忘れると、5分ごとのデータ更新のたびにLiveViewが地図のDOMを丸ごと差し替えようとし、Leafletの内部状態(タイルキャッシュやズームレベル)と衝突して地図が壊れたり、コンソールにLeafletのエラーが出たりします。地図系のJS Hookでは**ほぼ必須のおまじない**だと思ってよいにゃ
- **`push_event`のタイミング**: `mount/3`の初回(`connected?`が`false`)の時点で`push_event`を呼んでも、まだJS側のWebSocketが繋がっていないため届きません。必ず`connected?(socket)`が`true`になった後(今回は`handle_info(:tick, ...)`経由のデータ取得時)に呼ぶこと
- **Hookの`mounted()`と初回データの競合**: 地図がまだ`mounted()`されていないタイミングで最初の`push_event`が飛んでくる可能性があります。今回の設計では「地図の初回表示」と「初回のデータ取得(5分後)」が時間的にズレているため実害は出にくいですが、即座に初期データを見せたい場合は、Hook側の`mounted()`で`this.pushEvent("request_initial_data", {})`のようにHook→サーバー方向のイベントも使い、明示的に「準備できたよ」を伝える設計に発展させることができます(この往復パターンは第8章のGeolocationで本格的に扱います)
- **`destroyed()`での後片付け漏れ**: `this.map.remove()`を忘れると、LiveViewのナビゲーションでこのページから離れた際に地図インスタンスがメモリに残り続けます。SPA的な画面遷移を伴わない今回の構成では影響は小さいですが、癖として必ず書いておくにゃ
- **`diff`が`nil`のケースの扱い忘れ**: 観測データが欠測の地点は`diff`が`nil`になります(第1章の`Observation`参照)。JS側で先頭に`null`チェックを置かずに`r.diff <= -1.0`のような数値比較から書き始めると、意図しない分岐に落ちたり後続の比較でハマったりしやすいので、**`null`チェックを最初に置く**癖をつけるとよいにゃ(今回は先頭でチェック済み)
- **Leafletの`divIcon`にデフォルト枠線が残る**: `className`を指定しても、Leafletは`.leaflet-div-icon`クラスで白背景＆枠線を自動適用することがあります。`background: transparent !important; border: none !important;`を忘れると、青い三角の周りに不要な白枠が出るので要注意にゃ

---

## 章末チェックリスト

- [ ] `assets/js/hooks/weather_map.js`を作成し、`app.js`に登録した
- [ ] `phx-update="ignore"`の意味(LiveViewの管理外にする宣言)を理解した
- [ ] マーカー無しの地図がまず表示されることを確認してから、マーカー描画に進んだ
- [ ] `weather_live.ex`の`fetch_weather/1`で`push_event("update_markers", ...)`を送るようにした
- [ ] Hookの`handleEvent`でイベントを受け取り、`L.circleMarker`/`L.divIcon`でマーカーを描画した
- [ ] 気圧低下(赤・紫)と上昇(青い三角)が元のVue版の凡例通りに描き分けられている
- [ ] `rising-triangle-icon`に`background: transparent`を適用し、不要な枠線を消した
- [ ] `destroyed()`で`this.map.remove()`を呼び、後片付けを書いた
- [ ] ブラウザで実際にマーカーが表示され、5分ごとに更新されることを確認した

---

前章: [第6章 — 中間デプロイチェックポイント①](./chapter_06.md)  
次章: 第8章 — Geolocation往復(`pushEvent`と`push_event`の双方向通信)
