---
title: "Intl.DateTimeFormat の基本(#7)"
emoji: "🕰️"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: false
publication_name: "cybozu_frontend"
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 7 日目の記事です。

今回は Intl.DateTimeFormat について基本的な使い方と各メソッドについて解説します。

## 日時のフォーマット : Intl.DateTimeFormat

Intl.DateTimeFormat は ロケールに応じた日付と時刻のフォーマットをサポートする機能です。

### 基本的な使い方

Intl.DateTimeFormat も他の Intl のコンストラクタプロパティ同様、第１引数にロケール(ロケール識別子 or Intl.Locale オブジェクト)を第２引数にフォーマットのオプションを渡して初期化することで、Intl.DateTimeFormat インスタンスを生成できます。

```ts
const enFormatter = new Intl.DateTimeFormat("en-US", {});
const jaFormatter = new Intl.DateTimeFormat("ja-JP", {});
```

生成した Intl.DateTimeFormat インスタンスには format のようなメソッドが生えており(詳しくは次の章で記述)このメソッドに Date オブジェクトを渡すことでフォーマットされた文字列を取得できます。

```ts
const date = new Date(2024, 12, 7);
console.log(enFormatter.format(date)); // 1/7/2025
console.log(jaFormatter.format(date)); // 2025/1/7
```

[1 日目の記事の「なぜ一度インスタンスを作るのか」](<https://zenn.dev/sajikix/articles/intl-advent-calendar-24-01#%E3%81%AA%E3%81%9C%E4%B8%80%E5%BA%A6%E3%82%A4%E3%83%B3%E3%82%B9%E3%82%BF%E3%83%B3%E3%82%B9%E3%82%92%E4%BD%9C%E3%82%8B%E3%81%AE%E3%81%8B(%E6%83%B3%E5%83%8F%E3%82%92%E5%90%AB%E3%82%80)>)でも解説しましたが、Intl.DateTimeFormat インスタンスも他の組み込みオブジェクトと比べ生成が比較的重いオブジェクトです。同じフォーマットで書式化したい場所ではなるべく Intl.DateTimeFormat インスタンスやそのメソッドを共有するようにしましょう。

```ts
// formatter.js
export const myDateFormatter = new Intl.DateTimeFormat("ja-JP", {
  year: "numeric",
  month: "long",
  day: "numeric",
});

// App.tsx
const dateString = myDateFormatter.format(date); // '2024年12月7日'
```

## DateTimeFormat 　インスタンスの持つメソッド

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
new Intl.DateTimeFormat("ja-JP", {
  year: "numeric",
}).format(new Date(2024, 12, 7));
```

ちなみに引数を指定しない場合は `new Date()` で初期化した Date オブジェクトを渡したのと同じ挙動になります。

`formatToParts()` も Date オブジェクトを受け取る点は `format()` と同じですが

### `formatRange()` と `formatRangeToParts()`

### `resolvedOptions()`
