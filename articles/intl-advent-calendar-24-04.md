---
title: "ロケールネゴシエーションとIntl.LocaleMatcher Proposal(#4)"
emoji: ""
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: false
publication_name: "cybozu_frontend"
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 2 日目の記事です。

今回は Intl でロケールネゴシエーションと言われている操作についてと、Intl.LocaleMatcher Proposal について解説します。

## ロケールネゴシエーションとは何か

[1 日目の記事]()で解説した通り、 Intl のコンストラクタプロパティは Intl.Locale を除いて皆以下のような引数もらうのでした。

1. ロケール識別子 or Intl.Locale オブジェクト、あるいはそれらの配列
2. 初期化する際のオプション

第 1 引数に指定可能なロケール識別子と Intl.Locale オブジェクトについてはそれぞれ[2 日目の記事]()と[3 日目の記事]()で解説した通りです。

一方で今までの記事では以下のような場合の細かい挙動については触れてきませんでした。

- `undefined` は許容されるのか
- 不正なロケール識別子がきてしまったらどうなるのか
- 配列だった場合どうなるのか

これらについて「実際に利用されるロケール」はどうやって決定されていくのでしょうか？

### より厳密に、第 1 引数の取る値を考える

そもそも Intl のコンストラクタプロパティの第 1 引数が受け取れる値は以下の３種類です。

1. `undefined`
1. ロケール識別子 or Intl.Locale オブジェクト
1. 2 のリスト

このうち 1 の場合、利用者側の既定のロケールが使用されます。一方 2 の場合、与えられたロケール識別子 or Intl.Locale オブジェクトで指定したロケールが必ずしもランタイム上で利用できるとは限りませんし、単に指定をミスしている可能性もあります。そうなると「ユーザーが指定した値に一番マッチしそうかつ利用可能なロケールを探す」という処理が必ず必要になります。

このように指定されたロケールの情報から、一番マッチしそうかつ利用可能なロケールを探す処理を「ロケールネゴシエーション」と呼びます。

### ロケールネゴシエーション

ロケールネゴシエーションの方法には "lookup" と "best fit" という２種類の方法が用意されています。これらは Intl のコンストラクタプロパティのオプションにある localeMatcher オプションで指定できます。

```ts
new Intl.Segmenter("en-US", { localeMatcher: "lookup" });
```

#### "lookup" の挙動

"lookup" が指定された場合の挙動については ECMA402 における[LookupMatchingLocaleByPrefix](https://tc39.es/ecma402/#sec-lookupmatchinglocalebyprefix)という Abstract Operation で定義されています。また LookupMatchingLocaleByPrefix の挙動は RFC 4647(BCP47 の片方の仕様) の section 3.4 で定義されたアルゴリズムに従っています。

https://www.rfc-editor.org/rfc/rfc4647.html#section-3.4

具体的には利用可能なロケールと一致するまで末尾のサブタグを切り詰めて探していく方式を取るのが "lookup" アルゴリズムです。例えば `zh-Hant-CN-x-private1-private2` というロケール識別子が与えられた場合、以下の順で一致するロケールがあるかを探していきます。

1.  zh-Hant-CN-x-private1-private2
2.  zh-Hant-CN-x-private1
3.  zh-Hant-CN
4.  zh-Hant
5.  zh

ランタイム側で利用可能なロケールリストに `zh`,`zh-Hant`,`zh-Hant-CN` がある場合、3 手目の切り詰めで見つかった `zh-Hant-CN` がマッチした結果となります。

ただし、RFC 4647 に定義されていない Intl 独自の仕様として「ロケール識別子の Unicode 拡張は一度取り除いて "lookup" アルゴリズムで利用可能なアルゴリズムを探し、見つかったロケールに取り除いた Unicode 拡張部分を戻す」という仕様があります。

<!-- 図 -->

#### "best fit" の挙動

"lookup" の挙動が ECMA402 仕様書並びに RFC 4647 で定義されているのに対し、"best fit" は各ランタイムの実装に依存した操作になると ECMA402 仕様書で定義されています。ただしどのようなアルゴリズムでも良いというわけではなく、具体的には以下のように書かれています。

> It determines the best element of availableLocales for satisfying requestedLocales, ignoring Unicode locale extension sequences. The algorithm is implementation dependent, but should produce results that a typical user of the requested locales would consider at least as good as those produced by the LookupMatchingLocaleByPrefix algorithm.

おおまかに意訳すると、「方法は指定しないけど "lookup" と同じかそれ以上に賢い方法を提供してね」ということになるでしょうか。

ちなみに、Safari(JavaScriptCore)と FireFox(SpiderMonkey)は、暫定的に"lookup" のときと同じ挙動をするようです。これらのロケールネゴシエーションについてと実際のランタイムでの挙動についてはは以下の記事が詳しいのでぜひ読んでみてください。

https://sosukesuzuki.dev/posts/intl-locale-matching/

### 無効なロケールが指定された場合と

ではロケールネゴシエーションで最後まで見つからないような、そもそも無効なロケールを指定した場合はどうなるのでしょうか？

Intl で無効なロケールが指定された場合の挙動は、指定したロケールの形によって２パターンあります。

1. 与えられた文字列が明らかにロケール識別子の構文と異なる
2. ロケール識別子の構文は持っているが利用可能なロケールに存在しないもの

1 の場合、すべての Intl のコンストラクタプロパティはエラーを投げます。具体的には ECMA402 で定義された [IsStructurallyValidLanguageTag という Abstract Operation](https://tc39.es/ecma402/#sec-isstructurallyvalidlanguagetag)でチェックされます。１に当てはまるような例として `hoge-FUGA` のようなものがあります。

2 の場合、構造的な不備はないのでエラーが投げられることはありません。しかし実際にマッチするロケールが見つからないので最終的にはシステム既定のロケールが利用されることになります。2 に当てはまるような例として `xx-XX` のようなものがあります。(構文上の間違いはないが実際には `xx` で表される言語タグはない)

### ロケールが複数指定された場合

Intl.Locale 以外の Intl のコンストラクタプロパティは第 1 引数にロケール識別子か Intl.Locale オブジェクトの配列を受け取ることができます。このように複数のロケールが指定された場合の挙動はどうなるのでしょうか？

この場合は「配列の先頭からロケール解決をしていって最初に解決できたものにマッチする」という挙動をします。従って `["en-US","ja"]` のように指定すれば `"en-US"` にマッチします。`["xx-XX","ja"]` のように「ロケール識別子の構文は持っているがロケール解決できないもの」が混じっている場合は配列の次のロケールを解決しようとするので、`"ja"` にマッチします。

注意点として、明らかにロケール識別子の構文と異なるロケール識別子が与えられた場合、配列の位置に関係なくエラーが投げられます。従って、`["en-US","ja","hoge-FUGA"]` のように指定しても `"en-US"` にはマッチしません。

## Intl.LocaleMatcher Proposal

### モチベーション

### 提案されている API

## まとめと次回予告

今回はについて解説しました。次回 5 日目ではについて解説します。

## 参考文献
