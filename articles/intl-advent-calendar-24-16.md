---
title: "Smart Unit Preferences と Representing Measures Proposal(#16)"
emoji: "📐"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: false
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 16 日目の記事です。

今回は単位のフォーマットに関して新しい機能を提供しようとする２つの提案、Smart Unit Preferences Proposal と Representing Measures Proposal について解説します。

## Smart Unit Preferences Proposal

Smart Unit Preferences Proposal は、Intl.NumberFormat で単位表記をフォーマットする際に、ロケールに合わせた適切な**単位の選択並びに変換まで**行おうという野心的な提案です。

https://github.com/tc39/proposal-smart-unit-preferences

### 単位の変換と使い分け

皆さんもご存知の通り、利用する単位はロケールによって異なります。例えば、日本では長い距離に関して「キロメートル」を使いますが、米国では「マイル」を使います。アプリケーションの国際化をする上では単に文言を翻訳するだけでなく、こういった単位の違いも考慮し適宜変換する必要があります。

しかし現状の Intl でも単位のフォーマットはサポートされていますが、単位の選択や変換などまではサポートされていないので、開発者が自前で実装する必要があります。

```ts
const convertAndFormat = (kilometers: number, locale: string) => {
  if (locale === "ja-JP") {
    return new Intl.NumberFormat(locale, {
      style: "unit",
      unit: "kilometer",
    }).format(kilometers);
  } else {
    return new Intl.NumberFormat(locale, {
      style: "unit",
      unit: "mile",
    }).format(kilometers * 0.621371);
  }
};
```

また同じロケール内であっても、単位の使い分けは必要です。例えば同じ日本でも降水量に関しては「ミリメートル」を使うのが一般的にですが、道のりに関しては「メートル」や「キロメートル」を使います。普段はあまり意識しないかもしれませんがこういった「適切な単位の選択」も開発者やデザイナー・ライターが行ってきたことです。

このような「単位の選択」は自身の詳しくないロケールへ国際化する上でちょっとしたハードルになり得ます。例えば、アメリカ圏では身長を「フィート・インチ」で表すのが一般的(小さい値を除く)ですが、これを知らずに「インチ」だけで表記すると、おそらく多くのアメリカ人に馴染みのない表現になってしまいます。逆にアメリカでは降雨量・積雪は共に「インチ」で表すのが一般的ですが、日本を含むメートル法を使う国では降雨量には「ミリメートル」を使い、1cm 以上の積雪には「センチメートル」を使うのが一般的です。

そして何より、「適切な単位の選択」なしには「適切な値の変換」は実現できません。せっかく変換公式がわかっていて、ロジックが実装できても対象にする適切な単位がわからないことには変換ができません。

つまり、「適切な値の変換」と「適切な単位の選択」とは国際化の文脈上セットであり、「単位の国際化をもっと楽にする」にはこの２つをサポートすることが重要になります。

### Smart Unit Preferences Proposal の API

前置きが長くなりましたが、Smart Unit Preferences Proposal はこの「適切な単位の選択」と「適切な値の変換」をサポートするための Stage1 の提案です。

具体的には Intl.NumberFormat の初期化オプションに新しく `usage` というオプションを追加し、これを使って適切な単位の選択と値の変換をする提案になります。

`usage` というオプションで「何に対する単位なのか」を指定し、利用するロケールと組み合わせることで適切な単位を選択できるようになります。また上の例のように既存の `unit` オプションを使って入力値の単位を表すことができるので、適切な値の変換も行えるようになります。

```ts
const formatter = new Intl.NumberFormat("en-US", {
  style: "unit",
  unit: "meter", // 入力値の単位はメートル
  usage: "person-height", // NEW : 身長として扱う
  unitDisplay: "long",
});
// 変換するための情報が揃う
// - en-USでは身長はフィート・インチで表す
// - 入力はメートル
formatter.format(1.8); // '5 ft 11 in'
```

### Usage に指定できる値

usage に指定できる値としては CLDR で提供されている Unit Preferences の一覧を利用することを検討しているようです。

https://www.unicode.org/cldr/charts/45/supplemental/unit_preferences.html

この表では、基本となる単位ごとに使う対象(Usage)、ロケール、数値の範囲などで適切な単位がわかるようになっています。例えばアメリカでの身長表記を調べたい場合、Base Unit が [`meter`](https://www.unicode.org/cldr/charts/45/supplemental/unit_preferences.html#meter)、Usage が [`person-height`](https://www.unicode.org/cldr/charts/45/supplemental/unit_preferences.html#person-height) 、Sample Region が `US` を含む `"CA, …3"` と絞り込んでいくと、「３インチまではインチで表記し、それ以上はフィート・インチで表記する」ことが書かれています。

## Representing Measures Proposal

Representing Measures Proposal は Smart Unit Preferences Proposal からさらに進んで、単位の書式化だけでなく、単位の変換や計算までをサポートする統合的な単位用の API を提供しようとする、現在 Stage1 の提案です。

https://github.com/ben-allen/proposal-measure

まだ細かい API は提案段階ですが、`Temporal` のように `Measurement` という組み込みクラスを通して単位を扱うことが提案されています。

### 提案されている API

`Measurement` クラスは第 1 引数に数値、第 2 引数に単位に関するオプションを取ります。

```ts
const length = new Measurement(1.8, { unit: "meter" });
```

オプションとしては以下４つを受け取ることが提案されています。

- `unit` : 単位
  - Intl.NumberFormat の `unit` オプション同様 CLDR で管理された単位群に準拠する予定のようです。
- `usage` : 単位の使い方
  - Smart Unit Preferences Proposal の `usage` と同じような使い方をするようです。
- `exponent` : 単位につける指数
  - ㎡ のような単位につく指数を指定できます。
- `precision` : 精度
  - 表示する桁数などを指定できます。

また初期化された `Measurement` インスタンスは書式化や計算、データ変換などのために以下のようなメソッドを持つことが提案されています。

- `toString` : 文字列に変換する
- `toComponents` : 数値と単位を分解する
- `convertTo` : 他の単位に変換する
  - `convertTo(unit, precision)` のように変換先の単位と精度を指定する
  - `formatToParts` のように単位ごとに分解できる
- `exp` : 指数を指定する
  - 例えばメートルから立方メートルに変換するなど
- `add` / `subtract` / `multiply` / `divide` : 単位の四則演算
- `convertToLocale` / `toLocaleString` : ロケールに合わせた書式化
  - `Usage` に基づいた単位の選択も含め書式化する

それぞれ以下のように利用できるようです。

```ts
// データの変換処理
const length = new Measurement(30, { unit: "centimeter" });
footAndInch.toString(); // "5 feet and 6 inches"
footAndInch.toComponents(); // [ {value: 5, unit: "foot"}, {value: 6, unit: "inch"}]
// 単位変換
const lengthInInch = length.convertTo("inch", 1); // 11.8
const footAndInch = new Measurement(5.5, { unit: "foot-and-inch" });
// 四則演算
const length1 = new Measurement(1.8, { unit: "meter" });
const length2 = new Measurement(1, { unit: "meter" });
const sum = length1.add(length2);
const x10 = length1.multiply(10);
// 指数部分の変更 : 立方メートルに変換
const cubic = length1.exp(3);
// ロケールに合わせた書式化
const centimeters = Measurement(170, {
  unit: "centimeter",
  usage: "person-height",
});
centimeters.convertToLocale("en-US"); // {value: 5.6, unit: "foot-and-inch"}
centimeters.toLocaleString("en-US"); // "5 feet and 6 inches"
```

まだまだ使用の検討段階なので API も大きく変わる可能性がありますが、実現されれば単位周りの計算や書式化の手間が大幅に削減できることが期待されます。

## まとめと次回予告

この記事では数値フォーマットの中でも単位のフォーマットに関して新しい機能を提供しようとする２つの提案、Smart Unit Preferences Proposal と Representing Measures Proposal について解説しました。

次回[17 日目]()の記事では言語ごとに異なる複数形の扱いを分類できる `Intl.PluralRules` について解説します。

## 参考文献

- Smart Unit Preferences Proposal : https://github.com/tc39/proposal-smart-unit-preferences
- Representing Measures Proposal : https://github.com/ben-allen/proposal-measure
