---
title: "Intl.RelativeTimeFormat(#11)"
emoji: ""
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: false
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 11 日目の記事です。

今回は Intl.RelativeTimeFormat について解説します。

## Intl.RelativeTimeFormat

Intl.RelativeTimeFormat は「昨日」「三日後」のような相対的な日時の表記を書式化するための API です。単に「N 日前」「N 日後」といった数値による表記だけでなく、「昨日」「先月」といったロケール固有の相対的な日時文字列も書式化できる機能を持っています。

### 基本的な IF と使い方

他の Intl のコンストラクタプロパティ同様、第１引数にロケール(ロケール識別子 or Intl.Locale オブジェクト)を第２引数にフォーマットのオプションを渡して初期化することで、Intl.RelativeTimeFormat インスタンスを生成できます。

```ts
const enFormatter = new Intl.RelativeTimeFormat("en-US", {
  // オプションを指定
});
const jaFormatter = new Intl.RelativeTimeFormat("ja-JP", {
  // オプションを指定
});
```

生成した Intl.RelativeTimeFormat インスタンスには Intl.DateTimeFormat 同様 `format()` / `formatToParts()` のようなメソッドが生えており(詳しくは記述)、このメソッドに数値と単位(年/月/日/時/分/秒など)を渡すことで書式化できます。

```ts
jaFormatter.format(-1, "day"); // 昨日
```

### メソッド

Intl.RelativeTimeFormat インスタンスには以下の 3 つのメソッドがあります。

- `format()`
- `formatToParts()`
- `resolvedOptions()`

#### `format()` と `formatToParts()`

`format()` メソッドは第１引数に数値、第２引数に単位を指定することで初期化時に設定したオプションから書式化した文字列を返すメソッドです。

```ts
new Intl.RelativeTimeFormat("ja-JP").format(-1, "day"); //１日前
```

第１引数の数値は正負両方の値を受け取り、+の場合は「後」を-の場合は「前」を意味します。第２引数は単位のタイプとして `"year"`, `"quarter"`, `"month"`, `"week"`, `"day"`, `"hour"`, `"minute"`, `"second"` のいづれかの値を受け取ります。２つの引数を組み合わせることで「３日後」(=`(3,"day")`)や「1 年後」(=`(1,"year")`)といった相対日時を表現します。

引数はどちらも省略できないので指定しなかった場合は `RangeError` を throw します。

`formatToParts()` も同じように引数を受け取り、書式化した文字列を構成する部品の配列で返します。これは Intl.DateTimeFormat と同じ挙動ですね。(詳しくは[７日目の記事]を参照)

```ts
new Intl.RelativeTimeFormat("ja-JP").formatToParts(-1, "day"); //１日前
// [
//     {"type": "integer","value": "1","unit": "day"},
//     {"type": "literal","value": " 日前"}
// ]
```

配列の各要素は基本的に `{type: string, value: string}` というオブジェクトで、`type` には要素の種別、`value` には実際の値が入ります。ただし `type` が `"integer"` の場合は単位がわかるように unit プロパティが追加されます。これは第２引数で指定した値と同じになります。

#### `resolvedOptions()`

`resolvedOptions()` も Intl.DateTimeFormat 同様インスタンスの初期化時に指定したオプションを取得するメソッドです。暗黙的に解決されたオプションも取得できるので挙動を確認する際には便利です。

```ts
// 和暦のカレンダーとタイムゾーン : 東京を指定して初期化
const jaFormatter = new Intl.RelativeTimeFormat("ja-JP", {
  style: "long",
});
jaFormatter.resolvedOptions();
// {
//     "locale": "ja-JP",
//     "style": "long",
//     "numeric": "always",
//     "numberingSystem": "latn"
// }
```

#### 静的メソッド : `supportedLocalesOf()`

### オプション

次に Intl.RelativeTimeFormat に指定するオプションを見ていきましょう。Intl.RelativeTimeFormat に指定できるオプションは全コンストラクタプロパティ共通の `localeMatcher` を除いて以下の３つです。

- `style`
- `numeric`
- `numberingSystem`

`style` は `"long"`,`"short"`,`"narrow"` のいづれかの値をとります。デフォルトは"long"ですが、`"short"` や `"narrow"` を指定することでより短い表記に変更できます。

```ts
new Intl.RelativeTimeFormat("en-US", { style: "long" }).format(-1, "month"); // "1 month ago"
new Intl.RelativeTimeFormat("en-US", { style: "short" }).format(-1, "month"); // "1 mo. ago"
new Intl.RelativeTimeFormat("en-US", { style: "narrow" }).format(-1, "month"); // "1mo ago"
```

`numeric` は小さい数字でも「1 日前」「2 日後」のように必ず数字で表示するかどうかのオプションです。デフォルトは `"always"` なので必ず数値で表示しますが、`"auto"` を指定すると「昨日」「明日」のような自然な表記になります。

```ts
new Intl.RelativeTimeFormat("ja-JP", { numeric: "always" }).format(-1, "day"); // "１日前"
new Intl.RelativeTimeFormat("ja-JP", { numeric: "auto" }).format(-1, "day"); // "昨日"
```

`numberingSystem` は他の Intl に関するオプション同様利用する命数法のオプションです。

```ts
new Intl.RelativeTimeFormat("en-US", { numberingSystem: "arab" }).format(
  -7,
  "day"
); //'٧ days ago'
```

### ユースケース

Intl.RelativeTimeFormat が便利なユースケースとして「通知や履歴の UI でどれくらいのものかを表示する」と言うものがあります。

例えばあるアプリケーションで通知時刻を表示するのに以下のようなルールがあるとします。(そしてこれはよく見るパターンだと思います。)

- 当日なら時刻を表示
- 前日 ~ 1 週間前までは「n 日前」と表示する
- 1 週間前 ~ １ヶ月以内なら「n 週間前」と表示する
- それ以降は 1 年まで「n ヶ月前」と表示する
- 1 年以上までは「n 年前」と表示する

これを愚直に表示しようとするとかなり大変ですが、Intl.RelativeTimeFormat を使うと楽になります。

```ts
const formatNotifiedAt = (date: Date): string => {
  const elapsedDay = Math.floor(
    (new Date().getTime() - date.getTime()) / 1000 / 60 / 60 / 24
  );
  const formatter = new Intl.RelativeTimeFormat("ja-JP", { numeric: "auto" });
  if (elapsedDay < 1) {
    return new Intl.DateTimeFormat("ja-JP", { timeStyle: "short" }).format(
      date
    );
  }
  if (elapsedDay < 30) {
    return formatter.format(-Math.floor(elapsedDay), "day");
  }
  if (elapsedDay < 365) {
    return formatter.format(-Math.floor(elapsedDay), "month");
  }
  return formatter.format(-Math.floor(elapsedDay / 365), "year");
};
```

個人的には Intl.DateTimeFormat より影がうすい印象のある Intl.RelativeTimeFormat もこのように柔軟な相対日時の表記に便利な API なのでぜひ使いこなして欲しいと思います。

## まとめと次回予告

この記事では Intl.RelativeTimeFormat について基本的な使い方とオプション、ユースケースなどについて解説しました。次回 [8 日目]()は期間の書式化をおこなう提案である Intl.DurationFormat Proposal について細かく見ていきます。

## 参考文献
