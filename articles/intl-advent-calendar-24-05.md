---
title: "Intlにおける 2つの組み込みメソッド(#5)"
emoji: "🔧"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: true
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 5 日目の記事です。

今回は Intl における 2 つの組み込みメソッドについて解説します。

## Intl における 2 つの組み込みメソッド

Intl には `getCanonicalLocales()` と `supportedValuesOf()` という 2 つの組み込みメソッドが存在します。これらはともに、Intl でサポートする値を返したり、ロケールの正規化をしたりする補助的な役割を持ったメソッドです。

### getCanonicalLocales()

`Intl.getCanonicalLocales()` は、与えられたロケール識別子（と思われる）文字列を正規化する組み込みメソッドです。

引数として文字列または文字列の配列を受け取り、正規化された文字列の配列を返します。

```ts
console.log(Intl.getCanonicalLocales("EN-US")); //  ["en-US"]
console.log(Intl.getCanonicalLocales(["EN-US", "Fr"])); // ["en-US", "fr"]
```

ここでいう正規化はあくまで大文字小文字の正規化だけなので、ロケール識別子の構文から外れた文字列は不正な値として `RangeError` がスローされます。

```ts
try {
  Intl.getCanonicalLocales("hoge-FUGA");
} catch (err) {
  // RangeError: Incorrect locale information provided
}
```

この正規化処理と同じ処理は、Intl のコンストラクタプロパティにロケール識別子を渡した際にも自動的に行われています。そのため、「コンストラクタプロパティに渡す前には `getCanonicalLocales()` で正規化したほうが良い」ということはありません。

一方で、バックエンドとやり取りする際や入力値からステートにロケール識別子を保存する際などは、正規化手段として役に立ちます。特にサーバーなど JavaScript 以外のランタイムにデータを渡す場合、正規化していないロケールで不具合が発生する可能性もあるため、なるべく正規化しておくと安心です。

### supportedValuesOf()

`Intl.supportedValuesOf()` メソッドは、以下のそれぞれの項目について、Intl の実装がサポートしている値の一覧を取得できるメソッドです。

- `calendar` : 暦
- `collation` : 文字列の比較方法
- `currency` : 通貨
- `numberingSystem` : 命数法
- `timeZone` : タイムゾーン
- `unit` : 単位

例: そのランタイムでサポートするカレンダーの一覧を取得する。

```ts
console.log(Intl.supportedValuesOf("calendar"));
// ['buddhist', 'chinese',.... 'persian', 'roc']
```

上記以外の無効なキーを指定した場合は `RangeError` がスローされます。

```ts
try {
  Intl.supportedValuesOf("invalid");
} catch (err) {
  // RangeError: invalid key: "invalid"
}
```

`Intl.supportedValuesOf()` はあくまで、「実行された環境でサポートしている値の一覧」を返すことに注意してください。例えばタイムゾーンの場合、マスターデータにあたるようなデータは IANA が管理しており、時折これらのデータは更新されますが、必ずしもそれらがすぐに反映されるとは限りません。同様に、ブラウザが利用しているロケールに関するデータベース（詳しくは続く [6 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-06)で解説します）も環境によって異なる可能性があります。

一見使い所が少なさそうな `Intl.supportedValuesOf()` ですが、以下のようなユースケースに対応できます。

- ユーザーの環境で特定の機能をサポートするか判別して機能を制限したい時
- サーバーサイドで UI の選択肢を作る際などに情報源として利用する
  - 例: タイムゾーン選択や通貨選択を SSR で作成するときなど

## まとめと次回予告

今回は Intl にある 2 つの組み込みメソッドについて、その概要と注意点、簡単なユースケースなどについて解説しました。次回 [6 日目](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-06)では Intl を支える Unicode の仕様・データ・ライブラリについて解説します。
