---
title: "Intl.DurationFormat Proposal(#12)"
emoji: "⏱️"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: true
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 12 日目の記事です。

今回は Intl.DurationFormat Proposal について解説します。

## Intl.DurationFormat

Intl.DurationFormat Proposal は「１時間３分」のような期間(経過時間)を書式化するために提案されている Proposal です。この Proposal は今年(2024 年)の 12 月の TC39 ミーティングで Stage4 に上がったため、2025 年には正式な Intl の仕様として策定され、ほとんどの環境で使えるようになる予定です。

### 基本的な IF と使い方

他の Intl のコンストラクタプロパティ同様、第１引数にロケール(ロケール識別子 or Intl.Locale オブジェクト)を第２引数にフォーマットのオプションを渡して初期化することで、Intl.DurationFormat インスタンスを生成できます。

```ts
const enFormatter = new Intl.DurationFormat("en-US", {
  // オプションを指定
});
const jaFormatter = new Intl.DurationFormat("ja-JP", {
  // オプションを指定
});
```

生成した Intl.DurationFormat インスタンスには Intl.DateTimeFormat や Intl.RelativeTimeFormat 同様 `format()` / `formatToParts()` のようなメソッドが生えており(詳しくは記述)、このメソッドに `{hours: 1, minutes: 46, seconds: 40}` のようなデータを渡すことで書式化できます。

```ts
const duration = { hours: 1, minutes: 46, seconds: 40 };
new Intl.DurationFormat("ja-JP").format(duration); // 1 時間 46 分 40 秒
```

### オプション

次に Intl.DurationFormat に指定できるオプションを見ていきましょう。

まず、全体の長さを指定するオプションとして `style` オプションがあります。`style` は `"long"`, `"short"`, `"narrow"`,`"digital"` のいずれかの値を指定でき、それぞれのスタイルに応じた長さで書式化されます。デフォルト値は `"short"` です。

```ts
const duration = { hours: 1, minutes: 50 };
new Intl.DurationFormat("en-US", { style: "long" }).format(duration); // 1 hour and 50 minutes
new Intl.DurationFormat("en-US", { style: "short" }).format(duration); // 1 hr, 50 min
new Intl.DurationFormat("en-US", { style: "narrow" }).format(duration); // 1h 50m
new Intl.DurationFormat("en-US", { style: "digital" }).format(duration); // 1:50:00
```

また各単位ごとに表示を制御するオプションも用意されています。具体的にはそれぞれ以下のような値を取ります。

- `years` / `months` / `weeks` / `days`
  - `"long"` / `"short"` / `"narrow"` のいづれかの値を取る
  - 指定しない場合は基本的に `style` オプションで指定したスタイルに従うが、`style` オプションが `"digital"` の場合は `"short"` になる
- `hours` / `minutes` / `seconds`
  - `"long"` / `"short"` / `"narrow"` / `"numeric"` / `"2-digit"` のいづれかの値を取る
  - ただし `hours` が `"numeric"` または `"2-digit"` の場合は他のオプションも　`"numeric"` または `"2-digit"` でなければならない
  - 指定しない場合は基本的に `style` オプションで指定したスタイルに従うが、`style` オプションが `"digital"` の場合は `"numeric"` になる
- `milliseconds` / `microseconds` / `nanoseconds`
  - `"numeric"` / `"2-digit"` のいづれかの値を取る
  - 指定しない場合は `"numeric"` になる

さらに各単位ごとに値が 0 の場合に表示するかどうかを制御するオプションも用意されています。`years` であれば `yearsDisplay`、`hours` であれば `hoursDisplay` といった具合です。これらのオプションは `"auto"` / `"always"` のどちらかの値を取り、`"auto"` の場合は値が 0 の場合は表示されません。

```ts
const duration = { hours: 1, minutes: 50 };
new Intl.DurationFormat("en-US", { secondsDisplay: "auto" }).format(duration); // '1 hr, 50 min'
new Intl.DurationFormat("en-US", { secondsDisplay: "always" }).format(duration); // '1 hr, 50 min, 0 sec'
```

加えて、`style:"digital'` で表示した際、少数になる秒数の部分を最大何桁まで表示するかを制御する `fractionalDigits` オプションも用意されています。 オプションは 0 から 9 までの値を取り、デフォルトでは可能な限り精度を保つようになっています。

```ts
const duration = {
  minutes: 10,
  seconds: 40,
  milliseconds: 500,
  microseconds: 300,
  nanoseconds: 200,
};
new Intl.DurationFormat("en-US", { style: "digital" }).format(duration); // '0:10:40.5003002'
new Intl.DurationFormat("en-US", {
  style: "digital",
  fractionalDigits: 3,
}).format(duration); // '0:10:40.500'
```

また `numberingSystem` は他の Intl に関するオプション同様利用する命数法のオプションとして用意されています。

### メソッド

Intl.DurationFormat インスタンスにも Intl.RelativeTimeFormat 同様以下の 3 つのメソッドがあります。(Intl.RelativeTimeFormat については[11 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-11)で解説しました)

- `format()`
- `formatToParts()`
- `resolvedOptions()`

#### `format()` と `formatToParts()`

`format()` メソッドは引数に Duration を表すオブジェクトを渡すことで初期化時に設定したオプションから書式化した文字列を返すメソッドです。

```ts
new Intl.DurationFormat("en-US").format({ hours: 1, minutes: 50, seconds: 12 });
```

Duration を表すオブジェクトは、`years`, `months`, `weeks`, `days`, `hours`, `minutes`, `seconds`, `milliseconds`, `microseconds`, `nanoseconds` のプロパティを持つオブジェクトで、それぞれのプロパティに数字を指定することで期間を表します。引数は省略できないので指定しなかった場合は `RangeError` を throw します。

:::message
Date オブジェクトは最小単位として milliseconds までしか扱えないのに対し、Intl.DurationFormat は nanoseconds まで扱える点は面白いですね。新しい日付操作の仕様で [9 日目]()の記事でも解説した Temporal は nanoseconds まで扱えるので、そこを見越した設計となっていそうです。Intl.DurationFormat で nanoseconds まで扱うことの苦労については sosukesuzuki さんの記事が面白いのでぜひ読んでみてください。

- [Intl.DurationFormat の最大値を規定する仕様について](https://sosukesuzuki.dev/posts/intl-durationformat-limits/)

:::

`formatToParts()` も同じように引数を受け取り、書式化した文字列を構成する部品の配列で返します。

```ts
new Intl.DurationFormat("en-US").formatToParts({
  hours: 1,
  minutes: 50,
});
// [
//     { "type": "integer", "value": "1", "unit": "hour"},
//     { "type": "literal", "value": " ", "unit": "hour"},
//     { "type": "unit", "value": "hr", "unit": "hour"},
//     { "type": "literal", "value": ", "},
//     { "type": "integer", "value": "50", "unit": "minute"},
//     { "type": "literal", "value": " ", "unit": "minute"},
//     { "type": "unit", "value": "min", "unit": "minute"}
// ]
```

配列の各要素は基本的に `{type: string, value: string}` というオブジェクトで、`type` には要素の種別、`value` には実際の値が入ります。ただし `type` が `"integer"` の場合は単位がわかるように `unit` プロパティが追加されます。これも Intl.RelativeTimeFormat と同じ挙動ですね。

#### `resolvedOptions()`

`resolvedOptions()` も Intl.DateTimeFormat や Intl.RelativeTimeFormat 同様インスタンスの初期化時に指定したオプションを取得してくれます。

```ts
const formatter = new Intl.DurationFormat("en-US", {
  style: "long",
});
formatter.resolvedOptions();
// {
//     "locale": "en-US",
//     "numberingSystem": "latn",
//     "style": "long",
//     "years": "long",
//     "yearsDisplay": "auto",
//     "months": "long",
//     "monthsDisplay": "auto",
//     // ...中略
//     "nanoseconds": "long",
//     "nanosecondsDisplay": "auto"
// }
```

Intl.DateTimeFormat や Intl.RelativeTimeFormat の `resolvedOptions()` メソッドと異なる点として、オプションで指定していない各単位と各単位の表示制御に関するオプションも取得できる点があります。これはどの単位もスタイルと表示制御のデフォルト値を持っているためです。

## Intl の日付フォーマット系機能が出揃った！

Intl.DurationFormat が Stage4 に上がったことで、Intl における日付フォーマット系機能はほぼ出揃ったと言えそうです。今後は、

- 時刻や期間(日時~日時)の書式化 → Intl.DateTimeFormat
- 相対時間の書式化 → Intl.RelativeTimeFormat
- 期間(経過時間)の書式化 → Intl.DurationFormat

という使い分けができるようになります。

さらに、Intl.RelativeTimeFormat と Intl.DurationFormat などオプションの形にある程度の共通性があります。(例えば `style` や `year` のようなオプションが両方に存在します。)

アプリケーション内でこれらの日時系フォーマット機能を統一する際に、オプションも共通して整理しやすくなるのは Intl を使う上でのメリットと言えるでしょう。

## まとめと次回予告

この記事では Intl.DurationFormat について基本的な使い方とオプションなどについて解説しました。この記事で [7 日目](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-7)から続いていた日時系の機能並びに関連仕様の話は一旦終わりになります。

次回 [13 日目](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-13)からは数値の書式化機能である、Intl.NumberFormat について解説します。

## 参考文献

- Intl.DurationFormat Proposal
  - proposal: https://github.com/tc39/proposal-intl-duration-format
  - MDN : https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DurationFormat/DurationFormat
