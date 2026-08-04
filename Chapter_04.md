# 第4章: Context層で統合（Elixir入門編）

第1章で「データ変換の純粋関数」、第2章で「外部API通信」、第3章で「状態を持つキャッシュプロセス」を作りました。パズルのピースは揃いました。

この章では、それらを組み合わせて**「全観測地点の気圧差分データを取得する」という1つの公開API**にまとめます。Phoenixフレームワークでは、この「機能の窓口」となるモジュールを**Context**と呼びます。

そしてここで、TypeScriptの**命令型なループ処理（`for` と `continue`）**が、Elixirの**宣言型なデータ変換（`for` 内包表記とパターンマッチ）**によってどうのように美しく置き換わるのか、その醍醐味をじっくり味わいましょう。

## この章でじっくり学ぶこと

- **Context**という設計単位（「呼び出し側は実装詳細を知らなくていい」という境界）
- `with` の中で**ガード付きパターン**（`x when not is_nil(x) <- ...`）を使う理由
- リストの**分解パターンマッチ**（`[a, b | _]`）で配列の形状をチェックする
- **`for` 内包表記**でJSの `for` ループ＋ `continue` を置き換える魔法
- 「1要素リストを使ったジェネレータ」というElixirの定番テクニック
- 依存性注入は**GenServerだけでなく、普通の関数**にも同じように適用できる

---

## 移植対象: 元のTypeScriptコードの中核処理

今回移植するのは、APIエンドポイントの本体部分です。

```typescript
export default defineEventHandler(async (event) => {
  try {
    const stations = await getStationsMaster()
    const { nowData, pastData, nowTime } = await fetchObservationPair()
    if (!nowData || !pastData) return []

    const processed = []
    for (const [sid, data] of Object.entries(nowData)) {
      // --- ここから「1地点の判定・変換」 ---
      if (!(sid in pastData)) continue
      const pNowInfo = data.pressure
      const pPastInfo = pastData[sid].pressure
      if (!pNowInfo || !pPastInfo || pNowInfo[0] === null || pPastInfo[0] === null) continue

      const m = stations[sid]
      if (!m) continue
      const latRaw = m.lat
      if (!Array.isArray(latRaw)) continue
      // --- ここまで「continue（スキップ）の連続」 ---

      // ... データの組み立て ...
      processed.push({ name: m.kjName, ... })
    }
    return processed
  } catch (e) {
    return []
  }
})
```

このコードは大きく2層でできています。

1. **1観測地点分のデータを判定・変換する処理**（`continue` の連続 → 1レコード組み立て）
2. **全地点をループしてレコード配列を作る処理**

TypeScriptではこれらを1つの関数の中に混ぜ書きしますが、Elixirに移植する際は、この2層を**2つのモジュールの役割分担**として明確に切り離します。

| TSの役割 | Elixirでの置き場所 |
|---|---|
| 1地点分の判定・変換(`continue`の連続) | `Observation.build_record/4` (純粋関数、第1章のモジュールに追加) |
| 全地点をループしてまとめる | `Weather.fetch_pressure_diffs/1` (Context、通信・キャッシュを呼ぶ) |

---

## 実装1: `Observation.build_record/4` (1地点分のロジック)

`lib/weather_ex/weather/observation.ex` に、以下のコードを追加してください。

```elixir
@doc """
1観測地点分のデータから、表示用レコードを組み立てる。

必要なデータ(気圧・緯度経度)が欠けている場合は `:skip` を返す。
元のTypeScript実装の複数の `continue` 文に相当する。
"""
def build_record(station, now_reading, past_reading, now_time) do
  with pressure_now when not is_nil(pressure_now) <-
         first_value(Map.get(now_reading, "pressure")),
       pressure_past when not is_nil(pressure_past) <-
         first_value(Map.get(past_reading, "pressure")),
       [lat_deg, lat_min | _] <- Map.get(station, "lat"),
       [lon_deg, lon_min | _] <- Map.get(station, "lon") do
    
    elev = Map.get(station, "elevation") || Map.get(station, "altitude") || 0
    temp_now = first_value(Map.get(now_reading, "temp"))
    temp_past = first_value(Map.get(past_reading, "temp")) # 過去の気温を正しく取得

    # 現在の気圧・気温で現在の海面更正気圧を計算
    slp_now = sea_level_pressure(pressure_now * 1.0, elev * 1.0, temp_as_float(temp_now))
    # 過去の気圧・過去の気温で過去の海面更正気圧を計算
    slp_past = sea_level_pressure(pressure_past * 1.0, elev * 1.0, temp_as_float(temp_past))

    weather_code = first_value(Map.get(now_reading, "weather"))

    {:ok,
     %{
       name: Map.get(station, "kjName"),
       enname: Map.get(station, "enName"),
       lat: convert_coord({lat_deg, lat_min}),
       lon: convert_coord({lon_deg, lon_min}),
       slp: slp_now,
       diff: pressure_diff(slp_now, slp_past),
       pressure: round1(pressure_now),
       elev: elev,
       temp: temp_now && round1(temp_now),
       weather: weather_emoji(weather_code),
       time: now_time
     }}
  else
    # どれか一つでも条件を満たさなければ（マッチしなければ）:skipを返す
    _ -> :skip
  end
end

# --- プライベートヘルパー関数 ---

defp first_value(nil), do: nil
defp first_value([value | _]), do: value

defp temp_as_float(nil), do: nil
defp temp_as_float(temp), do: temp * 1.0

defp round1(nil), do: nil
defp round1(value), do: Float.round(value * 1.0, 1)

defp pressure_diff(nil, _slp_past), do: nil
defp pressure_diff(_slp_now, nil), do: nil
defp pressure_diff(slp_now, slp_past), do: Float.round(slp_now - slp_past, 1)
```

> **移植時の注意点**: 元のTSコードには、過去の海面更正気圧(`slpPast`)の計算に、現在の気温(`temp`)を使ってしまっているバグがありました。Elixirへの移植にあたり、正しく過去データ側の気温(`temp_past`)を使うように修正しています。移植は「仕様を正しく実装し直す」作業でもあります。

---

## Elixir的な着眼点(1): `with` のガード付きパターンで `if continue` を消す

TypeScriptでは「データが `null` なら `continue` で次のループに飛ぶ」という否定条件の連続でした。
Elixirの `with` では、**「正しい形のデータが取れるまで処理を続け、そうでなければ `else` に落ちる」**という肯定条件で書きます。

```elixir
with pressure_now when not is_nil(pressure_now) <- first_value(...),
```

第2章では `{:ok, val} <- ...` というタプルのマッチだけでしたが、ここでは `when` という**ガード節**を使っています。
「左辺の `first_value(...)` の結果を `pressure_now` に代入する。ただし、それが `nil` でなければマッチ成功とする」という意味になります。

もし `nil` が来たらマッチに失敗し、`with` の後ろにある `else _ -> :skip` に直行します。これにより、`if` 文をネストすることなく「失敗したらスキップする」というロジックを美しく表現しています。

### ガード(`when`)で使える式の制限
ガードの中には、複雑な関数呼び出しは書けません（例えば `when String.length(x) > 0` は不可）。書けるのは `is_nil/1`, `is_map/1`, `>`, `<`, `==` などの**「ガード安全な式」**だけです。これはコンパイラの最適化とパターンマッチの速さを保つためのElixirの設計です。

## Elixir的な着眼点(2): リスト分解パターンマッチで「配列であること」を検証する

```elixir
[lat_deg, lat_min | _] <- Map.get(station, "lat")
```

気象庁APIの緯度経度は `[35, 41]` のようなリスト（配列）で送られてきます。
このパターンマッチは、「**リストであり、かつ先頭に2つ以上の要素が存在する**」という条件を一発でチェックしています。

- `Map.get(station, "lat")` が `nil` なら → リストではないのでマッチ失敗（`else` へ）
- `Map.get(station, "lat")` が `"不正な文字列"` なら → マッチ失敗（`else` へ）
- `[35]` のように要素が1つしかなければ → 2つ目を取り出せないのでマッチ失敗（`else` へ）
- `[35, 41, ...]` なら → `lat_deg = 35`, `lat_min = 41` となりマッチ成功

TypeScriptの `if (!Array.isArray(latRaw)) continue` や長さチェックを、1行のパターンマッチに圧縮できています。

---

## 実装2: `Weather` モジュール（Context、全地点のループ）

次に、APIからデータを取り、全地点をループするContextモジュールを作ります。
`lib/weather_ex/weather.ex` を作成してください。

```elixir
defmodule WeatherEx.Weather do
  @moduledoc """
  気象データ機能の公開API(Context)。

  呼び出し側(LiveViewなど)はこのモジュールの関数だけを呼び、
  内部で`StationCache`や`JmaClient`をどう使っているかを意識しない。
  """

  alias WeatherEx.Weather.{JmaClient, Observation, StationCache}

  @doc """
  現在の気圧・気温・天気と、1時間前との気圧差分を全観測地点分取得する。
  """
  def fetch_pressure_diffs(opts \\ []) do
    stations_fun = Keyword.get(opts, :stations_fun, &StationCache.get_stations/0)
    observation_fun = Keyword.get(opts, :observation_fun, &JmaClient.fetch_observation_pair/0)

    with {:ok, stations} <- stations_fun.(),
         {:ok, %{now: now_data, past: past_data, now_time: now_time}} <- observation_fun.() do
      {:ok, build_records(stations, now_data, past_data, now_time)}
    end
  end

  defp build_records(stations, now_data, past_data, now_time) do
    for {station_id, now_reading} <- now_data,
        past_reading = Map.get(past_data, station_id),
        not is_nil(past_reading),
        station = Map.get(stations, station_id),
        not is_nil(station),
        {:ok, record} <- [Observation.build_record(station, now_reading, past_reading, now_time)] do
      record
    end
  end
end
```

---

## Elixir的な着眼点(3): `for` 内包表記の魔法

`build_records/4` で使われている `for` は、TypeScriptの `for...of` ループとは**根本的に異なる**概念です。これを**内包表記**と呼びます。

```elixir
for {station_id, now_reading} <- now_data,
    past_reading = Map.get(past_data, station_id),
    not is_nil(past_reading),
    station = Map.get(stations, station_id),
    not is_nil(station),
    {:ok, record} <- [Observation.build_record(...)] do
  record
end
```

`for` ブロックには複数の「句」がカンマ区切りで並んでいます。これらは大きく分けて3つの役割を持ちます。

1. **ジェネレータ (`<-` を使う)**: データソースから1つずつ要素を取り出します。`now_data` はマップなので、`{key, value}` のタプルとして展開されます（JSの `Object.entries` と同じ）。
2. **フィルタ (`=` や `not is_nil` など、真偽値を返す式)**: これが `false` と評価されると、**その要素はスキップされます**。ここがTSの `if (...) continue` の役割を果たします。
3. **最後の `{:ok, record} <- [...]`**: これが今回の最大のハイライトです。

### 「1要素リストを使ったジェネレータ」テクニック

`Observation.build_record` は正常なら `{:ok, record}` を、データ不足なら `:skip` を返します。
これを素直に `for` の中に書くとどうなるでしょう？

```elixir
# これはエラーになる（または意図しない動作になる）
result = Observation.build_record(...) 
{:ok, record} <- result 
```

`for` の中で `<-` の左辺に置けるのは「繰り返し可能（Enumerable）なデータ」だけです。`{:ok, record}` というタプルは繰り返せないためエラーになります。

そこで、**戻り値を `[ ]` でリストに包んで1要素にする**のがElixirの定番テクニックです。

```elixir
{:ok, record} <- [Observation.build_record(...)]
```

- 関数が `{:ok, record}` を返した場合：`[{:ok, record}]` となり、パターンマッチして `record` が取り出され、`do ... end` のブロックが実行されます。
- 関数が `:skip` を返した場合：`[:skip]` となり、`{:ok, record}` という形と一致しないため、**マッチ失敗となり自動的にスキップされます**。

例外を投げることなく、エラーをデータとして扱いながらループのスキップを実現する、非常にエレガントな関数型のパターンです。

## Elixir的な着眼点(4): Contextと依存性注入の汎用性

`Weather` モジュールは、内部で `StationCache` (GenServer) や `JmaClient` (HTTP通信) を使っていますが、それらを**直接呼び出していません**。

```elixir
stations_fun = Keyword.get(opts, :stations_fun, &StationCache.get_stations/0)
observation_fun = Keyword.get(opts, :observation_fun, &JmaClient.fetch_observation_pair/0)
```

第3章でGenServerに対してやったのと全く同じ「依存性注入」を、普通の関数に対しても適用しています。
これにより、このContextのテストを書くとき、GenServerを起動したり、ネットワークに繋がったりすることなく、純粋に「データを組み立てるロジック」だけを検証できます。

**Contextの役割**: LiveView（次章で登場）のようなUI層は、`Weather.fetch_pressure_diffs()` という関数を呼ぶだけで済みます。裏でキャッシュがどう動いているか、APIがどう叩かれているか、一切知る必要がありません。これが「関心事の分離」です。

---

## 動作確認: iexで叩く

```bash
iex -S mix
```

```elixir
iex> alias WeatherEx.Weather
iex> {:ok, records} = Weather.fetch_pressure_diffs()
iex> length(records)
900  # 実データ件数(観測地点数によって変動)
iex> hd(records)
%{name: "宗谷地方気象台", enname: "Soya", lat: 45.416..., slp: 1012.3, diff: -0.5, ...}
```

900件近いデータが、ネットワーク通信とキャッシュを経由して正しく組み立てられているのが確認できます。

---

## テストコード: ロジックと統合を切り離して検証する

依存性注入のおかげで、テストは劇的にシンプルになります。

### `Observation.build_record/4` 単体のテスト

```elixir
# test/weather_ex/weather/observation_build_record_test.exs より
test "slp・diffは海面更正気圧の計算式に沿った値になる(過去の気温をslpPastに使う)" do
  @station %{"lat" => [35, 41], "lon" => [139, 46], "elevation" => 25, "kjName" => "東京"}
  now = %{"pressure" => [1005.2, 0], "temp" => [28.5, 0]}
  past = %{"pressure" => [1006.0, 0], "temp" => [26.0, 0]}  # 過去の気温をセット

  {:ok, record} = Observation.build_record(@station, now, past, "t")

  # 現在の気圧・現在の気温で計算
  expected_slp_now = Observation.sea_level_pressure(1005.2, 25.0, 28.5)
  # 過去の気圧・過去の気温で計算（ここが正しい仕様）
  expected_slp_past = Observation.sea_level_pressure(1006.0, 25.0, 26.0) 
  expected_diff = Float.round(expected_slp_now - expected_slp_past, 1)

  assert record.slp == expected_slp_now
  assert record.diff == expected_diff
end

test "過去データが無い、または気圧がnilなら:skipになる" do
  now = %{"pressure" => [1005.2, 0]}
  past = %{"pressure" => [nil, 0]} # 欠測
  
  assert :skip == Observation.build_record(%{}, now, past, "t")
end
```

### `Weather.fetch_pressure_diffs/1` の統合テスト

```elixir
# test/weather_ex/weather_test.exs より
test "過去データに存在しない地点は結果から除外される(元のTSのcontinueに相当)" do
  stations_fun = fn -> {:ok, %{"47759" => %{"kjName" => "東京", "lat" => [35, 41], "lon" => [139, 46]}}} end

  observation_fun = fn ->
    {:ok,
     %{
       now: %{
         "47759" => %{"pressure" => [1005.2, 0]},
         "99999" => %{"pressure" => [1008.0, 0]} # 過去データが無い地点
       },
       past: %{"47759" => %{"pressure" => [1006.0, 0]}},
       now_time: "t"
     }}
  end

  # ダミー関数を注入
  {:ok, records} =
    Weather.fetch_pressure_diffs(stations_fun: stations_fun, observation_fun: observation_fun)

  # 99999は除外され、東京の1件だけになる
  assert length(records) == 1
  assert hd(records).name == "東京"
end
```

**実行結果(検証済み)**:
```
6 doctests, 29 tests, 0 failures
```

---

## つまずきやすい点

- **`for` の最後のジェネータでリスト `[ ]` を忘れないこと**。`{:ok, record} <- Observation.build_record(...)` と書くと「タプルを繰り返そうとした」としてエラーになります。単一の値を `for` のフィルタとして使いたいときは必ず `[値]` で包みます。
- **JSの `??`（Null合体演算子）の連鎖は `||` で書けるが、意味が少し違う**。`elevation || altitude || 0` は `nil` と `false` のみをフォールバックの対象にします。もし `elevation` に `0` が入っていた場合、JSの `??` は `0` を返しますが、Elixirの `||` は右辺の `altitude` を評価してしまいます。今回は標高が `0` の地点はあり得ないため `||` で問題ありませんが、`0` を正当な値として扱いたい場合は専用のマクロ等が必要になります。
- **ガード(`when`)の中で関数を呼べない**。`when not is_nil(x)` はOKですが、`when custom_validator(x) == :ok` のような書き方はコンパイルエラーになります。

---

## 章末チェックリスト

- [ ] 「命令型のループ」が「宣言型の内包表記」にどう変わるかを理解した
- [ ] `Observation` モジュールに `build_record/4` を追加し、`with` のガードと `else _ -> :skip` を体験した
- [ ] `lib/weather_ex/weather.ex` (Context) を作成し、依存性注入のパターンを関数に適用した
- [ ] `for` 内包表記の `[{:ok, record}] <- [func()]` というテクニックの意味を理解した
- [ ] テストコードで、ネットワークやGenServerを使わずにロジックだけを検証できることを確認した
- [ ] `mix test` が通過した
- [ ] `iex -S mix` で実データを取得し、正しくレコードが組み立てられることを確認した

---

前章: [第3章 — GenServerで観測地点マスタをキャッシュ](./chapter03_station_cache.md)
次章: 第5章 — LiveViewの骨組み(時計・カウントダウン)
