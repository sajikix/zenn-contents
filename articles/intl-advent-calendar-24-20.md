---
title: "Intl.Segmenter(#20)"
emoji: "🔪"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: true
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 20 日目の記事です。

今回は単語や見た目の１文字、文単位で文字列を分割できる `Intl.Segmenter` について解説します。

## `Intl.Segmenter`

`Intl.Segmenter` はロケールを考慮しつつ、指定された粒度で文字列を分割するための API です。

### Intl.Segmenter の基本的な使い方

他の Intl のコンストラクタプロパティ同様、第１引数にロケール(ロケール識別子 or Intl.Locale オブジェクト)を第２引数にフォーマットのオプションを渡して初期化することで、Intl.Segmenter インスタンスを生成できます。

```ts
const segmenter = new Intl.Segmenter("ja-JP", {
  // オプションを指定
});
```

生成した Intl.Segmenter インスタンスは `segment()` メソッドが生えており(詳しくは記述)、このメソッドに分割したい文字列を渡すことで分割されたイテレータが得られます。

```ts
const segments = new Intl.Segmenter("ja-JP").segment("こんにちは、世界。");
console.log([...segments]); // ["こんにちは", "、", "世界", "。"]
```

### メソッド

Intl.Segmenter インスタンスには以下の 2 つのメソッドがあります。

- `segment()`
- `resolvedOptions()`(他の Intl オブジェクトと同様なので省略)

#### `segment()` メソッド

`segment()` メソッドは文字列を指定された粒度で分割するためのメソッドです。返り値は分割されたデータを反復可能なコレクションとして保持する Intl.Segments インスタンスです。

```ts
const segments = new Intl.Segmenter("ja-JP", { granularity: "word" }).segment(
  "こんにちは、世界。"
); // Intl.Segments インスタンス
```

Intl.Segments インスタンスは反復可能なデータなので、`for...of` 構文やスプレッド演算子を使って展開できます。

```ts
console.log([...segments]);
// [
//     { "segment": "こんにちは", "index": 0, "input": "こんにちは、世界。", "isWordLike": true},
//     { "segment": "、", "index": 5, "input": "こんにちは、世界。", "isWordLike": false},
//     { "segment": "世界", "index": 6, "input": "こんにちは、世界。", "isWordLike": true},
//     { "segment": "。", "index": 8, "input": "こんにちは、世界。", "isWordLike": false}
// ]
```

各セグメントのデータは以下のプロパティを持つオブジェクトとして表現されます。

- `segment`: 分割された文字列
- `index`: 元の文字列におけるセグメントの開始位置
- `input`: 元の文字列
- `isWordLike`: 分割された文字列がワードかどうかの真偽値
  - (`granularity` オプションが `"word"` の時のみ)

また Intl.Segments インスタンスには `containing()` メソッドが生えており、指定された位置に含まれるセグメントを取得できます。`containing()` メソッドの返り値は上記のセグメントデータと同じオブジェクトです。

```ts
const segments = new Intl.Segmenter("ja-JP", { granularity: "word" }).segment(
  "こんにちは、世界。"
);
console.log(segments.containing(3)); // { "segment": "こんにちは", "index": 0, "input": "こんにちは、世界。", "isWordLike": true}
console.log(segments.containing(7)); // { "segment": "世界", "index": 6, "input": "こんにちは、世界。", "isWordLike": true}
```

`containing()` メソッドは指定された位置にセグメントが存在しない場合は `undefined` を返します。

### Intl.Segmenter のオプション

Intl.Segmenter の初期化で指定できるオプションは全コンストラクタプロパティ共通の `localeMatcher` を除くと、`granularity` の 1 つだけです。

#### `granularity` オプション

`granularity` オプションは分割する粒度を指定するオプションです。指定可能な値は `"grapheme"`, `"word"`, `"sentence"` のいずれかです。

##### `"grapheme"` を指定した時

`"grapheme"` を指定すると Intl.Segmenter は「見た目上の 1 文字(カーソルが 1 つ動く分)」で入力文字列を分割します。このような一文字の単位を書記素(grapheme)と呼びます。

「文字ごとに分割する」と聞くと簡単そうに聞こえますが、実際に「見た目上の 1 文字」で分割するのはかなり難しい問題です。

そもそも JavaScript において文字列は内部的に UTF-16 として表現されますが、UTF16 における基本単位の 2 バイトで見た目上の１文字が表現されるとは限りません。

- サロゲートペアを利用した文字 : 例) 𩸽
- 異体字セレクタを利用した文字 : 例) 肌の色の違う絵文字
- 結合文字と前にある文字の組み合わせ : 例) ̈ + o = ö
- ZWJ を利用した複数文字の結合 : 例) 👨‍👩‍👧‍👦

JS の文字列にある `length` プロパティは 16bit を 1 単位として計算するので、必ずしも「見た目の一文字 = String の　`length`　値」になるとは限りませんし、`length` プロパティを使って文字列を分割して「見た目上の１文字」にならない可能性があります。

```ts
console.log("𩸽".length); // 2
console.log("👨‍👩‍👧‍👦".length); // 11
```

また文字列のイテレータは CodePoint 単位で反復処理するので、fo-of 文や spreadOperator を使えば CodePoint 単位で一意な文字(サロゲートペアで表現されるようなもの)は 1 つの文字として分割できます。それでも異体字セレクタや結合文字、ZWJ などを利用した文字は 1 文字としてカウントできません。

```ts
console.log([..."𩸽"].length); // 2
console.log([..."👨‍👩‍👧‍👦"].length); // 7
```

このような「1 文字をどう扱うか」の問題に関しては以下の記事が詳しいです。

https://blog.jxck.io/entries/2017-03-02/unicode-in-javascript.html

このように「見た目上の 1 文字」で分割することが思ったより難しいことは理解いただけたと思いますが、`Intl.Segmenter` であればこの問題を解決できます。`Intl.Segmenter` で `granularity` オプションに `"grapheme"` が指定された場合、 [Unicode のセグメンテーションアルゴリズム](https://unicode.org/reports/tr29/)を利用し「見た目上の 1 文字」での分割をおこないます。

```ts
const segmenter = new Intl.Segmenter("ja-JP", { granularity: "grapheme" });
console.log([...segmenter.segment("𩸽")].length); // 1
console.log([...segmenter.segment("👨‍👩‍👧‍👦")].length); // 1
```

この処理では、サロゲートペアや異体字セレクタ、結合文字、ZWJ などを利用した文字も「見た目上の 1 文字」として正しく分割されます。

##### `"word"` を指定した時

`"word"` を指定すると Intl.Segmenter は「単語」で入力文字列を分割します。このような単語の単位はロケールによって異なりますが、一般的にはスペースや句読点で区切られた文字列を単語として扱います。

```ts
const enSegmenter = new Intl.Segmenter("en-US", { granularity: "word" });
console.log([...enSegmenter.segment("Hello, world!")]);
// 分割イメージ : ["Hello", ",", " ", "world", "!"]
```

ロケールによっては単語間にスペースが入らない言語もありますが、そのような言語に対してもある程度の精度で単語を分割できます。この実装に関しては別に以下の記事を書いたのでぜひ読んでみてください。

https://zenn.dev/cybozu_frontend/articles/explore-intl-segmenter

##### `"sentence"` を指定した時

`"sentence"` を指定すると Intl.Segmenter は「文」で入力文字列を分割します。このような文の単位はロケールによって異なりますが、一般的にはピリオドや改行文字で区切られた文字列を文として扱います。

```ts
const enSegmenter = new Intl.Segmenter("en-US", { granularity: "sentence" });
console.log([...enSegmenter.segment("Hello, world! How are you?")]);
// 分割イメージ : ["Hello, world!","How are you?"]
```

## まとめと次回予告

この記事では言語を考慮した単語や見た目の１文字、文単位で文字列を分割できる `Intl.Segmenter` について解説しました。

次回[21 日目]() では Intl.Segmenter に新しいオプションを追加する Intl.Segmenter v2 proposal について解説します。

## 参考文献

- JavaScript における文字コードと「文字数」の数え方
  - https://blog.jxck.io/entries/2017-03-02/unicode-in-javascript.html
- Unicode® Standard Annex #29 Unicode Text Segmentation
  - https://unicode.org/reports/tr29/
