---
title: "Intl.DateTimeFormat の基本(#7)"
emoji: "🕰️"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: true
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 7 日目の記事です。

今回は Intl.DateTimeFormat について基本的な使い方と各メソッドについて解説します。

## 日時のフォーマット : Intl.DateTimeFormat

Intl.DateTimeFormat はロケールに応じた日付と時刻の書式化をサポートする機能です。

### 基本的な使い方

Intl.DateTimeFormat も他の Intl のコンストラクタプロパティ同様、第１引数にロケール(ロケール識別子 or Intl.Locale オブジェクト)を第２引数にフォーマットのオプションを渡して初期化することで、Intl.DateTimeFormat インスタンスを生成できます。

```ts
const enFormatter = new Intl.DateTimeFormat("en-US", {
  // オプションを指定
});
const jaFormatter = new Intl.DateTimeFormat("ja-JP", {
  // オプションを指定
});
```

書式化の挙動(年月日をどう表示するか、時刻の表示は 24 時制か...など)は全て**インスタンスを生成するタイミングのオプションで指定**します。生成したインスタンスは初期化時のロケール・オプションに従って日時をフォーマットするだけで、ロケールやオプションの上書きもできないので注意が必要です。これらの書式化オプションについては次回[8 日目の記事]()で詳しく解説します。

生成した Intl.DateTimeFormat インスタンスには `format()` のようなメソッドが生えており(詳しくは記述)、このメソッドに Date オブジェクトを渡すことでフォーマットされた文字列を取得できます。

```ts
const date = new Date(2024, 12, 7);
console.log(enFormatter.format(date)); // 1/7/2025
console.log(jaFormatter.format(date)); // 2025/1/7
```

[1 日目の記事の「なぜ一度インスタンスを作るのか」](<https://zenn.dev/sajikix/articles/intl-advent-calendar-24-01#%E3%81%AA%E3%81%9C%E4%B8%80%E5%BA%A6%E3%82%A4%E3%83%B3%E3%82%B9%E3%82%BF%E3%83%B3%E3%82%B9%E3%82%92%E4%BD%9C%E3%82%8B%E3%81%AE%E3%81%8B(%E6%83%B3%E5%83%8F%E3%82%92%E5%90%AB%E3%82%80)>)でも解説しましたが、Intl.DateTimeFormat インスタンスも他の組み込みオブジェクトと比べ生成が比較的重いオブジェクトです。同じロケール・形式で書式化したい場合はなるべく Intl.DateTimeFormat インスタンスやそのメソッドを共有するようにしましょう。

```ts
// formatter.js
export const myDateFormatter = new Intl.DateTimeFormat("ja-JP", {
  year: "numeric",
  month: "long",
  day: "numeric",
}); // インスタンスを生成 & export

// App.tsx
import { myDateFormatter } from "./formatter";
const dateString = myDateFormatter.format(date); // インスタンスを使う
```

## DateTimeFormat インスタンスの持つメソッド

Intl.DateTimeFormat インスタンスには以下の 5 つのメソッドがあります。

- `format()`
- `formatToParts()`
- `formatRange()`
- `formatRangeToParts()`
- `resolvedOptions()`

また Intl.DateTimeFormat 自体への静的メソッドとして `supportedLocalesOf()` があります。それぞれ詳しく見ていきましょう。

### `format()` と `formatToParts()`

`format()` メソッドは Date オブジェクトを渡すことで、初期化時に設定したオプションから書式化した文字列を返すメソッドです。

```ts
new Intl.DateTimeFormat("ja-JP").format(new Date(2024, 11, 7));
// '2024/12/7'
```

ちなみに引数を指定しない場合は「`new Date()` で初期化した Date オブジェクト」を渡したのと同じ挙動になります。

`formatToParts()` も Date オブジェクトを受け取る点は `format()` と同じですが、返り値を書式化された文字列ではなく文字列を構成する部品の配列で返します。配列の各要素は `{type: string, value: string}` というオブジェクトで、`type` には要素の種別、`value` には実際の値が入ります。

```ts
new Intl.DateTimeFormat("ja-JP").formatToParts(new Date(2024, 11, 7));
// [
//   { "type": "year", "value": "2024" },
//   { "type": "literal", "value": "/" },
//   { "type": "month", "value": "12" },
//   { "type": "literal", "value": "/" },
//   { "type": "day", "value": "7" }
// ]
```

この例だと、年月日の書式化された部分がそれぞれ `year`, `month`, `day` という `type` になっており、それ以外の `/` は `literal` という `type` になっています。

`formatToParts()` は日時のフォーマット結果をカスタマイズする際に、`type` に応じて値を取得・操作できとても便利です。また操作した後の文字列は各要素の `value` を連結することで得ることができます。

```ts
// 例: 年の部分を2桁に変更するカスタマイズ
const beforeParts = new Intl.DateTimeFormat("ja-JP").formatToParts(
  new Date(2024, 11, 7)
); // [{type: "year", value: "2024"}, {type: "literal", value: "/"}, ...]
const afterParts = beforeParts.map((part) => {
  if (part.type === "year") {
    return { type: part.type, value: part.value.slice(2) }; // 例: 年の部分を2桁に変更
  }
  return part;
}); // 各 parts に対して操作
const result = afterParts.map((part) => part.value).join(""); // 24/12/7
```

続く[8 日目の記事]()で解説する通り、Int.DateTimeFormat には多くの書式化オプションがあり、基本的なユースケースはそちらでカバーできることが多いですが、どうしても書式化の結果をカスタマイズしたい場合は `formatToParts()` を使ってカスタマイズすると良いでしょう。

### `formatRange()` と `formatRangeToParts()`

`formatRange()` メソッドは年月日ではなく、 `"2024/12/1 - 2024/12/25"` のような日時の範囲を表す文字列を返すメソッドです。そのため、引数には startDate と endDate の 2 つの Date オブジェクトを受け取ります。

```ts
const startDate = new Date(2024, 11, 1);
const endDate = new Date(2024, 11, 25);
new Intl.DateTimeFormat("ja-JP").formatRange(startDate, endDate); // '2024/12/01～2024/12/25'
```

`formatRange()` メソッドは単に書式化された開始と終了の日時を `"-"` や `"～"` でつないだ文字列を返すのではなく、オプションやロケールに応じて省略できる年や月の部分を短縮した文字列を返します。

```ts
// 年と月は範囲内で共通(始まりも終わりも2024/12)なので短縮される例
const startDate = new Date(2024, 11, 1);
const endDate = new Date(2024, 11, 25);
new Intl.DateTimeFormat("en-US", {
  year: "numeric",
  month: "short",
  day: "numeric",
}).formatRange(startDate, endDate); // 'Dec 1 – 25, 2024'
```

このように期間を示す文字列に書式化する際は、単に `format()` で書式化した日時を「~」で繋ぐのではなく　`formatRange()` を活用できると良いでしょう。

また `format()` に `formatToParts()` があったのと同様に、`formatRange()` にも `formatRangeToParts()` があります。`formatRangeToParts()` は `formatRange()` と同じく開始と終了の日時を受け取り、書式化された文字列ではなく文字列を構成する部品の配列を返します。配列の各要素は `{type: string, value: string, source: "startRange" | "endRange" | "shared"}` というオブジェクトで、`type` には要素の種別、`value` には実際の値、`source` にはその要素が開始日時か終了日時かを示す情報が入ります。(type だけだと開始日時のものか終了日時のものかわからなくなるので。)

```ts
const startDate = new Date(2024, 11, 1);
const endDate = new Date(2024, 11, 25);
new Intl.DateTimeFormat("ja-JP").formatRangeToParts(startDate, endDate);
// [
//     {"type": "year", "value": "2024", "source": "startRange"},
//     { "type": "literal", "value": "/", "source": "startRange"},
//     { "type": "month", "value": "12", "source": "startRange"},
//     { "type": "literal", "value": "/", "source": "startRange"},
//     { "type": "day", "value": "01", "source": "startRange"},
//     { "type": "literal", "value": "～", "source": "shared"},
//     { "type": "year", "value": "2024", "source": "endRange"},
//     { "type": "literal", "value": "/", "source": "endRange"},
//     { "type": "month", "value": "12", "source": "endRange"},
//     { "type": "literal", "value": "/", "source": "endRange"},
//     { "type": "day", "value": "25", "source": "endRange"}
// ]
```

こちらも `formatToParts()` と同様に、日時の範囲を示す文字列をカスタマイズする際に便利です。

### `resolvedOptions()`

`resolvedOptions()` メソッドは Intl.DateTimeFormat インスタンスの初期化時に指定したオプションを取得するメソッドです。

```ts
// 和暦のカレンダーとタイムゾーン : 東京を指定して初期化
const jaFormatter = new Intl.DateTimeFormat("ja-JP-u-ca-japanese", {
  timeZone: "Asia/Tokyo",
});
jaFormatter.resolvedOptions();
// {
//     "locale": "ja-JP-u-ca-japanese",
//     "calendar": "japanese",
//     "numberingSystem": "latn",
//     "timeZone": "Asia/Tokyo",
//     "era": "narrow",
//     "year": "numeric",
//     "month": "numeric",
//     "day": "numeric"
// }
```

`resolvedOptions()` メソッドは初期化時に指定したオプションも取得できますが、**暗黙的に解決されたオプションも取得できること**が大きなメリットです。上記の例で言えば `era` や `year`, `month`, `day` などは初期化時に指定していませんが、実際の書式化で適応されるオプションとして取得できています。逆に、`hour`, `minute`, `second` などは含まれていませんから、そもそも書式化した際に表示されないことがわかります。実際に上記のオプションで書式化をすると、和暦の年月日のみが表示され、時分秒は表示されません。

```ts
const jaFormatter = new Intl.DateTimeFormat("ja-JP-u-ca-japanese", {
  timeZone: "Asia/Tokyo",
});
jaFormatter.format(new Date(2024, 11, 7)); // 'R6/12/7'
```

このことから、自分の指定したオプションが正しく適応されているか、またどのようなオプションがデフォルトで適応されているかを確認する際に `resolvedOptions()` メソッドは非常に便利です。

### 静的メソッド : `supportedLocalesOf()`

最後に Intl.DateTimeFormat には静的メソッドとして組み込まれている `supportedLocalesOf()` についても紹介します。このメソッドは指定したロケールのうち日時のフォーマットに対応しているものを選んで返すメソッドです。第 1 引数に、チェックしたいロケール識別子の配列を、第 2 引数にロケールマッチの方法を指定するオプションを渡すことで、対応しているロケールの配列を返します。(ロケールマッチについては[4 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-04)を参考にしてください。)

```ts
const locales1 = ["ban", "id-u-co-pinyin", "de-ID"];
const options1 = { localeMatcher: "lookup" };

Intl.DateTimeFormat.supportedLocalesOf(locales1, options1); // ["id-u-co-pinyin", "de-ID"]
```

実際に利用するケースは少ないかもしれませんが、事前に特定のロケール識別子がサポートされているかを確認する際などには利用できます。

## まとめと次回予告

この記事では Intl.DateTimeFormat について基本的な使い方と、インスタンスの持つメソッド、Intl.DateTimeFormat 自体が持つ静的メソッドなどについて実際のコード例などを交えながら解説しました。次回 [8 日目]()は Intl.DateTimeFormat の初期化オプションについて細かく見ていきます。
