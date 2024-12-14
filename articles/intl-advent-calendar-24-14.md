---
title: "Intl.NumberFormatの主要オプションを抑える(#14)"
emoji: "🔢"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: true
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 14 日目の記事です。

今回は Intl.NumberFormat の主要オプションについて実例を交えながら詳しく解説します。

## Intl.NumberFormat の主要なオプション

Intl.NumberFormat のオプションには様々なものがあり、これらを組み合わせる事で柔軟なフォーマットを実現できます。他の Intl のコンストラクタと比べてもオプションが多岐にわたるので、ここではなるべくオプションを分類しながら解説していきます。

### NumberFormat の `style` オプションと分類

Intl.NumberFormat の大切なオプションとして、書式化する数値の形を決める `style` オプションがあります。この `style` オプションには以下の 4 種類があり、デフォルトは `"decimal"` です。

- `"decimal"` : 通常の数値
- `"percent"` : パーセント付きの数値
- `"currency"` : 通貨のついた数値
- `"unit"` : 単位のついた数値

```ts
const decimalFormatter = new Intl.NumberFormat("ja-JP", {
  style: "decimal",
});
const percentFormatter = new Intl.NumberFormat("ja-JP", {
  style: "percent",
});
const currencyFormatter = new Intl.NumberFormat("ja-JP", {
  style: "currency",
  currency: "JPY",
});
const unitFormatter = new Intl.NumberFormat("ja-JP", {
  style: "unit",
  unit: "kilometer-per-hour",
});
decimalFormatter.format(12345.6); // '12,345.6'
percentFormatter.format(0.123); // '12%'
currencyFormatter.format(12345.6); // '¥12,346'
unitFormatter.format(12345.6); // '12,345.6 km/h'
```

このうち、`"currency"` と `"unit"` ではそれぞれ、「`style` を `"currency"` / `"unit"` にした時専用のオプション」が存在します。つまり、Intl.NumberFormat には大きく以下の３つのオプションが存在することになります。

- 数値部分に関するオプション
- 通貨に関するオプション
- 単位に関するオプション
- (その他のオプション)

それぞれのオプションについて基本的なものを見ていきましょう。

## 数値部分に関するオプション

数値部分全体に関わるオプションとして `notation` オプションと `compactDisplay` オプションがあります。

### `notation` オプション

まず全体の書式を指定するオプションとして、`notation` オプションがあります。この `notation` は数値全体の表記方法を指定するもので、以下の４種類の値のいずれかを指定できます。

- `"standard"` : 通常の表記方法
  - 例: `12,345.67` (12345.67 の場合)
- `"scientific"` : 整数部分 1 桁の指数表記
  - 例: `1.234567E4` (12345.67 の場合)
- `"engineering"` : 整数部分が最大 3 桁になる指数表記
  - 例: `12.346E3` (12345.67 の場合)
- `"compact"` : 概数での表記
  - 例: `12K` (12345.67 の場合)

`scientific` と `engineering` はともに指数表記ですが、`scientific` は整数部分が 1 桁になるように指数表記されるのに対し、`engineering` は整数部分が最大 3 桁になるように指数表記されます。(したがって `engineering` の場合、E の後の数値は 3 の倍数になります。)

### `compactDisplay` オプション

またこの `notation` で `"compact"` を指定した場合に限り、`compactDisplay` オプションでさらに細かく表示方法を指定できます。`compactDisplay` には `"short"` と `"long"` の２つの値があり、`"short"` の場合は数の単位部分が省略記法で書かれ、`"long"` の場合は数の単位部分が完全な形で書かれます。

```ts
// shortの場合
new Intl.NumberFormat("en", {
  notation: "compact",
  compactDisplay: "short",
}).format(123456789); // → 123M
// longの場合
new Intl.NumberFormat("en", {
  notation: "compact",
  compactDisplay: "long",
}).format(123456789); // → 123 million
```

### 通貨に関するオプション

通貨に関するオプションは上記の通り、`style` が `"currency"` の場合にのみ有効です。通貨に関するオプションは以下の 3 つがあります。

- `currency` : 通貨コード
- `currencyDisplay` : 通貨の表示方法
- `currencySign` : 通貨記号の表示方法

#### `currency` オプション

`currency` オプションは利用する通貨を指定するオプションで、通貨コードを指定します。通貨コードとは [ISO 4217](https://www.iso.org/iso-4217-currency-codes.html) で定義された 3 文字の文字列のことです。通貨コードを指定しない場合はエラーになるので　`style` オプションが `"currency"` の場合は必ず指定する必要があります。

```ts
new Intl.NumberFormat("ja-JP", {
  style: "currency",
  currency: "JPY",
}).format(12345.6); // '¥12,346'
new Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "USD",
}).format(12345.6); // '$12,345.60'
```

#### `currencyDisplay` オプション

`currencyDisplay` オプションは通貨の表示方法を指定するオプションで、以下の 4 つの値のいずれかを指定できます。

- `"symbol"` : 通貨記号を表示する
  - 例: `$`, `¥`, `€`
- `"narrowSymbol"` : 同じ通過記号などの場合情報を足した狭い形にする
  - 例: `US$` (`"$"` ではなく)
- `"name"` : 通貨名を表示する
  - 例: `US dollar`, `Japanese yen` / `"米ドル"`, `"円"`
- `"code"` : 通貨コードを表示する
  - 例: `USD`, `JPY`

#### `currencySign` オプション

海外の(特に英語圏？)会計・簿記の分野では、金額のマイナスを示すのに `()` で囲む方法が採用されます。このような表記に対するのが `currencySign` オプションで、以下の 2 つの値のいずれかを指定できます。

- `"standard"` : 通常のマイナス記号を使う
- `"accounting"` : 金額を `()` で囲む

```ts
new Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "USD",
  currencySign: "standard",
}).format(-12345.6); // '-$12,345.60'
new Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "USD",
  currencySign: "accounting",
}).format(-12345.6); // '($12,345.60)'
```

::: message
日本の会計・簿記の分野では、黒字に ▲(黒三角)を、赤字に △(白三角) を使うことが多いですが、残念ながら `currencySign` オプションではこのような表記はできません。
:::

### 単位に関するオプション

単位に関するオプションも `style` オプションが `"unit"` の場合にのみ有効です。単位に関するオプションは以下の 2 つがあります。

- `unit` : 単位
- `unitDisplay` : 単位の表示方法

#### `unit` オプション

`unit` オプションは利用する単位を指定するオプションで、単位を指定します。単位は [LDML の Unit Elements](https://unicode.org/reports/tr35/tr35-general.html#Unit_Elements) で定義される単位コードを指定します。[CLDR で定義されている単位の種類](https://github.com/unicode-org/cldr/blob/main/common/validity/unit.xml)は非常に多いですが、現状 Intl.NumberFormat でサポートされている単位は、以下の ECMA-402 にある表で定義されているものに限られるので注意してください。(LDML/CLDR については[6 日目](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-06)の記事で解説しました。)

https://github.com/unicode-org/cldr/blob/main/common/validity/unit.xml

また単位を指定しない場合はエラーになるので　`style` オプションが `"unit"` の場合は必ず指定する必要があります。

```ts
new Intl.NumberFormat("ja-JP", {
  style: "unit",
  unit: "milliliter",
}).format(123.45); // '123.45 ml'
```

さらに、上記の単位コードを `"-per-"` でつなげることで、比較的単純な「単位を組み合わせた単位」を表現できます。例えば、`"kilometer-per-hour"` は「km/h」を表します。

```ts
new Intl.NumberFormat("ja-JP", {
  style: "unit",
  unit: "kilometer-per-hour",
}).format(123.45); // '123.45 km/h'
```

ただしこの記法も注意が必要で、`"-per-"` でつなげることができるのはそれぞれ 1 つのサポートされた単位コードのみです。つまり、分母や分子に複数単位を指定する方法は現状ありません。また立方メートル（㎥）のように単位の 2 乗以上を表現する方法も現状ありません。

#### `unitDisplay` オプション

`unitDisplay` オプションは単位の表示方法を指定するオプションで、以下の 3 つの値のいずれかを指定できます。

- `"long"` : 完全な形で表示する
  - 例: `時速 123 キロメートル`
- `"short"` : 省略形で表示する
  - 例: `123 km/h`
- `"narrow"` : 省略形で表示するが、さらに狭い形で表示する
  - 例: `123km/h`

### その他のオプション

他の Intl.DateTimeFormat や Intl.RelativeTimeFormat などと同様に、Intl.NumberFormat にも命数法を指定する `numberingSystem` オプションがあります。またこれも他の機能同様 `numberingSystem` オプションは Unicode 拡張でも指定できます。(`"u-nu-[value]"` の形式。)

通常は `latn` が指定されますが、他の言語や地域に合わせて `arab` (アラブ数字)などを指定できます。

```ts
new Intl.NumberFormat("en-US", {
  numberingSystem: "arab",
}).format(12345.6); // '١٢٬٣٤٥٫٦'
```

## まとめと次回予告

この記事では Intl.NumberFormat の主要なオプションについて詳しく解説しました。一方で今回の記事では以下のようオプションについて触れていません。

- 表示桁数や値の丸めに関するオプション
- 桁区切り記号のグルーピングに関するオプションや、数値の正負の表現に関するオプション

これらは ECMAScript 2023(ES2023) で入った比較的新しいオプション群です。次回[15 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-15)では、これらのオプションについて詳しく解説します。合わせて現在提案されている「通貨表記を拡張する Proposal」についても紹介します。
