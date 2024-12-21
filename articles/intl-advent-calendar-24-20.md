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

### ユースケース

#### 先頭文字を利用する UI・表現

ユーザーやスペースの初期アイコンとして、名前の先頭文字を利用する UI はしばしば見かけます。例えば Google アカウントの初期アイコンなどは名前の先頭文字を利用しています。

![筆者のgoogleの初期アイコン](/images/intlAdventCalendar24_20/googleIcon.png)

このような UI を実装する場合、見た目上の１文字目を「正確に」抜き出す必要があります。前章の `granularity` オプションにも書いたように、見た目上の１文字を正確に抜き出すのは案外むずかく、実際多くのアプリケーションではサロゲートペアを利用した文字や ZWJ を利用した複数文字を結合した文字に対応できないことがあります。しかし `Intl.Segmenter` を利用すればこの問題を解決できます。

```ts
const getFistCharacter = (name: string) => {
  const segmenter = new Intl.Segmenter("ja-JP", { granularity: "grapheme" });
  return [...segmenter.segment(name)][0].segment;
};
```

#### 文字数カウントや適切に

同様に「見た目上の１文字目を正確に分割できる」機能は文字数カウントや適切な truncate にも利用できます。

テキスト入力にはあらかじめ文字数制限のある場合がほとんどですが、この文字数をデータ上の単位とするかユーザーの見た目の文字数とするかはアプリケーションによって異なります。データベースのことなどを考えうるとデータ上の単位(例えば２バイトで１文字分とする)でも良いかも入れませんが、ユーザーは書いた文字数のバイト数など気にしていないので「書いてあるよりも早く文字数制限が来る」ように感じる可能性があります。上限値が数百文字などであればきにならないかもしれませんが、タイトルの入力など上限文字数が少ない場合はこの問題が顕著になります。

```ts
const LIMIT = 10;
const validateLengthLimit = (text: string) => {
  return text.length <= LIMIT ? "OK" : "NG";
};
console.log(validateLengthLimit("あいうえおかきくけこ")); // "OK"
console.log(validateLengthLimit("👨‍👩‍👦‍👦")); // "NG
// 「上限10文字なのに１文字でアウトってどういうこと？」ってなりそうな例
```

このような場合、見た目上の１文字を正確に分割できる `Intl.Segmenter` を利用して文字数をカウントすることで、ユーザーが感じる文字数とデータ上の文字数を一致させることができます。

```ts
const LIMIT = 10;
const validateLengthLimit = (text: string) => {
  const segmenter = new Intl.Segmenter("ja-JP", { granularity: "grapheme" });
  return [...segmenter.segment(text)].length <= LIMIT ? "OK" : "NG";
};
console.log(validateLengthLimit("あいうえおかきくけこ")); // "OK"
console.log(validateLengthLimit("👨‍👩‍👦‍👦")); // "OK"
```

#### 文字列の truncate

スペースの関係などで文字列全体を表示できない場合、`「この記事は...」` のように省略して表示する truncate がよく利用されます。このような truncate 処理はそもそも表示領域に収まるように省略するわけですがから、見た目上の文字数でカウントして省略してほしいところです。意図した表示より短くなる(省略されすぎる)だけならまだ良いですが、見た目上の１文字を正確に分割できないと文字化けや意味不明な文字列が表示される可能性もあります。

```ts
const LIMIT = 10;
const badTruncate = (text: string) => {
  return text.length <= limit ? text : text.slice(0, limit) + "...";
};
console.log(badTruncate("🇯🇵 vs 🏴󠁧󠁢󠁥󠁮󠁧󠁿 この後すぐ！")); // "🇯🇵 vs 🏴..."
// 謎の「🏴」が表示されている
```

このような問題も `Intl.Segmenter` を利用すれば解決できます。

```ts
const LIMIT = 10;
const goodTruncate = (text: string) => {
  const segmenter = new Intl.Segmenter("ja-JP", { granularity: "grapheme" });
  const segments = [...segmenter.segment(text)];
  return segments.length <= LIMIT
    ? text
    : segments
        .slice(0, LIMIT)
        .map((segment) => segment.segment)
        .join("") + "...";
};
console.log(goodTruncate("🇯🇵 vs 🏴󠁧󠁢󠁥󠁮󠁧󠁿 この後すぐ！")); // "🇯🇵 vs 🏴󠁧󠁢󠁥󠁮󠁧󠁿 この後..."
```

#### 文レベルの省略

長い文章を省略して表示する際に、途中の文を省略して表記することがあります。日本語であれば「(中略)」といった表現を使うことが一般的です。

```
この記事では Intl.Segmenter について解説します。(中略)いかがでしたでしょうか？
```

このような文レベルの省略も `Intl.Segmenter` を利用すれば正確に行うことができます。

```ts
const truncateSentence = (text: string) => {
  const segmenter = new Intl.Segmenter("ja-JP", { granularity: "sentence" });
  const segments = [...segmenter.segment(text)];
  return segments[0].segment + "(中略)" + segments[segments.length - 1].segment;
};
```

#### 自力での改行位置改善

テキスト表示における自動改行の位置を改善するためにも `Intl.Segmenter` を利用できます。

特に日本語・中国語・韓国語などの CJK 言語では単語の途中で改行すると読みづらくなります。このような場合、単語の途中で改行せずに適切な位置で改行するために `Intl.Segmenter` の `granularity` オプションに `"word"` を指定して、単語の途中で改行しないような処理を実装できます。

```ts
const LINE_LIMIT = 30;
const breakTokLines = (text: string) => {
  const segmenter = new Intl.Segmenter("ja-JP", { granularity: "word" });
  const segments = [...segmenter.segment(text)];
  return segments.reduce(
    (acc, segment) => {
      if (acc.at(-1).length + segment.segment.length > LINE_LIMIT) {
        acc.push(segment.segment);
      } else {
        acc[acc.length - 1] += segment.segment;
      }
      return acc;
    },
    [""]
  );
};
breakTokLines(
  "メロスは激怒した。必ず、かの邪智暴虐の王を除かなければならぬと決意した。メロスには政治がわからぬ。"
);
// [
//     "メロスは激怒した。必ず、かの邪智暴虐の王を除かなければならぬ",
//     "と決意した。メロスには政治がわからぬ。"
// ]
```

ただし、[上述の記事](https://zenn.dev/cybozu_frontend/articles/explore-intl-segmenter)でも書いた通り Intl.Segmenter における日本語の単語分割は完璧ではなく、より高精度な分ち書き器も公開されているのであくまでも簡易的な実装と考えたほうがいいかもしれません。

分ち書き器の例 : [BudouX: 読みやすい改行のための軽量な分かち書き器](https://developers-jp.googleblog.com/2023/09/budoux-adobe.html)

- github : https://github.com/google/budoux

## まとめと次回予告

この記事では言語を考慮した単語や見た目の１文字、文単位で文字列を分割できる `Intl.Segmenter` について解説しました。

次回[21 日目]() では Intl.Segmenter に新しいオプションを追加する Intl.Segmenter v2 proposal について解説します。

## 参考文献

- JavaScript における文字コードと「文字数」の数え方
  - https://blog.jxck.io/entries/2017-03-02/unicode-in-javascript.html
- Unicode® Standard Annex #29 Unicode Text Segmentation
  - https://unicode.org/reports/tr29/
