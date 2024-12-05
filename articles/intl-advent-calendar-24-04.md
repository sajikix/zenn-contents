---
title: "ロケールネゴシエーションとIntl.LocaleMatcher Proposal(#4)"
emoji: "🤝"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: true
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 4 日目の記事です。

今回は Intl で「ロケールネゴシエーション」と呼ばれる操作についてと、その機能をカスタマイズ可能にする Intl.LocaleMatcher プロポーザルについて解説します。

## ロケールネゴシエーションとは何か

[1 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-01)で解説した通り、Intl のコンストラクタプロパティは Intl.Locale を除いて、以下のような引数を受け取ります。

1. ロケール識別子または Intl.Locale オブジェクト、あるいはそれらの配列
2. 初期化する際のオプション

第 1 引数に指定可能なロケール識別子と Intl.Locale オブジェクトについては、それぞれ[2 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-02)と[3 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-03)で解説しました。

一方で、今までの記事では以下のような場合の細かい挙動について触れてきませんでした。

- `undefined` は許容されるのか
- 不正なロケール識別子がきた場合どうなるのか
- 配列だった場合どうなるのか

これらについて「実際に利用されるロケール」はどのように決定されるのでしょうか？

### より厳密に、第 1 引数の取る値を考える

そもそも Intl のコンストラクタプロパティの第 1 引数が受け取れる値は以下の 3 種類です。

1. `undefined`
2. ロケール識別子または Intl.Locale オブジェクト
3. 2 のリスト

このうち 1 の場合、利用者側の既定のロケールが使用されます。一方 2 の場合、与えられたロケール識別子または Intl.Locale オブジェクトで指定したロケールが必ずしもランタイム上で利用できるとは限りませんし、単に不正な値を指定している可能性もあります。そうなると「ユーザーが指定した値に最もマッチしそうで利用可能なロケールを探す」という処理が必要になります。

このように指定されたロケールの情報から、最もマッチしそうで利用可能なロケールを探す処理を「**ロケールネゴシエーション**」と呼びます。

### ロケールネゴシエーションの指定

ロケールネゴシエーションの方法には "lookup" と "best fit" の 2 種類があります。これらは Intl のコンストラクタプロパティのオプションにある localeMatcher オプションで指定できます。

```ts
new Intl.Segmenter("en-US", { localeMatcher: "lookup" });
```

#### "lookup" の挙動

"lookup" が指定された場合の挙動は、ECMA-402 における[LookupMatchingLocaleByPrefix](https://tc39.es/ecma402/#sec-lookupmatchinglocalebyprefix)という抽象操作(Abstract Operations)[^1]で定義されています。また、LookupMatchingLocaleByPrefix の挙動は RFC 4647（BCP47 の片方の仕様）のセクション 3.4 で定義されたアルゴリズムに従っています。

https://www.rfc-editor.org/rfc/rfc4647.html#section-3.4

具体的には、利用可能なロケールと一致するまで末尾のサブタグを切り詰めて探していく方式を取るのが "lookup" アルゴリズムです。例えば `zh-Hant-CN-x-private1-private2` というロケール識別子が与えられた場合、以下の順で一致するロケールがあるかを探していきます。

1. zh-Hant-CN-x-private1-private2
2. zh-Hant-CN-x-private1
3. zh-Hant-CN
4. zh-Hant
5. zh

ランタイム側で利用可能なロケールリストに `zh`、`zh-Hant`、`zh-Hant-CN` がある場合、3 手目の切り詰めで見つかった `zh-Hant-CN` がマッチした結果となります。

ただし、RFC 4647 に定義されていない Intl 独自の仕様として「ロケール識別子の Unicode 拡張は一度取り除いて "lookup" アルゴリズムで利用可能なロケールを探し、見つかったロケールに取り除いた Unicode 拡張部分を戻す」という仕様があります。

![extendsとoverrideによる依存](/images/intlAdventCalendar24_04/localeMatch.png)

この仕様により、Unicode 拡張で指定したオプションなどがロケールネゴシエーションの過程で削除されることを防ぎます。

#### "best fit" の挙動

"lookup" の挙動が ECMA-402 仕様書並びに RFC 4647 で定義されているのに対し、"best fit" の挙動は各ランタイムの実装に依存すると ECMA-402 仕様書で定義されています。ただし、どのようなアルゴリズムでも良いというわけではなく、具体的には以下のように書かれています。

> It determines the best element of availableLocales for satisfying requestedLocales, ignoring Unicode locale extension sequences. The algorithm is implementation dependent, but should produce results that a typical user of the requested locales would consider at least as good as those produced by the LookupMatchingLocaleByPrefix algorithm.

おおまかに意訳すると、「方法は指定しないけど "lookup" と同じかそれ以上に賢い方法を提供してね」ということになります。

ちなみに、Safari（JavaScriptCore）と Firefox（SpiderMonkey）は、`"best fit"` を指定しても暫定的に `"lookup"` のときと同じ挙動をするようです。これらのロケールネゴシエーションについてと実際のランタイムでの挙動については、以下の記事が詳しいのでぜひ読んでみてください。

https://sosukesuzuki.dev/posts/intl-locale-matching/

### 無効なロケールが指定された場合

では、ロケールネゴシエーションで最後まで見つからないような、そもそも無効なロケールを指定した場合はどうなるのでしょうか？

Intl で無効なロケールが指定された場合の挙動は、指定したロケールの形によって 2 パターンあります。

1. 与えられた文字列が明らかにロケール識別子の構文と異なる
2. ロケール識別子の構文は保っているが利用可能なロケールに存在しないもの

1 の場合、すべての Intl のコンストラクタプロパティは `RangeError` を投げます。具体的には ECMA-402 で定義された [IsStructurallyValidLanguageTag という抽象操作](https://tc39.es/ecma402/#sec-isstructurallyvalidlanguagetag)でチェックされます。1 に当てはまるような例として `hoge-FUGA` のようなものがあります。

```ts
new Intl.Locale("hoge-FUGA"); // ❌
// RangeError: Incorrect locale information provided
```

2 の場合、構造的な不備はないのでエラーが投げられることはありません。しかし、実際にマッチするロケールが見つからないので、最終的にはシステム既定のロケールが利用されることになります。2 に当てはまるような例として `xx-XX` のようなものがあります。（構文上の間違いはないが、実際には `xx` で表される言語タグはない）

### ロケールが複数指定された場合

Intl.Locale 以外の Intl のコンストラクタプロパティは第 1 引数にロケール識別子または Intl.Locale オブジェクトの配列を受け取ることができます。このように複数のロケールが指定された場合の挙動はどうなるのでしょうか？

この場合は「配列の先頭からロケール解決をしていって最初に解決できたものにマッチする」という挙動をします。したがって `["en-US", "ja"]` のように指定すれば `"en-US"` にマッチします。`["xx-XX", "ja"]` のように「ロケール識別子の構文は持っているがロケール解決できないもの」が混じっている場合は、配列の次のロケールを解決しようとするので、`"ja"` にマッチします。

注意点として、明らかにロケール識別子の構文と異なるロケール識別子が与えられた場合、配列の位置に関係なく `RangeError` が投げられます。したがって、例えば `["en-US", "ja", "hoge-FUGA"]` のように指定したとしても `"en-US"` にはマッチしません。

```ts
new Intl.Locale(["en-US", "ja", "hoge-FUGA"]); // ❌
// RangeError: Incorrect locale information provided
```

## Intl.LocaleMatcher プロポーザル

このようにロケールネゴシエーションの挙動には "lookup" と "best fit" の 2 種類がありますが、"best fit" はブラウザの実装依存ですし、何よりユーザーがこのロケールネゴシエーションに介入する方法がありません。このような背景から、ユーザー側で優先するロケールや利用可能なロケールのリストを指定する方法を提供しようとするのが、現在 Stage 1 の Intl.LocaleMatcher プロポーザルです。

https://github.com/tc39/proposal-intl-localematcher

### 考えられているユースケース

ユーザー側で優先するロケールを指定できることで、以下のようなユースケースを満たすことができるとしています。

- 特定言語の翻訳しかないアプリケーションで、なるべくユーザーのロケールに合った言語を選択したり、フォールバック言語を指定したりできる。
- ロケール識別子における x 拡張を利用して独自の挙動を実装できる。
  - 同じ言語でもフォーマルなものと多言語話者にわかりやすい簡易な表現のものを切り替えられるようにするなどができる
- JS エンジン実装側や Polyfill でサポートするロケールが少なくても、効率的で柔軟なロケールネゴシエーションを実装できる。

### 提案されている API

具体的には、以下のようなインターフェースを持つ `Intl.LocaleMatcher.match` メソッドが提案されています。

```
Intl.LocaleMatcher.match(
    requestedLocales: string[],
    availableLocales: string[],
    defaultLocale: string,
    options?: {algorithm: 'lookup' | 'best fit'}
): string
```

引数はそれぞれユーザーが指定したロケールのリスト、利用可能なロケールのリスト、デフォルトのロケール、ロケールネゴシエーションの方法を指定できます。これらの情報から、`Intl.LocaleMatcher.match` メソッドでは利用可能なロケールとユーザーが指定したロケールリストを比べ、一番マッチするロケール（マッチしない場合デフォルトのロケール）を返します。例えば以下のように指定すると `'fr'` が返ります。

```ts
Intl.LocaleMatcher.match(["fr-XX", "en"], ["fr", "en"], "en"); // 'fr'
```

## まとめと次回予告

今回は Intl がロケールを判別する際に行う「ロケールネゴシエーション」についてその挙動を解説しました。また、この「ロケールネゴシエーション」の挙動をよりカスタマイズするための Intl.LocaleMatcher プロポーザルという提案も紹介しました。次回 [5 日目](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-05)では Intl における 2 つの組み込みメソッド、`getCanonicalLocales()` と `supportedValuesOf()` について解説します。

## 参考文献

- Intl.LocaleMatcher Proposal
  - https://github.com/tc39/proposal-intl-localematcher
- RFC 4647: Matching of Language Tags / 3.4. Lookup Algorithm
  - https://www.rfc-editor.org/rfc/rfc4647.html#section-3.4

[^1]: ECMAScript 仕様書内で使われる記法で、同じ仕様上の操作を何度も書かなくていいように切り出された操作のこと。あくまで仕様を書きやすくするために定義されているもので JavaScript コードに関数として公開されないため「抽象」操作と呼ばれる。
