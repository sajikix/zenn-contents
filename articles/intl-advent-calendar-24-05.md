---
title: " Intlにおける 2つの組み込みメソッド(#5)"
emoji: ""
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: false
publication_name: "cybozu_frontend"
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 3 日目の記事です。

今回は Intl における 2 つの組み込みメソッドについて解説します。

Intl には getCanonicalLocales() と supportedValuesOf() という 2 つの組み込みメソッドが存在します。これらはともに Intl でサポートする値やロケールの正規化なとを返す補助的な役割を持ったメソッドたちです。

## getCanonicalLocales()

Intl.getCanonicalLocales() は与えられたロケール識別子(と思われる)文字列を正規化する組み込みメソッドです。

引数として文字列または文字列の配列を受け取り、正規化された文字列の配列を返します。

```ts
console.log(Intl.getCanonicalLocales("EN-US")); //  ["en-US"]
console.log(Intl.getCanonicalLocales(["EN-US", "Fr"])); // ["en-US", "fr"]
```

ここでいう正規化はあくまで大文字小文字の正規化だけなのでロケール識別子の構文から外れた文字列は不正な値としてエラーが throw されます。

この正規化処理と同じ処理は Intl のコンストラクタプロパティにロケール識別子を渡した際にも自動的に行われているため、「コンストラクタプロパティに渡す際にこの `getCanonicalLocales()` で正規化したほうが良い」ということはありません。

一方でバックエンドとやり取りする際や入力値からステートにロケール識別子を保存する際などは正規化手段として役に立つ可能性があります。

## supportedValuesOf()

Intl.supportedValuesOf() は以下の値について Intl の実装がサポートしている値の一覧を取得できるメソッドです。

- `calendar` : 暦
- `collation` : 文字列の比較方法
- `currency` : 通過
- `numberingSystem` : 命数法
- `timeZone` : タイムゾーン
- `unit` : 単位

```ts
console.log(Intl.supportedValuesOf("calendar"));
// ['buddhist', 'chinese',.... 'persian', 'roc']
```

無効な key の場合はエラーが throw されます。

```ts
try {
  Intl.supportedValuesOf("invalid");
} catch (err) {
  //Error: RangeError: invalid key: "invalid"
}
```

`Intl.supportedValuesOf()` はあくまで、「実行された環境でサポートしている値の一覧」を返すことに注意してください。例えばタイムゾーンの場合、マスターデータにあたるようなデータは IANA が管理しており時折これらのデータは更新されますが、必ずしもそれらがすぐに反映されるとは限りません。同様にブラウザが利用する CLDR(ロケールに関するデータベース・詳しくは続く 6 日目の記事で解説します)も環境によって差異がある可能性があります。

一見使い所が少なさそうですが、`Intl.supportedValuesOf()` を利用することで以下のような以下のようなユースケースに対応できます。

- ユーザーの環境で特定の機能をサポートするか判別して機能を制限したい時
- サーバーサイドで UI の選択肢を作る際などに情報源として利用する
  - 例 : タイムゾーン選択や通貨選択を SSR で作成するときなど

## まとめと次回予告

今回は Intl にある２つの組み込みメソッドについてその概要と注意点ユースケースなどについて解説しました。次回 6 日目では Intl を支える LDML と CLDR について解説します。
