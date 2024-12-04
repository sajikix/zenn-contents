---
title: "Intlにおける 2つの組み込みメソッド(#5)"
emoji: "🔧"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: false
publication_name: "cybozu_frontend"
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 5 日目の記事です。

今回は Intl における 2 つの組み込みメソッドについて解説します。

Intl には `getCanonicalLocales()` と `supportedValuesOf()` という 2 つの組み込みメソッドが存在します。これらはともに Intl でサポートする値を返したりロケールの正規化をしたりする補助的な役割を持ったメソッドたちです。

## getCanonicalLocales()

Intl.getCanonicalLocales() は与えられたロケール識別子(と思われる)文字列を正規化する組み込みメソッドです。

引数として文字列または文字列の配列を受け取り、正規化された文字列の配列を返します。

```ts
console.log(Intl.getCanonicalLocales("EN-US")); //  ["en-US"]
console.log(Intl.getCanonicalLocales(["EN-US", "Fr"])); // ["en-US", "fr"]
```

ここでいう正規化はあくまで大文字小文字の正規化だけなのでロケール識別子の構文から外れた文字列は不正な値として `range error` が throw されます。

この正規化処理と同じ処理は Intl のコンストラクタプロパティにロケール識別子を渡した際にも自動的に行われています。そのため「コンストラクタプロパティに渡す前には `getCanonicalLocales()` で正規化したほうが良い」ということはありません。

一方でバックエンドとやり取りする際や入力値からステートにロケール識別子を保存する際などは正規化手段として役に立ちます。特にサーバーなど JavaScript 以外のランタイムにデータを渡す場合、正規化していないロケールで不具合が発生する可能性もあるため、なるべく正規化しておくと安心です。

## supportedValuesOf()

`Intl.supportedValuesOf()` メソッドは以下のそれぞれの項目について、Intl の実装がサポートしている値の一覧を取得できるメソッドです。

- `calendar` : 暦
- `collation` : 文字列の比較方法
- `currency` : 通過
- `numberingSystem` : 命数法
- `timeZone` : タイムゾーン
- `unit` : 単位

例: そのランタイムでサポートするカレンダーの一覧を取得する。

```ts
console.log(Intl.supportedValuesOf("calendar"));
// ['buddhist', 'chinese',.... 'persian', 'roc']
```

上記以外の無効な key を指定した場合は `RangeError` が throw されます。

```ts
try {
  Intl.supportedValuesOf("invalid");
} catch (err) {
  //Error: RangeError: invalid key: "invalid"
}
```

`Intl.supportedValuesOf()` はあくまで、「実行された環境でサポートしている値の一覧」を返すことに注意してください。例えばタイムゾーンの場合、マスターデータにあたるようなデータは IANA が管理しており時折これらのデータは更新されますが、必ずしもそれらがすぐに反映されるとは限りません。同様にブラウザが利用しているロケールに関するデータベース(詳しくは続く [6 日目の記事]()で解説します)も環境によって異なる能性があります。

一見使い所が少なさそうな `Intl.supportedValuesOf()` ですが、以下のようなユースケースに対応できます。

- ユーザーの環境で特定の機能をサポートするか判別して機能を制限したい時
- サーバーサイドで UI の選択肢を作る際などに情報源として利用する
  - 例 : タイムゾーン選択や通貨選択を SSR で作成するときなど

## まとめと次回予告

今回は Intl にある２つの組み込みメソッドについてその概要と注意点、簡単なユースケースなどについて解説しました。次回 [6 日目]()では Intl を支える Unicode の仕様・データ・ライブラリについて解説します。
