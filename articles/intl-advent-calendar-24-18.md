---
title: "Intl.ListFormat(#18)"
emoji: "🧾"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: false
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 18 日目の記事です。

今回はリストの表記をロケールに合わせて書式化してくれるについて解説します。

## `Intl.ListFormat`

### リスト表記とは

リスト表記とは英語で言う `A, B and C` のようにものを列挙する際の表記方法です。このような列挙する表記は言語によって異なり、日本語では `A、B、C` のように、カンマの代わりに読点を使うことが一般的です。英語では `A, B, and C` のように、最後の要素の前に `and / or` を入れることが一般的です。

このようなリスト表記は言語やロケールによって異なる文法規則を持つため、これを自前で判別・書式化するのは大変です。

### Intl.ListFormat の基本的な使い方

Intl.ListFormat は、このようなリストの表記をロケールに合わせて書式化してくれる Intl の API です。

他の Intl のコンストラクタプロパティ同様、第１引数にロケール(ロケール識別子 or Intl.Locale オブジェクト)を第２引数にフォーマットのオプションを渡して初期化することで、Intl.ListFormat インスタンスを生成できます。

```ts
const enFormatter = new Intl.ListFormat("en-US", {
  // オプションを指定
});
const jaFormatter = new Intl.ListFormat("ja-JP", {
  // オプションを指定
});
```

生成した Intl.ListFormat インスタンスは `format()` / `formatToParts()` のようなメソッドが生えており(詳しくは記述)、このメソッドに配列などの反復可能オブジェクトを渡すことで、リスト表記をロケールに合わせて書式化してくれます。

```ts
enFormatter.format(["Train", "Bus", "Car"]); // "Train, Bus, and Car"
```

### メソッド

Intl.ListFormat インスタンスには以下の 3 つのメソッドがあります。

- `format()`
- `formatToParts()`
- `resolvedOptions()`(他と同様なので省略)

#### `format()`

`format()` メソッドは配列などの反復可能オブジェクトを渡すことで、リスト表記をロケールに合わせて書式化した文字列を返すメソッドです。

```ts
new Intl.ListFormat("en-US").format(["Train", "Bus", "Car"]);
// "Train, Bus, and Car"
```

そのため引数を省略したり、反復可能でないオブジェクトを渡すとエラーになります。

```ts
// Not iterable error
new Intl.ListFormat("en-US").format(true);
new Intl.ListFormat("en-US").format(3);
```

#### `formatToParts()`

`formatToParts()` メソッドは `format()` メソッドと異なり、返り値を書式化された文字列ではなく文字列を構成する部品の配列で返します。配列の各要素は `{type: string, value: string}` というオブジェクトで、`type` には要素の種別、`value` には実際の文字列が入ります。`type` には `"element"` と `"literal"` があり、`"element"` はリストの要素、`"literal"` は区切り文字部分を表します。

```ts
new Intl.ListFormat("en-US").formatToParts(["Train", "Bus", "Car"]);
// [
//   { type: "element", value: "Train" },
//   { type: "literal", value: ", " },
//   { type: "element", value: "Bus" },
//   { type: "literal", value: ", and " },
//   { type: "element", value: "Car" },
// ];
```

### オプション

Intl.ListFormat の初期化で指定できるオプションは以下の２つです。(全コンストラクタプロパティ共通の `localeMatcher` を除く)。

- `type` : グループ化の種類
- `style` : リスト表記の見た目

#### `type` オプション

`type` オプションはグループ化する時の種類を指定するオプションです。以下の 3 つの値が指定できます。

- `"conjunction"` : リストの要素を and で繋ぐ(デフォルト)
- `"disjunction"` : リストの要素を or で繋ぐ
- `"unit"` : and や or ではなく区切り文字だけで繋ぐ

```ts
new Intl.ListFormat("en-US").format(["Train", "Bus", "Car"]);
// "Train, Bus, and Car"
new Intl.ListFormat("en-US", { type: "disjunction" }).format([
  "Train",
  "Bus",
  "Car",
]); // "Train, Bus, or Car"
new Intl.ListFormat("en-US", { type: "unit" }).format(["Train", "Bus", "Car"]); // "Train, Bus, Car"
```

日本語などではあまり関係ないですが、英語などでは and か or かを指定できると便利です。

#### `style` オプション

`style` オプションはリスト表記の見た目を指定するオプションです。以下の 3 つの値が指定できます。

- `"long"` : and や or などを省略しない(デフォルト)
- `"short"` : 区切り文字だけで繋ぐ
- `"narrow"` : 空白だけで繋ぐ

```ts
new Intl.ListFormat("en-US").format(["Train", "Bus", "Car"]);
// "Train, Bus, and Car"
new Intl.ListFormat("en-US", { style: "short" }).format([
  "Train",
  "Bus",
  "Car",
]); // "Train, Bus, Car"
new Intl.ListFormat("en-US", { style: "narrow" }).format([
  "Train",
  "Bus",
  "Car",
]); // "Train Bus Car"
```

### 簡単なユースケース

Intl.ListFormat は特に要素数の確定していない要素を書式化する時に便利です。

英語でのリスト書式化を自前で実装する場合以下のように要素の順番に応じた細かい処理が必要になります。

```ts
const formatEnList = (items: string[]) => {
  items
    .map((item, index) => {
      if (index === 0) {
        return item; // 最初の要素はそのまま
      } else if (index === items.length - 1) {
        return ` and ${item}`; // 最後の要素は and をつける
      } else {
        return `, ${item}`; // それ以外はカンマをつける
      }
    })
    .join("");
};
```

これらの処理をサポートする言語ごとに実装するのは大変ですが、Intl.ListFormat を使うことでロケールに合わせたリスト表記を簡単に実現できます。

```ts
const listFormatter = new Intl.ListFormat("en");
const message = `${listFormatter.format([
  "Cash",
  "credit card",
  "various payment",
])} are available for payment`;
// "Cash, credit card and various payment apps are available for payment."
```

このメッセージ例で言えば、今後支払い方法が増減した場合でも、配列に要素を追加するだけで自然なメッセージを生成できます。

## まとめと次回予告

この記事ではリスト表記の書式化を行う `Intl.ListFormat` について基本的な使い方、API について解説しました。

次回[19 日目]()は言語を考慮した文字列の比較を可能にする `Intl.Collator` について解説します。お楽しみに！
