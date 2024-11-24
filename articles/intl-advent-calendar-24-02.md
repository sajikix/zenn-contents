---
title: "Intl.Localeオブジェクトとオプション(#2)"
emoji: ""
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: false
publication_name: "cybozu_frontend"
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 2 日目の記事です。

今回は Intl.Locale オブジェクトについてその使い所やオプションを解説します。

## Intl.Locale オブジェクトとは

1 つの Intl.Locale オブジェクトは Intl が用意している、「ロケール情報をより簡単に操作するため」のオブジェクトです。

ここで言うロケール情報とは以下のような情報のことです。

- 言語
- 地域
- 文字体系
- カレンダー
- etc..

同じくロケール情報を表せるものとして、`ja-JP` のようなロケール識別子(言語タグ)がありますが、これらと比べ Intl.Locale オブジェクトは以下の点でアドバンテージがあります。

- ローケルに関する各項目を簡単に取得できる
  - ロケール識別子の場合パースが必要ですが Intl.Locale オブジェクトでは言語や地域といった情報をプロパティにアクセスするだけで取得できます。
- ローケルに関する各項目を簡単に変更できる
  - ロケール識別子の場合パース → 該当部分の書き換えという処理が必要になりますが、Intl.Locale オブジェクトではプロパティに値をセットするだけです。

一方 Intl.Locale オブジェクトは JS のオブジェクトなので当然ロケール識別子と比べるとシリアライズが面倒という問題はあります。従ってサーバーなど違うランタイムとのやり取りなどではロケール識別子の方が優れる場合もあります。

### どこで使うか

[1 日目の記事]()で解説した通り、Intl のコンストラクタプロパティは必ず第１引数にロケール識別子か Intl.Locale オブジェクトを受け取ります。

```ts
const jpLocale = new Intl.Locale("ja-JP", { hourCycle: "h12" });
// Intl.LocaleをIntl のコンストラクタプロパティで第１引数として利用する例
const formatter = new Intl.DateTimeFormat(jpLocale, { calendar: "japanese" });
```

したがって一度用途にあったロケール情報を含む Intl.Locale オブジェクトを作成すれば、Intl のコンストラクタプロパティで共通のロケールを利用できるようになります。

## Intl.Locale オブジェクトの作成とオプション

ここからは Intl.Locale オブジェクトを利用する方法について具体的に見ていきます。Intl.Locale オブジェクトは `Intl.Locale()` コンストラクタを初期化することで得られます。

```ts
const jpLocale = new Intl.Locale("ja-JP", { hourCycle: "h12" });
```

第１引数はロケール識別子(言語タグ)を受け取ります。一見他の Intl のコンストラクタプロパティと同様に思えますが、以下の２点で異なるので注意が必要です。

- 文字列のロケール識別子しか受け取らない
- ロケール識別子配列や `undefined` を受け取らない

第２引数はロケール情報を構成または補う情報を指定できるオプションです。具体的には以下の９つのオプションが存在しています。

- `language`
  - 言語を指定するオプション
  - 例 : `ja`,`en` のような言語の種別
  - 基本的にはロケール識別子で指定することが多いがオプションでも指定できる
- `script`
  - 文字体系を指定するオプション
  - 例 : `Hant` : 繁体字と `Hans` : 簡体字など
  - 基本的にはロケール識別子で指定することが多いがオプションでも指定できる
- `region`
  - 地域を指定するオプション
  - 例: `US`/`UK`(`en-US`/`en-UK`)など
  - 基本的にはロケール識別子で指定することが多いがオプションでも指定できる
- `calendar`
  - 使用する暦のオプション
  - 例: `japanese`(和暦)や `hebrew`(ヘブライ暦)など
  - ここの指定すると `Intl.DateTimeFormat` で考慮してくれる
- `collation`
  - a
  - The collation. Any syntactically valid string following the type grammar is accepted, but the implementation only recognizes certain kinds, which are listed in Intl.Locale.prototype.getCollations.
- `numberingSystem`
  - The numbering system. Any syntactically valid string following the type grammar is accepted, but the implementation only recognizes certain kinds, which are listed in Intl.Locale.prototype.getNumberingSystems.
- `caseFirst`
  - The case-first sort option. Possible values are "upper", "lower", or "false".
- `hourCycle`
  - The hour cycle. Possible values are "h23", "h12", "h11", or the practically unused "h24", which are explained in Intl.Locale.prototype.getHourCycles
- `numeric`
  - The numeric sort option. A boolean.

ちなみにロケール識別子とオプションがぶつかった場合はが優先されます

<!-- コード例 -->

### 実際のユースケース
