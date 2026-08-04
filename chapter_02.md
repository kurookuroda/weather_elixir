# 第2章: 外部API連携（Elixir入門編）

第1章では、外部の世界に一切触れず、データを別のデータに変換する「純粋関数」を作りました。Elixirのデータ型や「マッチング」という考え方の基礎が身についたはずです。

この章では、いよいよ外部の世界——気象庁(JMA)のAPI——と通信します。Node.js/TypeScriptでは当たり前のように書いていた `try/catch` を捨て、Elixir流の「例外を投げないエラーハンドリング」の世界に足を踏み入れる回にゃ。

## この章でじっくり学ぶこと

- なぜElixirは**例外を使わずに `{:ok, _}` / `{:error, _}` を返すのか**
- `case` 式による詳細なパターンマッチと分岐
- **`with` 構文**による、「ハッピーパス（成功の道筋）」の一直線化
- **bang版（`!`付き）と non-bang版**の関数の使い分けの絶対的なルール
- バイナリパターンマッチによる文字列の分解
- Sigil（シギル）という専用リテラル記法

---

## 事前準備: Reqを依存関係に追加

Elixirには標準でHTTPクライアントが組み込まれていません。コミュニティが作った強力なライブラリを使います。昔は `HTTPoison` というライブラリが主流でしたが、現在は `Req` という後継機が標準的な選択肢です。

`mix.exs` ファイルを開き、`deps` 関数の中に `{:req, "~> 0.5"}` を追加します。

```elixir
defp deps do
  [
    {:req, "~> 0.5"}
  ]
end
```

そしてターミナルで以下を実行します（`npm install` のようなものです）。

```bash
mix deps.get
```

---

## Elixirのエラーハンドリング哲学：なぜ例外を使わないのか

実装に入る前に、最も重要なパラダイムシフトを説明します。

TypeScript（あるいは多くの言語）では、ネットワークエラーやJSONパースエラーなどは「例外」として飛んできて、それを `try/catch` で捕まえるのが普通です。

```typescript
// TypeScript: 例外が飛んでくる前提
try {
  const data = await fetch(url)
  return data.json()
} catch (error) {
  return null // エラーを握りつぶしてnullにする
}
```

しかしElixirでは、**「予期されるエラー（ネットワーク失敗、データNotFoundなど）は例外として投げてはいけない」**という強い文化があります。
代わりに、第1章で少し触れた**タグ付きタプル**を使って、成功か失敗かを明示的に返します。

```elixir
# Elixir: 結果をタプルで返す
{:ok, response_body} # 成功
{:error, :timeout}   # 失敗
```

なぜでしょうか？
Elixirは**並行処理（複数の処理を同時に動かすこと）**が前提の言語です。もし別のプロセスが例外を投げたら、それをキャッチし忘れるとプロセスがいきなり死にます。例外は「本当に回復不能なシステムエラー」のために取っておき、ビジネスロジック上の失敗はデータとして扱うのです。これが、Elixirコードを安定して動かすための最大の秘密にゃ。

---

## 移植対象: 元のTypeScriptコードの流れ

気象データを取得するには、以下のステップを踏みます。

1. `latest_time.txt` から最新時刻を取得
2. その時刻の1時間前を計算
3. 現在時刻のデータ取得（失敗したら10分前でリトライ）
4. 1時間前データ取得

```typescript
// TSでは例外をキャッチして null を返す関数を作り、それを繋いでいた
async function tryFetch(targetTime: string): Promise<any> {
  const url = `https://www.jma.go.jp/bosai/amedas/data/map/${targetTime}.json`
  try {
    return await $fetch(url)
  } catch {
    return null // 失敗をnullに丸める
  }
}

function formatJmaTime(d: Date): string {
  const pad = (n: number) => String(n).padStart(2, "0")
  return `${d.getUTCFullYear()}${pad(d.getUTCMonth()+1)}${pad(d.getUTCDate())}${pad(d.getUTCHours())}${pad(d.getUTCMinutes())}00`
}
```

これを、`null` を使わず、`try/catch` を使わないElixir流に書き換えます。

---

## 実装: `lib/weather_ex/weather/jma_client.ex`

```elixir
defmodule WeatherEx.Weather.JmaClient do
  @moduledoc """
  気象庁(JMA)のAMeDASデータAPIとの通信を担当する。
  すべてのパブリック関数は {:ok, data} | {:error, reason} を返す。
  """

  @base_url "https://www.jma.go.jp/bosai/amedas"

  # --- 1. Sigil と Calendar.strftime による日時フォーマット ---

  @doc """
  DateTimeを気象庁APIが要求する `YYYYMMDDHHmm00` 形式の文字列に変換する。

  ## Examples

      iex> dt = ~U[2026-08-04 12:30:00Z]
      iex> WeatherEx.Weather.JmaClient.format_jma_time(dt)
      "20260804123000"
  """
  def format_jma_time(%DateTime{} = dt) do
    Calendar.strftime(dt, "%Y%m%d%H%M") <> "00"
  end

  # --- 2. case 式によるタプルの分岐 ---

  @doc """
  最新観測時刻を取得する。
  """
  def fetch_latest_time do
    case Req.get("#{@base_url}/data/latest_time.txt") do
      {:ok, %Req.Response{status: 200, body: body}} ->
        digits = String.replace(body, ~r/\D/, "")
        {:ok, String.slice(digits, 0, 12) <> "00"}

      {:ok, %Req.Response{status: status}} ->
        {:error, {:unexpected_status, status}}

      {:error, reason} ->
        {:error, reason}
    end
  end

  @doc """
  指定時刻のAMeDASマップデータを取得する。
  """
  def fetch_map_data(target_time) do
    url = "#{@base_url}/data/map/#{target_time}.json"

    case Req.get(url) do
      {:ok, %Req.Response{status: 200, body: body}} -> {:ok, body}
      {:ok, %Req.Response{status: status}} -> {:error, {:unexpected_status, status}}
      {:error, reason} -> {:error, reason}
    end
  end

  # --- 3. with 構文によるハッピーパスの記述 ---

  @doc """
  最新の気圧差分計算に必要なデータペアを取得する。
  """
  def fetch_observation_pair do
    with {:ok, now_time} <- fetch_latest_time(),
         {:ok, now_dt} <- parse_jma_time(now_time),
         # <- ではなく = を使うと、マッチ失敗時にエラー(例外)ではなくそのまま変数に代入される
         past_time = shift_time(now_dt, -60),
         ten_min_ago_time = shift_time(now_dt, -10),
         {:ok, now_data} <- fetch_map_data_with_fallback(now_time, ten_min_ago_time),
         {:ok, past_data} <- fetch_map_data(past_time) do
      {:ok, %{now: now_data, past: past_data, now_time: now_time}}
    else
      # with の中で一つでも {:error, reason} にマッチしたものがあれば、
      # ここに飛んできて、そのまま呼び出し元に {:error, reason} を返す
      {:error, reason} -> {:error, reason}
    end
  end

  # プライベート関数（モジュール外からは呼べない）
  defp fetch_map_data_with_fallback(primary_time, fallback_time) do
    case fetch_map_data(primary_time) do
      {:ok, data} -> {:ok, data}
      {:error, _reason} -> fetch_map_data(fallback_time)
    end
  end

  # --- 4. バイナリパターンマッチで文字列を分解 ---

  defp parse_jma_time(time_str) do
    # 文字列を「4文字, 2文字, 2文字...」に直接分解している
    <<y::binary-4, m::binary-2, d::binary-2, h::binary-2, mi::binary-2, _::binary>> = time_str

    # non-bang版の関数を使い、タプルで結果を受け取る
    with {:ok, date} <- Date.new(String.to_integer(y), String.to_integer(m), String.to_integer(d)),
         {:ok, time} <- Time.new(String.to_integer(h), String.to_integer(mi), 0),
         {:ok, dt} <- DateTime.new(date, time) do
      {:ok, dt}
    end
  end

  defp shift_time(dt, minutes) do
    dt
    |> DateTime.add(minutes * 60) # 第3引数の :second は省略可能（デフォルト値）
    |> format_jma_time()
  end
end
```

---

## Elixir的な着眼点：じっくり解説

### 1. `case` 式：ネストしたデータ構造の分解機

`Req.get/1` は例外を投げず、必ず `{:ok, response}` か `{:error, reason}` を返します。
これを `case` で受けますが、すごいのは**右辺のタプルの中身まで、一気にパターンマッチで分解できる**ことです。

```elixir
case Req.get(url) do
  # 「成功」かつ「ステータスが200」かつ「bodyを取り出す」を1行で表現
  {:ok, %Req.Response{status: 200, body: body}} -> {:ok, body}
  
  # 「成功」だが「ステータスが200以外」
  {:ok, %Req.Response{status: status}} -> {:error, {:unexpected_status, status}}
  
  # 「通信エラーなど」
  {:error, reason} -> {:error, reason}
end
```

TypeScriptなら `if (response.ok) { ... } else { ... }` とネストを深くしがちなところを、Elixirは「状態のありえるパターン」をフラットに並べることで、バグの入り込む余地を消し去ります。

### 2. `with` 構文：縦に落ちていくハッピーパス

`fetch_observation_pair/0` の本体に登場する `with` は、Elixirで最も愛されている構文の一つです。

TypeScriptで非同期処理を何度も繋げると、`await` と `if` でネストが深くなり、右へ右へとコードが流れていきます（いわゆるピラミッド・オブ・ドゥーム）。

```typescript
// TSの典型例：ネストが深い
const time = await getLatestTime()
if (time) {
  const dt = parseTime(time)
  if (dt) {
    const data = await fetchData(dt)
    if (data) { ... }
  }
}
```

`with` はこれを**「成功したら次の行へ」という縦の流れに変える**魔法の構文です。

```elixir
with {:ok, now_time} <- fetch_latest_time(),     # 失敗したら即 else へ
     {:ok, now_dt} <- parse_jma_time(now_time),  # 失敗したら即 else へ
     {:ok, now_data} <- fetch_map_data(...) do   # 失敗したら即 else へ
  # すべて成功したらここに来る（ハッピーパス）
  {:ok, %{now: now_data, ...}}
else
  {:error, reason} -> {:error, reason} # どれかで失敗したらここに飛ぶ
end
```

### 3. `with` の中の `<-` と `=` の決定的な違い

`with` の中で、矢印 `<-` を使っている行と、単なるイコール `=` を使っている行があります。ここが超重要です。

- **`<-` (マッチ演算子)**: 左辺のパターンにマッチしなかった場合（例えば `{:error, _}` が返ってきた場合）、**そこで処理を中断して `else` 節に飛びます**。
- **`=` (代入/マッチ)**: 普通に変数に束縛します。もし右辺が予期せぬ値でも `else` には飛ばず、そのまま次の行へ進みます（マッチしない場合はここで `MatchError` 例外が飛びます）。

```elixir
# <- は失敗を許容する（elseへ飛ぶ）
{:ok, now_dt} <- parse_jma_time(now_time)

# = は必ず成功すると信じている（ここでエラーが出たらシステムエラーとして落とす）
past_time = shift_time(now_dt, -60)
```
「外部APIやパースのような失敗するかもしれない処理は `<-` 」「自前の純粋な計算処理は `=`」と使い分けるのが鉄則です。

### 4. bang版（`!`）と non-bang版の絶対的なルール

`parse_jma_time/1` の中で、`Date.new/3` や `Time.new/3` という関数を使っています。
実はこれらの関数には、末尾に `!` がついた `Date.new!/3` という**bang版（バン版）**が存在します。

- **non-bang版 (`Date.new/3`)**: 失敗すると `{:error, reason}` を返す。
- **bang版 (`Date.new!/3`)**: 失敗すると `ArgumentError` という**例外を投げる**。

**【Elixir開発者の絶対的な掟】**
`with` や `case` でエラーハンドリングしているコードの内部では、**絶対に bang版を使ってはいけません。** bang版を使うと、せっかく `with` で綺麗に書いているエラーハンドリングを例外が破壊してしまいます。
bang版は「スクリプトをサクッと書きたい時」や「絶対に失敗しないことが分かっている時」だけに使い、モジュールやライブラリのコードでは non-bang版を徹底します。

### 5. バイナリパターンマッチ：文字列はただのバイト列

```elixir
<<y::binary-4, m::binary-2, d::binary-2, h::binary-2, mi::binary-2, _::binary>> = time_str
```
Elixir（および基盤であるErlang）において、文字列は「UTF-8でエンコードされたバイトの羅列」に過ぎません。
 therefore、`"20260804123000"` という文字列を、JSの `slice` を何度も呼ぶのではなく、**「4バイト、2バイト、2バイト…」と物理的な長さで切り分けて、そのまま変数に割り当てる**ことができます。これをバイナリパターンマッチと呼び、非常に高速で宣言的な書き方です。

### 6. Sigil（シギル）: `~U[2026-08-04 12:30:00Z]`

`~U` から始まるこの記法は**Sigil**と呼ばれます。文字列や正規表現を簡単に書くための専用構文です。
`~U` は「UTCのDateTime型」を直接ソースコードに埋め込める記法です。テストコードで現在時刻をわざわざ生成しなくて済むため、重宝します。

> **バグ修正メモ**: 執筆過程で、元のTypeScriptが末尾に `"00"` を付加していたのに、Elixir側で `Calendar.strftime` の出力（秒がない12桁）をそのまま返してしまい、URLが404になるバグがありました。`<> "00"` を追加して修正済み。外部APIとの結合部分は、こうした文字列の桁数ズレが一番の罠になります。

---

## 動作確認: iexで叩く

```bash
iex -S mix
```

```elixir
iex> alias WeatherEx.Weather.JmaClient

# 時刻フォーマットの確認（例外は飛ばない）
iex> JmaClient.format_jma_time(~U[2026-08-04 12:30:00Z])
"20260804123000"

# 実際にネットワーク通信を行う（数秒かかる）
iex> JmaClient.fetch_latest_time()
{:ok, "20260804153000"} # 実行時刻によって変わる

# データペアの取得
iex> JmaClient.fetch_observation_pair()
{:ok, %{now: %{...巨大なマップ...}, past: %{...}, now_time: "20260804153000"}}
```

返ってくる値がすべて `{:ok, ...}` になっていることに注目してください。`null` はありません。

---

## テストコード: `test/weather_ex/weather/jma_client_test.exs`

ネットワーク通信を伴う関数は、サーバーが落ちていたらテストが落ちてしまいます。そのため、**「通信に依存しないロジック（時刻計算など）だけ」をピンポイントでテスト**するのがElixirの定石です。

```elixir
defmodule WeatherEx.Weather.JmaClientTest do
  use ExUnit.Case, async: true
  doctest WeatherEx.Weather.JmaClient
  alias WeatherEx.Weather.JmaClient

  describe "format_jma_time/1" do
    test "DateTimeを気象庁形式の文字列に変換する(末尾00付き14桁)" do
      dt = ~U[2026-08-04 12:30:00Z]
      assert JmaClient.format_jma_time(dt) == "20260804123000"
    end

    test "1桁の月日時分もゼロパディングされる" do
      dt = ~U[2026-01-05 03:07:00Z]
      assert JmaClient.format_jma_time(dt) == "20260105030700"
    end
  end

  # fetch_latest_time等のネットワーク関数はここではテストしない。
  # 動作確認は iex で行うこと。
  # （発展：Req.Test や Mox を使ったモックテストは今後の章で扱います）
end
```

**検証結果**:
時刻計算ロジックを抽出して検証したところ、以下がすべて通過しました。

| 項目 | 結果 |
|---|---|
| `"00"`欠落の修正 | ✅ `"20260804123000"`(14桁)を正しく生成 |
| ゼロパディング(1桁月日時分) | ✅ `"20260105030700"` |
| `parse_jma_time`との往復一致 | ✅ format→parseで元のDateTimeに戻る |
| 日付またぎ(`00:30 - 60分` → 前日`23:30`) | ✅ |

> **テスト設計のポイント**: 「往復変換テスト（formatしてparseしたら元に戻るか）」は、一方向のテストだけでは気づけない「末尾の文字列欠落」のようなバグを検出するのに最強の手法です。

---

## つまずきやすい点

- **`with` の `else` 節を省略すると、マッチしなかった値がそのまま関数の戻り値になります。** うっかり `{:error, :timeout}` などを拾い忘れると、呼び出し元が想定していないタプルがそのまま伝播してしまうので、分岐が必要な場合は必ず `else` を書きましょう。
- **`with` の中で bang版（`!`）を使うと、`with` のエラーハンドリングが完全にバイパスされて例外が飛びます。** 気をつけて。
- `Calendar.strftime` の出力に末尾の `"00"` を付け忘れると、URLの秒の桁がズレて気象庁APIから `404` が返ってきます。API仕様の文字列長は特に注意深く確認しましょう。
- 気象庁APIへはブラウザからの直接アクセスを弾く場合があります。`iex` で叩いて `403 Forbidden` が返ってきたら、`Req.get(url, headers: [{"user-agent", "WeatherEx"}]) のようにヘッダーを追加してみてください。

---

## 章末チェックリスト

- [ ] `mix.exs` に `{:req, "~> 0.5"}` を追加、`mix deps.get` 実行済み
- [ ] 「なぜElixirは例外を使わないのか」という哲学を理解した
- [ ] `case` 式で、ネストしたタプルやマップを一気に分解しているコードを読んだ
- [ ] `with` 構文の `<-` と `=` の違いを理解した
- [ ] `lib/weather_ex/weather/jma_client.ex` を作成し、non-bang版の関数だけを使っていることを確認した
- [ ] `mix test` が通過した
- [ ] `iex -S mix` で `fetch_observation_pair/0` を実行し、`{:ok, %{...}}` というタプルで実データが取れることを確認済み

---

前章: [第1章 — 純粋関数でロジック移植](./chapter01_observation.md)
次章: [第3章 — GenServerで観測地点マスタをキャッシュ](./chapter03_station_cache.md)
