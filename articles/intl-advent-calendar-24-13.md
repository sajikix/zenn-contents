---
title: "Intl.NumberFormat の基本(#13)"
emoji: "🧮"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: true
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 13 日目の記事です。

今回は Intl.NumberFormat について基本的な使い方と各メソッドについて解説します。また簡単に Intl.NumberFormat のユースケースについても触れます。

## 数値のフォーマット : Intl.NumberFormat

Intl.NumberFormat はロケールに応じた数値全般の書式化をサポートする機能です。純粋な数値のフォーマットはもちろんのこと、単位や通貨、パーセントを含んだ数値のフォーマットにも対応します。

### 基本的な使い方

他の Intl のコンストラクタプロパティ同様、第１引数にロケール(ロケール識別子 or Intl.Locale オブジェクト)を第２引数にフォーマットのオプションを渡して初期化することで、Intl.NumberFormat インスタンスを生成できます。

```ts
const jaFormatter = new Intl.NumberFormat("ja-JP", {
  // オプションを指定
});
```

生成した Intl.NumberFormat インスタンスには Intl.DateTimeFormat 同様 `format()` / `formatToParts()` / `formatRange()` / `formatRangeToParts()` のようなメソッドが生えており(詳しくは記述)、このメソッドに数値を渡すことで書式化できます。

```ts
jaFormatter.format(12345.6); // '12,345.6'
```

## NumberFormat インスタンスの持つメソッド

Intl.NumberFormat インスタンスには以下の 5 つのメソッドがあります。

- `format()`
- `formatToParts()`
- `formatRange()`
- `formatRangeToParts()`
- `resolvedOptions()`

気づいた方も多いでしょうが基本的には Intl.DateTimeFormat と同じメソッドを持っています。Intl.DateTimeFormat の持つメソッドについては[7 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-7)を参照してください。

### `format()` と `formatToParts()`

`format()` メソッドは Number または BigInt を渡すことで、初期化時に設定したオプションから書式化した文字列を返すメソッドです。

```ts
new Intl.NumberFormat("ja-JP").format(12345.6); // '12,345.6'
```

ちなみに引数を指定しない場合は `NaN` を渡したのと同じ挙動になります。

```ts
new Intl.NumberFormat("ja-JP").format(); // 'NaN'
```

`formatToParts()` は返り値を書式化された文字列ではなく文字列を構成する部品の配列で返します。配列の各要素は `{type: string, value: string}` というオブジェクトで、`type` には要素の種別、`value` には実際の文字列が入ります。

```ts
new Intl.NumberFormat("ja-JP").formatToParts(12345.6);
// [
//   { type: "integer", value: "12", },
//   { type: "group", value: ",", },
//   { type: "integer", value: "345", },
//   { type: "decimal", value: ".", },
//   { type: "fraction", value: "6", },
// ];
```

この `type` ですが以下のように書式化のパターンやオプションによって様々なパターンが存在します。

- `integer` : 整数部分
- `fraction` : 小数部分
- `group` : `","` のようなグループ化のための記号
- `decimal` : 小数点の記号 (`"."`)
- `plusSign` : 正の数のプラス記号
- `minusSign` : 負の数のマイナス記号
- `percentSign` : パーセント記号(`style` オプションが percent の場合)
- `currency` : 通貨記号(`style` オプションが currency の場合)
- `unit` : 単位(`style` オプションが unit の場合)
- `nan` : `NaN` の場合の文字列(`"NaN"`)
- `infinity` : `Infinity` の場合の文字列(`"∞"`)
- `exponentInteger` : 指数表記の整数部分
- `exponentMinusSign` : 指数表記のマイナス記号
- `exponentSeparator` : 指数表記の区切り文字
- `literal` : 上記以外の文字列
- `unknown` : 未知の文字列

`formatToParts()` を使って値を分類しカスタマイズするときは適切な type を指定できているか注意しましょう。

### `formatRange()` と `formatRangeToParts()`

`formatRange()` メソッドは `"200 ~ 300` のような数値の範囲を表す文字列を返すメソッドです。そのため、引数には startRange と endRange の 2 つの Number / BigInt を受け取ります。

```ts
new Intl.NumberFormat("ja-JP").formatRange(200, 300); // '200~300'
```

`formatRange()` メソッドは基本的に書式化された数値を `"-"` や `"～"` でつないだ文字列を返しますが、オプションで指定された通りに丸めた結果 startRange と endRange が一緒になる場合は `"~[number]"` のように表示されます。

```ts
// 2.9も3.1もオプション通り少数部分を丸めると3になるため、"〜3" と表示される
new Intl.NumberFormat("ja-JP", { maximumFractionDigits: 0 }).formatRange(
  2.9,
  3.1
); // '〜3'
```

また通貨や単位などに関しては startRange と endRange 両方につけるかまとめるかなどをロケールや通貨に応じて調整してくれます。

```ts
new Intl.NumberFormat("es-ES", {
  style: "currency",
  currency: "EUR",
}).formatRange(3, 5); // '3,00-5,00 €'
new Intl.NumberFormat("es-US", {
  style: "currency",
  currency: "USD",
}).formatRange(3, 5); // '$3.00 - $5.00'
```

`formatRangeToParts()` は `formatRange()` と同じく startRange と endRange を受け取り、文字列を構成する部品の配列を返します。配列の各要素は `{type: string, value: string, source: "startRange" | "endRange" | "shared"}` というオブジェクトで、`source` にはその要素が startRange か endRange か(あるいはどちらでもない部分か)を示す情報が入ります。

```ts
new Intl.NumberFormat("ja-JP").formatRangeToParts(12.3, 45.6);
// [
//   { type: "integer", value: "12", source: "startRange",},
//   { type: "decimal", value: ".", source: "startRange",},
//   { type: "fraction", value: "3", source: "startRange",},
//   { type: "literal", value: "～", source: "shared",},
//   { type: "integer", value: "45", source: "endRange",},
//   { type: "decimal", value: ".", source: "endRange",},
//   { type: "fraction", value: "6", source: "endRange",},
// ];
```

### `resolvedOptions()`

`resolvedOptions()` では Intl.DateTimeFormat などと同様インスタンスの初期化時に指定したオプションを取得できます。暗黙的に解決されたオプションも取得できるので挙動を確認する際には便利です。(NumberFormat は特に暗黙的なデフォルト値が多いので役に立ちます。)

```ts
new Intl.NumberFormat("ja-JP", {
  style: "currency",
  currency: "JPY",
}).resolvedOptions();
// {
//     "locale": "ja-JP",
//     "numberingSystem": "latn",
//     "style": "currency",
//     "currency": "JPY",
//     "currencyDisplay": "symbol",
//     "currencySign": "standard",
//     "minimumIntegerDigits": 1,
//     "minimumFractionDigits": 0,
//     "maximumFractionDigits": 0,
//     "useGrouping": "auto",
//     "notation": "standard",
//     "signDisplay": "auto",
//     "roundingIncrement": 1,
//     "roundingMode": "halfExpand",
//     "roundingPriority": "auto",
//     "trailingZeroDisplay": "auto"
// }
```

## ユースケース

Intl.NumberFormat が特に便利なユースケースについて自分が思いつくものをいくつか紹介します。(もちろんこれ以外にも様々なユースケースがあると思います。)

### 1. 数値の丸め

細かいオプションは次の[14 日目の記事]()で紹介しますが、Intl.NumberFormat は数値の丸めや桁数の調整をするオプションが豊富にあります。四捨五入・切り捨て・切り上げなども選べるので、今まで「10 倍して `Math.round` してまた 10 で割って」のような方法で行っていた数値の丸め処理はほとんどこの Intl.NumberFormat に任せることができます。

```ts
const formatter1 = new Intl.NumberFormat("en", {
  maximumFractionDigits: 2, // 小数部分の最大桁を2桁
  minimumFractionDigits: 2, // 小数部分の最小桁を2桁
});
// 小数部分を必ず2桁にする
formatter1.format(123.456); // → 123.46
formatter1.format(123.4); // → 123.40
```

またこれらのオプションでは整数部分や少数部分の桁数だけでなく、全体の桁数を指定したりと言った細かい調整もできます。

```ts
const formatter2 = new Intl.NumberFormat("en", {
  minimumSignificantDigits: 5, // 全体の最大桁を5桁
  maximumSignificantDigits: 5, // 全体の最小桁を5桁
  useGrouping: false, // ","による桁区切りを無効化
});
// 全体の桁数を必ず5桁にする
formatter2.format(123.4567); // → 123.46
formatter2.format(12345.67); // → 12346
```

このような全体の桁数での調整は表示スペースが限られている部分での表示などにとても便利です。

### 2.具体的な数値から概数を表示したい

数値の表示スペースが限られている場合などに具体的な数値からおよその数を表記したい場合があります。このような場合も Intl.NumberFormat で簡単に対応でき、言語ごとの桁上がりの違いもちゃんと考慮してくれます。(日本は 4 桁ごとに桁上がりしますが、英語は 3 桁ごとに桁上がりしますよね。)

```ts
const enFormatter = new Intl.NumberFormat("en", {
  notation: "compact",
  compactDisplay: "long",
});
const jaFormatter = new Intl.NumberFormat("ja", {
  notation: "compact",
  compactDisplay: "long",
});

enFormatter.format(123456789); // → 123 million
jaFormatter.format(123456789); // → 1.2億
```

### 3. 通貨や単位の表示を便利に

Intl.NumberFormat はオプションにより通貨や単位の表示にも対応しています。例えば料金表を各ロケールに合わせて行いたいというニーズはよくあると思いますが、このような場合も Intl.NumberFormat で簡単に対応できます。

```ts
const enFormatter = new Intl.NumberFormat("en", {
  style: "currency",
  currency: "USD",
});
const jaFormatter = new Intl.NumberFormat("ja", {
  style: "currency",
  currency: "JPY",
});

enFormatter.format(12345); // → $12,345.00
jaFormatter.format(12345); // → ￥12,345
```

また単位の表示などでは複数形の扱いや単位の表記の違いなども考慮してくれるので、これらの出しわけ自ら行うよりも遥かに簡単です。

```ts
const formatter = new Intl.NumberFormat("en", {
  style: "unit",
  unit: "foot",
  unitDisplay: "long",
});

formatter.format(1); // → 1 foot
formatter.format(2); // → 2 feet
```

## まとめと次回予告

この記事では Intl.NumberFormat について基本的な使い方とオプション、ユースケースなどについて解説しました。この記事を見て実際のプロダクトで使うイメージが持てたら嬉しいです。

次回 [14 日目]()は Intl.NumberFormat で指定できる書式化のオプションについて詳しく解説します。
