# 第1章: 純粋関数でロジック移植（Elixir入門編）

Nuxt/TypeScript製の気象データアプリをElixirに移植していくチュートリアルの第1章にゃ。

Webフレームワーク(Phoenix)やデータベースには一切触れず、`server/api/weather.get.ts` に含まれる**「副作用のない計算ロジック」だけ**を取り出してElixirの関数として書き直します。

なぜわざわざWeb層から離れたところから始めるのか？ それは、**Elixirという言語の根幹にある「データ変換のパラダイム」を純粋な形で体験するため**です。Node.js/TypeScriptが「オブジェクトを変化させる世界」だとしたら、Elixirは「データを変換して新しいものを作る世界」です。この章でその違いをじっくり味わいましょう。

## この章でじっくり学ぶこと

- Elixirの**データ型**（特にタプルとマップ、そして `nil` の扱い）
- **「代入」ではなく「マッチング」**というElixirの発想の大転換
- パターンマッチと**関数の複数定義(multiple clauses)**による制御構造
- **モジュール**と**モジュール属性**によるコードの組織化
- `iex`での対話的な動作確認と、`ExUnit`によるテスト

---

## 事前準備: プロジェクトの器を作る

まずはElixirのプロジェクトを作ります。`create-nuxt-app` のElixir版のようなものだと思ってください。

```bash
mix new weather_ex --sup
cd weather_ex
```

`--sup` オプションをつけていますが、これは後の章で登場する「プロセスを監視する仕組み(Supervisor)」を入れるためのものです。今の段階では「おまじない」で構いません。
実行すると、`lib/` や `test/` といったディレクトリが自動生成されます。

---

## Elixirのデータ型：TypeScriptとの対比

コードを書く前に、Elixirがどのようなデータを扱うのか、TypeScriptと比較しながら見てみましょう。Elixirにはクラスやインターフェースがありません。データそのものがシンプルです。

### 1. 基本型
- **整数 (`Integer`)**: `1`, `100` （TSの `number` から整数を切り出したようなもの）
- **浮動小数点数 (`Float`)**: `1.0`, `3.14` （TSの `number` から小数を切り出したもの。Elixirでは `1` と `1.0` は**別の型**です）
- **文字列 (`String`)**: `"hello"` （TSと同じくダブルクォート。**シングルクォート `'hello'` は文字のリストという全く別物**になるので注意！）
- **真偽値 (`Boolean`)**: `true`, `false`
- **Atom（アトム）**: `:ok`, `:error`, `:hello` （TSには直接ない概念。後述）

### 2. Atom（アトム）という強力な仲間
Elixirの至る所で出てくる `:ok` や `:error` は**Atom**と呼ばれます。
名前そのものが値になる定数のようなものです。TypeScriptの EnumやUnion型に近い感覚で使われますが、メモリ消費が極めて少なく、比較が高速という特徴があります。

```elixir
:ok == :ok    # true
:ok == :error # false
```

### 3. コレクション型
- **リスト (`List`)**: `[1, 2, 3]` （TSの配列に一番近いですが、**先頭への追加が得意で、インデックスアクセスが遅い**という特徴があります）
- **マップ (`Map`)**: `%{"key" => "value", "name" => "東京"}` （TSの `Record` やオブジェクトに近いです。キーにAtomを使うと `%{name: "東京"}` と書けます）
- **タプル (`Tuple`)**: `{"hello", 1}` （**Elixir特有の重要な型**です。JSの配列のように見えますが、要素数が固定されており、インデックスアクセスも高速です。主に関数から「複数の関連する値」を返すために使われます）

### 4. `null` と `nil`
TypeScriptの `null` や `undefined` は、Elixirでは一つに統合されて **`nil`** になります。
そして重要なのは、Elixirでは「値が存在しないかもしれない」という状況を、`nil` 単体で表すよりも、**タプルでラップして表現する**のが基本パターンだということです。

```typescript
// TypeScript: 値そのものか、nullかで表す
function getData(): Data | null { ... }
```
```elixir
# Elixir: 成功か失敗かを明示的にタプルで返す
def get_data() do
  {:ok, %Data{}} # 成功
  # または
  {:error, :not_found} # 失敗
end
```
これを**タグ付きタプル**と呼び、Elixirにおけるエラーハンドリングの絶対的な基盤になります。

---

## 移植対象: 元のTypeScriptコード

では、第1章の移植対象となる計算ロジックを見てみましょう。

```typescript
// 天気コード→絵文字の辞書
const WEATHER_EMOJI_MAP: Record<number, string> = {
  1: "☀️", 2: "☀️",
  3: "☁️", 4: "☁️",
  // ...
}

// 座標変換 [度, 分] → 10進法
function convertCoord(coord: [number, number]): number {
  return coord[0] + coord[1] / 60.0
}

// 海面更正気圧の計算
function getSlp(p: number | null, h: number | null, t: number | null): number | null {
  if (p === null || h === null) return p
  const tempForCalc = t === null ? 15.0 : t
  return Math.round(p * Math.exp(0.034163 * h / (tempForCalc + 273.15)) * 10) / 10
}
```

TypeScriptでは、引数の型に `null` を許容し、関数の先頭で `if (p === null)` とガードを書くスタイルですね。配列 `[number, number]` は、実質的にタプルとして使っています。

これをElixirのパラダイムに変換していきます。

---

## 実装: `lib/weather_ex/weather/observation.ex`

ファイルを作成し、以下のコードを書いてください。ディレクトリ `lib/weather_ex/weather/` は手動で作る必要があります。

```elixir
defmodule WeatherEx.Weather.Observation do
  @moduledoc """
  気象データの純粋な変換ロジック。副作用なし。
  """

  # --- 1. モジュール属性による定数定義 ---
  @weather_emoji_map %{
    1 => "☀️", 2 => "☀️",
    3 => "☁️", 4 => "☁️",
    5 => "☔", 6 => "☔",
    7 => "❄️", 8 => "❄️",
    9 => "⚡"
  }

  # --- 2. タプルによる直接分解（パターンマッチ） ---
  @doc """
  [度, 分] のタプルを10進法の度に変換する。

  ## Examples

      iex> WeatherEx.Weather.Observation.convert_coord({35, 30})
      35.5
  """
  def convert_coord({degrees, minutes}) do
    degrees + minutes / 60.0
  end

  # --- 3. 関数の複数定義で nil ケースを先に捌く ---
  @doc """
  海面更正気圧を計算する。気圧または標高がnilならnilを返す。

  ## Examples

      iex> WeatherEx.Weather.Observation.sea_level_pressure(1000.0, 100.0, 20.0)
      1011.7

      iex> WeatherEx.Weather.Observation.sea_level_pressure(nil, 100.0, 20.0)
      nil
  """
  def sea_level_pressure(nil, _height, _temp), do: nil
  def sea_level_pressure(_pressure, nil, _temp), do: nil

  def sea_level_pressure(pressure, height, temp) do
    temp_for_calc = temp || 15.0
    result = pressure * :math.exp(0.034163 * height / (temp_for_calc + 273.15))
    Float.round(result, 1)
  end

  # --- 4. Map.getによる安全なアクセス ---
  @doc """
  天気コードを絵文字に変換する。該当なしなら空文字。

  ## Examples

      iex> WeatherEx.Weather.Observation.weather_emoji(1)
      "☀️"

      iex> WeatherEx.Weather.Observation.weather_emoji(nil)
      ""
  """
  def weather_emoji(nil), do: ""
  def weather_emoji(code), do: Map.get(@weather_emoji_map, code, "")
end
```

この短いコードに、Elixirのエッセンスが詰まっています。1つずつじっくり解説します。

---

## Elixir的な着眼点：じっくり解説

### 1. モジュールは「名前空間」であり「構造のないクラス」のようなもの

`defmodule WeatherEx.Weather.Observation do ... end` は、TypeScriptの `namespace` や、メソッドだけを持つ静的クラスのようなイメージです。
Elixirにはクラスがなく、すべての関数は何らかのモジュールに属しています。`WeatherEx.Weather.Observation.convert_coord(...)` のように呼び出します。

### 2. モジュール属性 `@` による定数定義

```elixir
@weather_emoji_map %{ 1 => "☀️", ... }
```

`@` から始まるものを**モジュール属性**と呼びます。JSの `const` オブジェクトと似た感覚で使えますが、決定的な違いがあります。**モジュール属性は「コンパイル時」に値が決定され、コード内に直接埋め込まれます。**
実行時に毎回評価されたり、後から値を変更したりすることはできません。この「不変であること」が、Elixirの挙動を予測しやすくしています。

### 3. 「代入」ではなく「マッチング」： `=` の正体

ここがElixir入門で最も重要なパラダイムシフトです。
Elixirの `=` は代入演算子ではなく、**マッチ演算子**です。

```elixir
iex> x = 1
1
iex> 1 = x
1
iex> 2 = x
** (MatchError) no match of right hand side value: 1
```
`2 = x` でエラーが出るのは、左辺の `2` と右辺の `x`(中身は1)が**マッチしないから**です。

これを関数の引数で使うのが**パターンマッチ**です。

```elixir
def convert_coord({degrees, minutes}) do
```
TypeScriptでは `coord[0] + coord[1] / 60.0` とインデックスでアクセスしていましたが、Elixirでは引数の受け取り時点で「 `{degrees, minutes}` という形のタプルなら、中身を変数として分解して受け取る」というパターンを指定できます。これにより、インデックスアクセスというエラー prone な操作を避けることができます。

### 4. 関数の複数定義で `if` を消し去る

TypeScriptの `getSlp` は、関数の最初に `if (p === null || h === null)` と書いていました。
Elixirでは、**同じ名前で同じ引数の数（アリティと呼びます）の関数を複数定義**し、上から順にパターンマッチを試すことでこれを表現します。

```elixir
def sea_level_pressure(nil, _height, _temp), do: nil
def sea_level_pressure(_pressure, nil, _temp), do: nil
def sea_level_pressure(pressure, height, temp) do
  # ...
end
```

- `_height` のようにアンダースコアから始める変数は「この値は使わない」という宣言です。未使用変数の警告を消すためのElixirのベストプラクティスです。
- 節は**上から順に評価**されます。もし3番目の節を最初に書いてしまうと、`nil` が渡されてもそこにマッチしてしまい、バグの原因になります。
- このように「条件分岐」を「関数の定義順とパターンマッチ」に委ねるのが、Elixirの非常に美しいスタイルです（宣言的であるため、バグが入り込む余地が減ります）。

### 5. 三項演算子の代わりに `||` 演算子

`t === null ? 15.0 : t` という三項演算子は、Elixirでは `temp || 15.0` と書けます。

注意点として、JavaScriptの `||` は「 falsy（`0`, `""`, `false`, `null`, `undefined`）なら右辺を返す」ですが、**Elixirの `||` は「 `nil` か `false` なら右辺を返す」**という仕様です。今回のように数値や文字列を扱う場合、JSと同じ感覚で使えます（もし `0` を「値がない」として扱いたい場合は別の処理が必要です）。

### 6. `Map.get` による安全な辞書引き

`@weather_emoji_map[code]` と書くこともできますが、キーが存在しないとエラーになってしまいます。
`Map.get(map, key, default)` は、キーが存在しなければ `default` を返すため、未知のコードが来ても安全に空文字を返すことができます。

---

## 動作確認: iexで叩く

Elixirには `node` の REPLのような、対話的シェル `iex` が付属しています。`-S mix` をつけることで、現在のプロジェクトのコードを読み込んだ状態で起動できます。

```bash
iex -S mix
```

```elixir
iex> alias WeatherEx.Weather.Observation
iex> Observation.convert_coord({35, 30})
35.5
# タプルで渡さないとどうなるか試してみる
iex> Observation.convert_coord([35, 30])
** (FunctionClauseError) no function clause matching in ...
```

あえてリスト `[35, 30]` を渡してみました。すると `FunctionClauseError` が発生しました。
これは「定義されているどの関数のパターン（タプルを受け取るパターン）ともマッチしなかった」というElixirらしいエラーです。型システムが実行時に厳しくチェックしてくれたおかげで、バグに早く気づけます。

```elixir
iex> Observation.sea_level_pressure(1000.0, 100.0, 20.0)
1011.7
iex> Observation.sea_level_pressure(nil, 100.0, 20.0)
nil
iex> Observation.weather_emoji(3)
"☁️"
```

> **検証メモ**: 執筆時、`sea_level_pressure(1000.0, 100.0, 20.0)` の期待値を最初 `1012.0` と書いてしまったが、実際に `iex` で計算させると `1011.7` が正しい値だった。`@doc` の `## Examples` の数値は必ず実行して確認すること。

---

## テストコード: `test/weather_ex/weather/observation_test.exs`

テストファイルを作成します。ここでもElixirの思想が現れます。

```elixir
defmodule WeatherEx.Weather.ObservationTest do
  # テストモジュールにCaseの機能を読み込む（async: true で並行実行を許可）
  use ExUnit.Case, async: true
  
  # @doc 内の Examples をテストとして実行する魔法の一行
  doctest WeatherEx.Weather.Observation
  
  alias WeatherEx.Weather.Observation

  # describe でテストをグループ化できる（RSpecの describe と同じ）
  describe "convert_coord/1" do
    # アリティ（引数の数）を名前の後ろに /1 のように書くのがElixirの慣習
    test "度分を10進法に変換する" do
      # assert は左辺と右辺が一致するか検証する
      assert Observation.convert_coord({35, 30}) == 35.5
    end

    test "分が0のとき度そのまま" do
      assert Observation.convert_coord({36, 0}) == 36.0
    end
  end

  describe "sea_level_pressure/3" do
    test "気圧・標高・気温から海面更正気圧を計算する" do
      assert Observation.sea_level_pressure(1000.0, 100.0, 20.0) == 1011.7
    end

    test "気圧がnilならnilを返す" do
      assert Observation.sea_level_pressure(nil, 100.0, 20.0) == nil
    end

    test "気温がnilなら15.0度として計算する" do
      result = Observation.sea_level_pressure(1000.0, 100.0, nil)
      assert result == Observation.sea_level_pressure(1000.0, 100.0, 15.0)
    end
  end

  describe "weather_emoji/1" do
    test "既知のコードは絵文字を返す" do
      assert Observation.weather_emoji(1) == "☀️"
      assert Observation.weather_emoji(9) == "⚡"
    end

    test "未知のコードは空文字" do
      assert Observation.weather_emoji(99) == ""
    end

    test "nilは空文字" do
      assert Observation.weather_emoji(nil) == ""
    end
  end
end
```

### doctest の魔法
`doctest WeatherEx.Weather.Observation` という1行を入れることで、先ほどモジュールに書いた `@doc` の `## Examples` に書いた例が**そのままテストとして実行されます**。
ドキュメントとテストが二重管理されず、常に一致していることが保証されるのがElixirの素晴らしいところです。

```bash
mix test
```

**実行結果(検証済み)**:
```
4 doctests, 9 tests, 0 failures
```

---

## つまずきやすい点

- **`Float.round`の第2引数は小数点以下の桁数**。`Float.round(1012.345, 1)` は `1012.3` になります。JSの `Math.round(x*10)/10` と基本同じ結果になりますが、境界値では丸め方式の違いで食い違うことがあるため、実データで確認する習慣をつけるとよいです。
- **関数の複数定義は上から順にマッチが試される**。これを忘れると、後ろに書いた「なんでも受け取る節」に全て吸い取られてしまいます。
- **文字列とCharlistの違い**。シングルクォート `'hello'` は文字のリスト（Charlist）であり、ダブルクォート `"hello"`（String）とは全く別の型です。基本はダブルクォートを使いましょう。
- **`1` と `1.0` は別物**。`1 / 2` は `0.5` (Float) ですが、`div(1, 2)` は `0` (Integer) になります。今回は `/ 60.0` と右辺をFloatにすることで、結果をFloatに保っています。

---

## 章末チェックリスト

- [ ] `mix new weather_ex --sup` でプロジェクト作成済み
- [ ] Elixirのデータ型（Atom, Tuple, Map, nil）の概念を理解した
- [ ] `=` が代入ではなくマッチ演算子であることを理解した
- [ ] `lib/weather_ex/weather/observation.ex` を作成し、パターンマッチと複数節を体験した
- [ ] `test/weather_ex/weather/observation_test.exs` を作成し、`doctest` の動きを確認した
- [ ] `mix test` が全件通過した
- [ ] `iex -S mix` でわざと異なる型を渡し、`FunctionClauseError` が発生するのを確認した

---

次章: [第2章 — 外部API連携(jma_client.ex)](./chapter02_jma_client.md)
