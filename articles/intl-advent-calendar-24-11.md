---
title: "Intl.RelativeTimeFormat(#11)"
emoji: "↔️"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: true
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 11 日目の記事です。

今回は Intl.RelativeTimeFormat について解説します。

## Intl.RelativeTimeFormat

Intl.RelativeTimeFormat は「昨日」「3 日後」のような相対的な日時の表記を書式化するための API です。単に「N 日前」「N 日後」といった数値による表記だけでなく、「昨日」「先月」といったロケール固有の相対的な日時文字列も書式化できる機能を持っています。

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

第１引数の数値は正負両方の値を受け取り、`+` の場合は「後」を `-` の場合は「前」を意味します。第２引数は単位のタイプとして `"year"`, `"quarter"`, `"month"`, `"week"`, `"day"`, `"hour"`, `"minute"`, `"second"` のいづれかの値を受け取ります。２つの引数を組み合わせることで「３日後」(=`(3,"day")`)や「1 年前」(=`(-1,"year")`)といった相対日時を表現します。

引数はどちらも省略できないので指定しなかった場合は `RangeError` を throw します。

`formatToParts()` も同じように引数を受け取り、書式化した文字列を構成する部品の配列で返します。これは Intl.DateTimeFormat と同じ挙動ですね。(詳しくは[７日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-7)を参照)

```ts
new Intl.RelativeTimeFormat("ja-JP").formatToParts(-1, "day"); //１日前
// [
//     {"type": "integer","value": "1","unit": "day"},
//     {"type": "literal","value": " 日前"}
// ]
```

配列の各要素は基本的に `{type: string, value: string}` というオブジェクトで、`type` には要素の種別、`value` には実際の値が入ります。ただし `type` が `"integer"` の場合は単位がわかるように `unit` プロパティが追加されます。これは第２引数で指定した値と同じになります。

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

`numeric` は小さい数字でも「1 日前」「2 日後」のように必ず数字で表示するかどうかのオプションです。デフォルトは `"always"` なので必ず数値で表示しますが、`"auto"` を指定すると「昨日」「明日」のようにロケールに応じて自然な表記になります。

```ts
new Intl.RelativeTimeFormat("ja-JP", { numeric: "always" }).format(-1, "day"); // "１日前"
new Intl.RelativeTimeFormat("ja-JP", { numeric: "auto" }).format(-1, "day"); // "昨日"
```

`numberingSystem` は他の Intl に関するオプション同様利用する命数法のオプションです。

```ts
// アラブ数字を利用する例
new Intl.RelativeTimeFormat("en-US", { numberingSystem: "arab" }).format(
  -7,
  "day"
); //'٧ days ago'
```

### ユースケース

ここまでで Intl.RelativeTimeFormat の基本的な使い方とオプションについて解説しましたが、実際にどのようなユースケースで使うのが良いのかを見ていきましょう。

Intl.RelativeTimeFormat が便利なユースケースとして「通知や履歴の UI でどれくらい日時が経っているかを表示する」と言うものがあります。(エンジニアの方であれば github のコミット履歴や issue の更新日時などでもよく見るはずです。)

例えばあるアプリケーションで通知時刻を表示する際に以下のようなルールがあるとします。

- 当日なら時刻を表示
- 前日 ~ 1 週間前までは「n 日前」と表示する
- 1 週間前 ~ １ヶ月以内なら「n 週間前」と表示する
- それ以降は 1 年まで「n ヶ月前」と表示する
- 1 年以上までは「n 年前」と表示する
- 昨日や昨年のように数字でなく自然な表現ができると尚良い

この挙動を愚直に表示しようとするとかなり大変ですが、Intl.RelativeTimeFormat を使うとかなり楽に実装できます。実際に簡単な実装例を見てみましょう。

```ts
const formatNotifiedAt = (date: Date): string => {
  // 経過日数を計算
  const elapsedDay = Math.floor(
    (new Date().getTime() - date.getTime()) / 1000 / 60 / 60 / 24
  );
  // numeric: "auto" でフォーマットするので数字でなく自然な表現に
  const formatter = new Intl.RelativeTimeFormat("ja-JP", { numeric: "auto" });
  // 当日なら
  if (elapsedDay < 1) {
    return new Intl.DateTimeFormat("ja-JP", { timeStyle: "short" }).format(
      date
    );
  }
  // １週間以内なら
  if (elapsedDay < 7) {
    return formatter.format(-Math.floor(elapsedDay), "day"); // 「N日前」の表示
  }
  // １ヶ月以内なら
  if (elapsedDay < 30) {
    return formatter.format(-Math.floor(elapsedDay / 7), "week"); // 「N週間」の表示
  }
  // １年以内なら
  if (elapsedDay < 365) {
    return formatter.format(-Math.floor(elapsedDay / 30), "month"); // 「Nヶ月前」の表示
  }
  return formatter.format(-Math.floor(elapsedDay / 365), "year"); // 「N年前」の表示
};
```

この例では最初に経過日数を計算し、その経過日数に応じて `format()` メソッドの第２引数を変えることで「n 日前」「n 週間前」「n ヶ月前」「n 年前」といった相対的な日時を統一されたフォーマットで作成しています。

実際はもっと細かい部分(１ヶ月 = 30 日でいいのかや閏年を考えなくていいのかなど)を考慮する必要がありますが、このように Intl.RelativeTimeFormat を使うことで相対的な日時の表記を簡単に行えること、何より意外と使い所がありそうなことがわかったと思います。

個人的には Intl.DateTimeFormat より影がうすい印象のある Intl.RelativeTimeFormat もぜひ使いこなして欲しいなと思っています。

## まとめと次回予告

この記事では Intl.RelativeTimeFormat について基本的な使い方とオプション、ユースケースなどについて解説しました。次回 [12 日目](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-12)は期間の書式化をおこなう提案である Intl.DurationFormat Proposal について細かく見ていきます。
