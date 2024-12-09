---
title: "Intl.DateTimeFormatとTemporalの関連(#9)"
emoji: "⏰"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: true
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 9 日目の記事です。

今回は日時の操作に関する ECMAScript Proposal、Temporal と Intl における日時系フォーマット機能の関連について解説します。

## Temporal とは

Temporal は ECMAScript で提案されている新しい日時操作のための標準オブジェクトで、現在 Stage 3 の Proposal です。

https://github.com/tc39/proposal-temporal

Temporal は既存の Date オブジェクトの以下のような不満点を解消、日時操作をより簡単に行えるようにすることを目指しています。

- ユーザーの現地時間と UTC 以外のタイムゾーンがサポートされていない
- 日時文字列のパース動作がわかりづらく不安定
- 日付オブジェクトが何を指しているのかわかりづらい
- サマータイム動作が不安定
- 日時計算の API が扱いにくい
- 非グレゴリオ暦がサポートされていない

具体的に Temporal は以下のようなコンセプトを持って設計されています。

- Wall-Clock Time と Exact Time の分離
- タイムゾーン・カレンダーのサポート
- 便利な計算メソッドの追加

### Wall-Clock Time と Exact Time の分離

Temporal ではタイムゾーンやカレンダーに依存したローカルな時間の情報を持つデータを Wall-Clock Time、UTC 時間の情報しか持たないデータを Exact Time と呼び、これらを厳密に分離して扱います。従って Exact Time Type と Wall-Clock Time Type の相互変換にはタイムゾーンとカレンダーの情報が必要になります。

### タイムゾーン・カレンダーのサポート

現状の Date オブジェクトは UTC とシステムのタイムゾーンしか扱うことができず、カレンダーに関してもグレゴリオ暦しかサポートしていません。Temporal ではこれらの制約を取り除き、任意のタイムゾーンやカレンダーを扱えるようになります。

### 便利な計算メソッドの追加

Temporal では日時の比較や足し引の計算、経過時間の算出など一般的な日時ライブラリと同じような API を提供します。これにより今までの Date オブジェクトよりも直感的に日時操作が行えるようになります。

## Intl.DateTimeFormat との関連

Temporal は Intl.DateTimeFormat 同様日時というドメインを扱うため、両者の仕様は密接に関連しています。実際 Temporal の持つ日時のフォーマットや指定できるオプションなどについては Intl に準拠した仕様策定が進められています。

:::message
この解説は全て 2024 年 12 月時点の情報です。Temporal はまだ Proposal の段階なので、様々な議論によって仕様などが変わる可能性があります。最新の情報は仕様のリポジトリや TC39 のミーティングノートを参照してください。
:::

### Temporal の toLocaleString()メソッド

Temporal の toLocaleString()メソッドは Intl.DateTimeFormat と同じ引数を受け取り、Intl.DateTimeFormat の format()メソッドと同様の文字列を返します。

```ts
const jpDateTime = Temporal.ZonedDateTime.from(
  "2024-12-09T12:00+09:00[Japan/Tokyo]"
);
// Intl.DateTimeFormat と同じ引数を受け取る
jpDateTime.toLocaleString("ja-JP", {
  dateStyle: "full",
  timeStyle: "full",
});
```

Intl.DateTimeFormat での知識(特に各オプションの挙動)をそのまま利用できるので、Intl について詳しくなっておくと知見がそのまま活かせるというメリットがあります。

### Temporal のカレンダーと Duration

Temporal.Duration を `from()` で生成する際は Intl.DurationFormat(詳しくは [12 日目]()に解説)の format メソッドと同じデータを指定することになる予定です。

```ts
// Intl
const formatted = new Intl.DurationFormat("ja-JP").format({
  hours: 1,
  minutes: 23,
  seconds: 45,
});
// Temporal
new Temporal.from({
  hours: 1,
  minutes: 23,
  seconds: 45,
});
```

またそれぞれの値が取りうる最大値なども同じように定義されています。

### サポートするかレンダー

Temporal.Calendar でサポートするカレンダーは少なくとも Intl.DateTimeFormat でサポートするカレンダーを含む必要があります。具体的には Temporal の draft spec における `AvailableCalendars()` という抽象操作の説明として以下のように書かれています。

> (前略) The returned List is sorted according to lexicographic code unit order, and contains unique calendar types in canonical form (12.1) identifying the calendars for which the implementation provides the functionality of Intl.DateTimeFormat objects, including their aliases (e.g., either both or neither of "islamicc" and "islamic-civil").

### Intl 側に対する変更

また逆に一部の Intl の仕様に関しても Temporal が入ることで変更されることが予定されています。

- Intl.DateTimeFormat が Date オブジェクト以外にも Temporal に対応するように
- Temporal はナノ秒まで扱えるため、Intl.DateTimeFormat の定義でもナノ秒まで扱えるように
  - 結果的に内部的にも BigInt で扱うようになっている
- 一部の ECMA402 にあった仕様が ECMA262 側に移行される
  - Temporal 側で参照する都合上だと考えられる
- Temporal の `toLocaleString()` メソッド用に Timezone の上書き挙動が追加される

## まとめ

この記事では新しい日時操作 API Temporal Proposal と Intl、特に Intl.DataTimeFormat との関連を解説しました。Temporal が標準仕様として採用されるまではもう少し時間がかかりそうですが、Intl と相互に影響を与える仕様になる予定なので今のうちに両方の関係を押さえておくと双方の仕様を読みやすくなります。

次回 [10 日目]()は世界の多様な暦に対してより柔軟にサポートするための２つの proposal について解説します。

## 参考文献

- Temporal Proposal
  - spec : https://tc39.es/proposal-temporal
  - github : https://github.com/tc39/proposal-temporal
