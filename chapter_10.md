# 第10章(任意): 鯨アニメーション移植 — サーバーに送るべきでない状態を見極める

第9章までで、気圧マップアプリの主要機能は本番環境で一通り動くようになりました。この章は**任意**です。元のVueアプリには、5分おきに太平洋上をクジラのスプライトアニメーションが泳いでいくという、機能とは無関係な遊び心のある演出がありました。これを移植します。

なぜわざわざ最後にこれをやるのか。理由は技術的な難しさではなく、**このチュートリアルを通じて一貫して問うてきた「その状態は誰が持つべきか」という問いに、はっきりした反例を与えてくれるから**です。第5〜8章では「状態はサーバー(LiveViewプロセス)に置き、ブラウザは描画に徹する」という設計を繰り返し学んできました。鯨のアニメーションは、**あえてその原則から外れる**ことが正解になる、数少ないケースです。

---

## この章でじっくり学ぶこと

- **どんな状態でもLiveViewに送るべきではない、という判断基準**
- 既存のJS Hook(第7章の`WeatherMap`)を**拡張**する形での機能追加(新しいHookを作らない選択)
- `phx-update="ignore"`の適用範囲が**子要素にも及ぶ**という性質の応用
- Konva.jsによるスプライトシートアニメーションの基本
- Leafletの`latLngToContainerPoint`で「地図上の緯度経度」を「画面上のピクセル座標」に変換する仕組み

---

## 移植対象: 元のVueコードの構造

```typescript
// 太平洋上の片道区間（日本列島から離れた海域）
const WHALE_START: [number, number] = [31.0, 130.0]
const WHALE_END: [number, number] = [34.0, 144.0]
const WHALE_SWIM_MS = 10000             // 泳いでいる時間（片道）
const WHALE_FADE_MS = 2500              // フェードイン・アウトの時間
const WHALE_INTERVAL_MS = 5 * 60 * 1000 // 出現間隔（5分）

function startWhaleSwim() {
  if (map) {
    const startPoint = map.latLngToContainerPoint(WHALE_START)
    const endPoint = map.latLngToContainerPoint(WHALE_END)
    const dx = endPoint.x - startPoint.x
    const dy = endPoint.y - startPoint.y
    whaleRotation.value = Math.atan2(dy, dx) * (180 / Math.PI)
  }
  whaleSprite?.start()
  const startTime = performance.now()
  whaleAnimId = requestAnimationFrame(() => animateWhaleSwim(startTime))
}

function animateWhaleSwim(startTime: number) {
  const elapsed = performance.now() - startTime
  if (elapsed >= WHALE_SWIM_MS) {
    whaleOpacity.value = 0
    whaleSprite?.stop()
    scheduleNextWhale()
    return
  }
  const t = elapsed / WHALE_SWIM_MS
  const lat = lerp(WHALE_START[0], WHALE_END[0], t)
  const lon = lerp(WHALE_START[1], WHALE_END[1], t)
  if (map) {
    const point = map.latLngToContainerPoint([lat, lon])
    whaleX.value = point.x
    whaleY.value = point.y
  }
  // フェードイン・フェードアウト計算(省略、後述のJSで完全版を示す)
  whaleAnimId = requestAnimationFrame(() => animateWhaleSwim(startTime))
}
```

ポイントは2つあります。

1. **`map`という、Leafletの地図インスタンスに直接アクセスしている**。第7章で作った`WeatherMap`Hookがすでに`this.map`としてこのインスタンスを保持しています
2. **アニメーションの状態(`whaleX`, `whaleY`, `whaleOpacity`, タイマーID)は、すべてブラウザのメモリ上だけに存在**。サーバーには一切送られていません

---

## Step 0: この機能は本当にサーバーの状態にすべきか、を考える

移植を始める前に、あえて立ち止まって設計判断をします。「LiveViewのassignに`whale_position`を持たせて、`push_event`で位置を送る」という、第7章と同じパターンも技術的には可能です。しかし、それは**やるべきではありません**。理由を整理します。

| 観点 | マーカーデータ(第7章) | 鯨の位置(この章) |
|---|---|---|
| データの出どころ | JMAの実測データ(サーバー側でしか取得できない) | 固定の2地点間を補間するだけ(定数だけで完結) |
| 更新頻度 | 5分ごと(サーバー主導) | 60fps相当(`requestAnimationFrame`、ブラウザ主導) |
| サーバーが知る必要性 | ある(表示以外の用途にも使える可能性がある実データ) | 無い(見た目の演出以外に一切意味を持たない) |
| WebSocketで60fps分の座標を送ったら | (該当なし) | 毎秒何十回もサーバー↔ブラウザ間の通信が発生し、無意味な帯域とプロセス負荷を消費する |

**「本物のデータで、複数箇所から参照されうるもの」はサーバーに置く。「見た目だけの演出で、ブラウザ内で完結するもの」はブラウザに置く。** これが判断基準です。第5〜8章で「LiveViewは状態をサーバーに持つ」と繰り返し学んできましたが、それは「**全部**の状態をサーバーに持つべきだ」という意味ではありません。今回の鯨は、後者の典型例として意図的に選びました。

---

## Step 1: アセットを配置する

スプライトシート画像とJSONを、Phoenixの静的ファイル置き場にコピーします。

```bash
cp path/to/whale_spritesheet.png priv/static/images/whale_spritesheet.png
cp path/to/whale_spritesheet.json priv/static/images/whale_spritesheet.json
```

> **補足**: 元のNuxtリポジトリには`public/whale.mp4`という動画ファイルもありますが、`pages/index.vue`のコードを確認したところ**どこからも参照されていません**でした(使われなくなった過去の実装の名残と思われます)。移植対象はスプライトシート版(`whale_spritesheet.png`/`.json`)だけで十分です。不要なファイルまで律儀に移植する必要はありません。

`whale_spritesheet.json`の中身を確認しておきましょう。

```json
{
  "images": ["whale_spritesheet.png"],
  "frames": [[0, 0, 300, 169], [300, 0, 300, 169], ...],
  "animations": { "swim": { "frames": [0, 1, 2, ..., 119], "speed": 1 } },
  "framerate": 12
}
```

`frames`は`[x, y, width, height]`の配列(スプライトシート画像内の各コマの切り出し座標)、`animations.swim.frames`はどのコマ番号を順番に再生するかのリスト(120コマ)、`framerate`は再生速度(12fps)です。この構造はKonva.jsの`Sprite`がそのまま使える形ではなく、後述するように**フラットな数値配列へ変換する処理**が必要です(これは元のVueコードにも存在した変換で、Elixir側の仕事ではなくJS側の仕事です)。

Phoenixの`priv/static/images/`に置いたファイルは、`lib/weather_ex_web/endpoint.ex`の`Plug.Static`設定で自動的に`/images/whale_spritesheet.png`のようなURLで配信されます(`mix phx.new`のデフォルト設定のままで問題ありません)。

---

## Step 2: Konvaをインストールする

```bash
cd assets
npm install konva
```

---

## Step 3: HEExテンプレートに鯨用のdivを追加する

第7章で作った`weather_live.html.heex`の`#weather-map`の**内側**に、鯨アニメーション用のdivを追加します。

```heex
<div
  id="weather-map"
  phx-hook="WeatherMap"
  phx-update="ignore"
  class="h-[70vh] w-full rounded-2xl border border-slate-200 relative overflow-hidden"
>
  <div id="whale-canvas" class="whale-canvas"></div>
</div>
```

**ここが今回の設計の要です。** `#whale-canvas`を`#weather-map`の**子要素**として配置しました。第7章で学んだ通り、`phx-update="ignore"`は「この要素配下は二度とLiveViewが手を出さない」という宣言です。**その効果は子孫要素にも及びます**。つまり`#whale-canvas`もLiveViewの差分レンダリングの対象外になり、新しく`phx-hook`を追加しなくても、既存の`WeatherMap`Hookの`mounted()`の中から`this.el.querySelector("#whale-canvas")`で素直に参照できます。

新しいHookを作らなかったのには理由があります。鯨のアニメーションはLeafletの`this.map`(地図インスタンス)に依存しており、それは`WeatherMap`Hookだけが持っている値です。別のHookに分けると「地図インスタンスをどうやってHook間で共有するか」という余計な複雑さが生まれます。**同じDOM領域・同じデータに依存する処理は、無理に分割せず1つのHookにまとめる**というのも実用的な設計判断です。

---

## Step 4: JS Hookに鯨アニメーションを追加する

第7章の`assets/js/hooks/weather_map.js`を拡張します。追加分だけ示します(既存の`mounted`/`renderMarkers`/`buildMarker`/`destroyed`はそのまま残します)。

```javascript
import L from "leaflet"
import Konva from "konva"

// 太平洋上の片道区間(日本列島から離れた海域)
const WHALE_START = [31.0, 130.0]
const WHALE_END = [34.0, 144.0]
const WHALE_SWIM_MS = 10000 // 泳いでいる時間(片道)
const WHALE_FADE_MS = 2500 // フェードイン・アウトの時間
const WHALE_INTERVAL_MS = 5 * 60 * 1000 // 出現間隔(5分)

function lerp(a, b, t) {
  return a + (b - a) * t
}

const WeatherMap = {
  mounted() {
    this.map = L.map(this.el).setView([35.5, 137.5], 6)

    L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
      attribution: "&copy; OpenStreetMap contributors",
    }).addTo(this.map)

    this.markersLayer = L.layerGroup().addTo(this.map)

    this.handleEvent("update_markers", ({ records }) => {
      this.renderMarkers(records)
    })

    this.isDestroyed = false
    this.initWhale()
  },

  // --- ここから鯨アニメーション ---

  async initWhale() {
    const whaleEl = this.el.querySelector("#whale-canvas")
    if (!whaleEl) return

    const stage = new Konva.Stage({
      container: whaleEl,
      width: 300,
      height: 169,
    })
    const layer = new Konva.Layer()
    stage.add(layer)

    const res = await fetch("/images/whale_spritesheet.json")
    const spriteSheetData = await res.json()

    // spriteSheetData.frames は [[x, y, width, height], ...] の形式を前提としている。
    // 各フレームインデックスを flatMap で [x, y, w, h, x, y, w, h, ...] に展開する
    // (Konva.Sprite の animations オプションが要求するフラットな数値配列にするため)
    const flatFrames = spriteSheetData.animations.swim.frames.flatMap(
      (frameIndex) => spriteSheetData.frames[frameIndex]
    )

    const image = new Image()
    image.src = "/images/whale_spritesheet.png"
    image.onload = () => {
      // fetch/画像読み込みが完了する前にページを離れ、destroyed()が
      // 先に呼ばれているケースがある。this.map は既に remove() 済み、
      // whaleEl も DOM から外れている可能性があるため、ここで打ち切る。
      if (this.isDestroyed) return

      this.whaleSprite = new Konva.Sprite({
        x: 0,
        y: 0,
        image: image,
        animation: "swim",
        animations: { swim: flatFrames },
        frameRate: spriteSheetData.framerate,
      })
      layer.add(this.whaleSprite)
      layer.draw()

      this.startWhaleSwim() // 読み込み完了後にすぐ1回泳がせる。以降5分おき
    }
  },

  startWhaleSwim() {
    if (this.isDestroyed) return

    const whaleEl = this.el.querySelector("#whale-canvas")
    const startPoint = this.map.latLngToContainerPoint(WHALE_START)
    const endPoint = this.map.latLngToContainerPoint(WHALE_END)
    const dx = endPoint.x - startPoint.x
    const dy = endPoint.y - startPoint.y
    const rotation = Math.atan2(dy, dx) * (180 / Math.PI)

    this.whaleSprite?.start()
    const startTime = performance.now()
    this.whaleAnimId = requestAnimationFrame(() =>
      this.animateWhaleSwim(startTime, rotation, whaleEl)
    )
  },

  animateWhaleSwim(startTime, rotation, whaleEl) {
    if (this.isDestroyed) return

    const elapsed = performance.now() - startTime

    if (elapsed >= WHALE_SWIM_MS) {
      whaleEl.style.opacity = 0
      this.whaleSprite?.stop()
      this.whaleTimeoutId = setTimeout(
        () => this.startWhaleSwim(),
        WHALE_INTERVAL_MS
      )
      return
    }

    const t = elapsed / WHALE_SWIM_MS
    const lat = lerp(WHALE_START[0], WHALE_END[0], t)
    const lon = lerp(WHALE_START[1], WHALE_END[1], t)
    const point = this.map.latLngToContainerPoint([lat, lon])

    let opacity
    if (elapsed < WHALE_FADE_MS) {
      opacity = elapsed / WHALE_FADE_MS
    } else if (elapsed > WHALE_SWIM_MS - WHALE_FADE_MS) {
      opacity = (WHALE_SWIM_MS - elapsed) / WHALE_FADE_MS
    } else {
      opacity = 1
    }

    whaleEl.style.transform = `translate3d(${point.x}px, ${point.y}px, 0) rotate(${rotation}deg)`
    whaleEl.style.opacity = opacity

    this.whaleAnimId = requestAnimationFrame(() =>
      this.animateWhaleSwim(startTime, rotation, whaleEl)
    )
  },

  // --- ここまで鯨アニメーション ---

  renderMarkers(records) {
    /* 第7章のまま */
  },

  buildMarker(r) {
    /* 第7章のまま */
  },

  destroyed() {
    this.isDestroyed = true
    if (this.whaleAnimId) cancelAnimationFrame(this.whaleAnimId)
    if (this.whaleTimeoutId) clearTimeout(this.whaleTimeoutId)
    if (this.map) {
      this.map.remove()
    }
  },
}

export default WeatherMap
```

### なぜ`isDestroyed`フラグが必要なのか

`initWhale()`は`async`関数ですが、`mounted()`内で`await`していません(意図的です — 画像の読み込みを待ってからLiveViewの他の初期化処理をブロックしたくないため)。この非同期性が、次の競合状態を生みます。

1. `mounted()`が呼ばれ、`initWhale()`が開始する(`fetch`と画像読み込みが進行中)
2. `fetch`や画像読み込みが完了する**前**に、ユーザーが別ページへ遷移する
3. `destroyed()`が呼ばれ、`this.map.remove()`が実行される
4. その**後**になって、`image.onload`が発火する
5. `onload`内の処理が、すでに破棄済みの`this.map`(`latLngToContainerPoint`を呼ぶと例外)や、DOMツリーから外れた`whaleEl`(`style`を書き換えても何も起きないが、無駄な処理が走り続ける)にアクセスしようとする

`this.isDestroyed`フラグを`mounted()`で`false`初期化し、`destroyed()`で`true`に立てた上で、`onload`・`startWhaleSwim`・`animateWhaleSwim`の入り口でそれぞれ確認することで、**「もう破棄された後なら何もしない」**を徹底しています。`requestAnimationFrame`のループ内でも毎回チェックしているのは、ループの1周目は生きていても、次のフレームが来るまでの間に`destroyed()`が呼ばれる可能性があるためです。

元のVueコードは`whaleX`/`whaleY`/`whaleOpacity`という`ref`を更新し、`computed`の`whaleStyle`が自動的にCSSの`transform`/`opacity`に変換していました。JS Hookにはリアクティブシステムが無いため、`animateWhaleSwim`の中で`whaleEl.style.transform`/`whaleEl.style.opacity`を**直接書き換えています**。

これは「劣化した書き方」ではありません。むしろ、Vueの`ref`が実現していた「値の変更を検知してDOMに反映する」という間接層が、`requestAnimationFrame`のような**高頻度更新が要求される場面ではオーバーヘッドになりうる**ことを踏まえると、DOM操作を直接行うこの書き方の方が自然です。LiveViewもHookのこの領域には一切干渉しないので、リアクティブシステムを介さない生のDOM操作を思う存分書ける、というのが`phx-update="ignore"`配下の特権です。

---

## Step 5: CSSを追加する

```css
.whale-canvas {
  position: absolute;
  left: 0;
  top: 0;
  width: 300px;
  height: 169px;
  pointer-events: none;
  z-index: 400;
  opacity: 0;
  transform-origin: center;
}
```

元のVue版は`position: fixed`(画面全体基準)でした。これはVue側で`#whale-canvas`が`<ClientOnly>`直下、つまり`#map`と**兄弟要素**として`body`に近い階層に置かれていたためです(第7章冒頭の元コードを見ると、`#map`と`.whale-canvas`が同じ階層に並んでいます)。今回は`#whale-canvas`を`#weather-map`の**子要素**としてネストする設計に変更したため(Step 3参照)、`position: absolute`にして**親要素(`#weather-map`、`relative`指定済み)基準の配置**に変更しています。地図の外に鯨がはみ出さず、地図の表示領域内だけで泳ぐようになります。座標計算自体は`map.latLngToContainerPoint`が「地図コンテナ内でのピクセル位置」を返すため、この変更と自然に整合します。

---

## Step 6: 動作確認

```bash
mix phx.server
```

`http://localhost:4000/weather`を開きます。

1. ページを開いた直後(スプライト画像の読み込み完了後)、鯨が太平洋上を斜めに泳いでいくのが見える
2. 泳いでいる間、フェードイン→フルオパシティ→フェードアウトの変化が滑らかに起きる
3. 泳ぎ終わったら消え、5分後に再び現れる(待つのが大変な場合、後述の「つまずきやすい点」でタイマー値を一時的に短縮する方法に触れています)
4. 地図をズーム・パンしても、鯨のアニメーション自体は壊れない(次の周回で新しい`latLngToContainerPoint`を計算し直すため)
5. 第7章のマーカー更新(5分ごとのデータ取得)と、この鯨のアニメーションが**互いに干渉せず同時に動く**ことを確認する

---

## つまずきやすい点

- **画像読み込み中のページ離脱による競合状態**: `initWhale()`は非同期処理であり、`fetch`や画像読み込みが完了する前に`destroyed()`が呼ばれる可能性があります。`isDestroyed`フラグでガードしていない場合、破棄済みの`this.map`にアクセスして例外が発生することがあります(詳細は上記の解説を参照)。素早いページ遷移を繰り返すテストで再現しやすいので、実装したら意図的に素早く行き来して確認するとよいにゃ
- **5分待つのが辛い場合**: `WHALE_INTERVAL_MS`を一時的に`5000`(5秒)などに変更すれば動作確認が速くなります。確認が終わったら**必ず元の`5 * 60 * 1000`に戻す**のを忘れないこと(戻し忘れて本番にデプロイすると、5秒おきに鯨が泳ぐ落ち着かないサイトになります)
- **`#whale-canvas`が見つからない**: `initWhale()`内の`this.el.querySelector("#whale-canvas")`が`null`を返す場合、HEExテンプレート側で`#whale-canvas`が`#weather-map`の**外**に置かれてしまっている可能性があります。`phx-update="ignore"`の適用範囲(子要素まで)を思い出し、必ず`#weather-map`の内側にネストしてください
- **画像のパスミス**: `priv/static/images/`に置いたファイルは`/images/...`というURLで配信されます。`priv/static/`直下に置いた場合は`/whale_spritesheet.png`になるなど、配置場所とURLの対応がNuxtの`public/`(ルート直下がそのままURLになる)と異なるので注意してください
- **`destroyed()`でのクリーンアップ漏れ**: `whaleAnimId`/`whaleTimeoutId`の解除を忘れると、LiveViewのナビゲーションでこのページを離れた後もタイマーが動き続け、存在しないDOM要素(`whaleEl`)にアクセスしようとしてJSエラーになることがあります
- **Konvaのバンドルサイズ**: Konvaは決して小さくないライブラリです。この鯨アニメーションのためだけに導入するのが気になる場合、Konvaを使わず素のCanvas API(`drawImage`でスプライトシートの矩形を切り出して描画)で同等のことも実現できます。今回は元のVue実装に忠実に移植する方針のためKonvaを採用しましたが、「本当にこのライブラリが必要か」を都度考える習慣は持っておくとよいでしょう

---

## 章末チェックリスト

- [ ] `priv/static/images/`にスプライトシート(`.png`/`.json`)を配置した
- [ ] `assets`ディレクトリで`npm install konva`を実行した
- [ ] `#whale-canvas`を`#weather-map`の**内側**に配置し、`phx-update="ignore"`の適用範囲を利用した
- [ ] 既存の`WeatherMap`Hookを拡張する形で実装し、新しいHookを作らなかった理由を説明できる
- [ ] 「なぜこの状態をLiveViewのassignに送らないのか」を自分の言葉で説明できる
- [ ] 鯨が地図上を泳ぎ、フェードイン・アウトすることを確認した
- [ ] `destroyed()`でタイマーとアニメーションフレームを確実に解除した
- [ ] `isDestroyed`フラグで、非同期処理完了時の競合状態(破棄後のコールバック発火)をガードした
- [ ] `WHALE_INTERVAL_MS`を確認用に変更した場合、元の値に戻した

---

## このチュートリアルを振り返って

第0章から第10章まで、Nuxt/Vueの気圧マップアプリをElixir/Phoenixに移植しながら進めてきました。振り返ると、各章で扱ったテーマは技術要素の羅列ではなく、一貫して**「この処理・この状態は、どこに置くのが自然か」という問い**でした。

- 純粋な計算は関数として、副作用は`with`とタグ付きタプルで明示的に(第1〜2章)
- 「一度取得したら保持し続けたい」状態はGenServerのプロセスに(第3章)
- 内部実装は呼び出し側から隠すContextという境界を引く(第4章)
- ブラウザに状態を持たせず、サーバーのプロセスに状態を持たせるのがLiveViewの基本形(第5〜6章)
- しかしDOM操作が本質のライブラリ(Leaflet)は、あえてLiveViewの管理外に置く(第7〜8章)
- 本番環境という「見えない前提」を早い段階で洗い出しておく(第0, 6, 9章)
- そして最後に、**その原則をわきまえた上で、あえて外れるべき場面を見極める**(第10章)

これらはすべて、Nuxt/Vueで無意識にできていたことを、Elixir/Phoenixでは**意識的に設計判断として行う**という体験だったはずです。お疲れ様でした。

---

前章: [第9章 — 本番デプロイ最終調整](./chapter_09.md)
