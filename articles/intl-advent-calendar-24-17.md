---
title: "Intl.PluralRules(#17)"
emoji: "✌️"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: true
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 17 日目の記事です。

今回は言語ごとに固有な複数形を処理するための `Intl.PluralRules` について解説します。

## 複数形のある言語と国際化

### 文法上の「数」の扱い

日本語は基本的に文中の名詞の単数・複数で文法が変わらない言語ですが、英語のような多くの言語は名詞の単数・複数によって様々な文法的変化があります。

```
1 sheep is here.
2 sheep are here.
```

単数・複数の２パターンで文法規則が変化する言語が一般的ですが、中には数に応じてより複雑に文法の変わる言語もあります。

- ２つで対になるもの時だけ文法の変わる「双数形」があるもの
- 複数形の中でも「少数」な場合と「多数」な場合で文法が変化するもの
- etc...

また「０」の扱いについても、英語のように複数形利用する言語もあれば、フランス語のように「0」の場合単数形を使う言語もあります。

```
英語 : 1 hour left. / 0 hours left.
フランス語 : 1 heure restante. / 0 heure restante.
```

さらに、数字が大きくても１桁目の数字によって単数・複数形が変わる言語なども存在します。

```
// アイスランド語
1 maður /21 maður（1人/21人の男）
2 menn / 22 menn（2人/22人の男）
```

このように「文法がどう変わるか」という観点で行う数の分類は、言語によって全く異なることがわかります。

### 序数

上記の話は量を表す「基数」についての話でしたが、言語によっては順序を示す「序数」が基数とは異なる体系を持つことがあります。(特にインドヨーロッパ語族に多いようです。)

みなさんご存知の通り英語にも序数(1st,2nd,...)がありますが、その規則はそれなりに複雑です。(恥ずかしながら特に大きな桁だと筆者もパッと言えなくなる時があります。)

1. 下 2 桁が 11, 12, 13 の場合と下 1 桁が 0,4~9 の場合は th
2. １以外の場合は下 1 桁が 1 の場合 st, 2 の場合 nd, 3 の場合 rd

そしてこのような序数のルールも言語によって様々です。また逆に日本語のように単独の序数詞を持たず、「番目」といった接尾辞で補えてしまう言語もあります。

### 複数形の処理

このように基数や序数で文法が変わる言語においては、文言の単数形・複数形の判別処理が重要になります。例えば、以下のような「英語の単数形・複数形のことを全く考えてないコード」を書いてしまうと、特定の値で文法的に正しくない文言が出力されてしまいます。

```ts
const message = `There are ${count} items.`;
// count が 1 の場合 : 「There are 1 items.」になってしまう
```

もちろん英語だけを考えるなら、`count` が 1 かどうかで出す文言を変えれば良いことになります。

```ts
const message = `There ${count === 1 ? "is" : "are"} ${count} item${
  count === 1 ? "" : "s"
}.`;
```

しかしより多くの言語をサポートする場合、上記の通り単純な単数・複数の分類だけでは文言の決定ができない場合もあります。このようなことを考えるとどうしても「あるロケールにおいて数字ごとの文法上の扱いがどう違うのか」を判別する機能が欲しくなります。

## `Intl.PluralRules`

前置きが長くなりましたが、`Intl.PluralRules` はこのような言語ごとの基数と序数のグルーピングを判別するための API です。

### Intl.PluralRules の基本的な使い方

他の Intl のコンストラクタプロパティ同様、第１引数にロケール(ロケール識別子 or Intl.Locale オブジェクト)を第２引数にフォーマットのオプションを渡して初期化することで、Intl.PluralRules インスタンスを生成できます。

```ts
const enRules = new Intl.PluralRules("en-US", {
  // オプションを指定
});
const jaRules = new Intl.PluralRules("ja-JP", {
  // オプションを指定
});
```

生成した Intl.PluralRules インスタンスは `select()` のようなメソッドが生えており(詳しくは記述)、このメソッドに数値を渡すことその数に応じた複数形の規則を示すキーワードを返します。

```ts
enRules.select(1); // 'one'
enRules.select(2); // 'other'
```

### メソッド

Intl.PluralRules インスタンスには以下の 3 つのメソッドがあります。

- `select()`
- `selectRange()`
- `resolvedOptions()`

#### `select()`

`select()` メソッドは引数に数値を受け取り、その数値に対応する複数形のキーワードを返すメソッドです。

```ts
new Intl.PluralRules("ja-JP").select(1); // 'other'
```

返り値は `zero`, `one`, `two`, `few`, `many`, `other` のいずれかです。ユーザーはこのキーワードを使って、文言の出しわけや変更などを行えます。この 6 つの分類に関しては、ECMAScript が準拠し JS エンジンが利用している CLDR の Plural Rules のページが詳しいです。(CLDR については[6 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-06)を参照してください。)

https://cldr.unicode.org/index/cldr-spec/plural-rules#determining-plural-categories

また各ロケールでどのような値が返ってくるか(=基数と序数のルールがどうなっているか)に関しても CLDR の Language Plural Rules を見てみると良いでしょう。

https://www.unicode.org/cldr/charts/46/supplemental/language_plural_rules.html

ちなみに、引数を省略したり、数値以外の値を渡した場合は `"other"` が返ります。

```ts
new Intl.PluralRules("ja-JP").select(); // 'other'
new Intl.PluralRules("ja-JP").select("hoge"); // 'other'
```

#### `selectRange()`

`selectRange()` メソッドは引数に数値の範囲を受け取り、その範囲に対応する複数形のキーワードを返すメソッドです。「範囲に対応する複数形」というと分かりづらいですが、イメージとしては「1~3 個」のような範囲を示す文言に対して「単数系・複数形どちらを適用すべきか」と言ったルールを返してくれます。第１引数に範囲の開始値、第２引数に範囲の終了値を指定します。

```ts
new Intl.PluralRules("en-US").selectRange(1, 3); // 'other'
new Intl.PluralRules("sl").selectRange(102, 201); // 'few'
```

#### `resolvedOptions()`

他の Intl インスタンス同様、初期化時に指定したオプションを取得するメソッドです。暗黙的に解決されたオプションも取得できるので挙動を確認する際には便利です。

```ts
const jaRules = new Intl.PluralRules("ja-JP", {
  type: "ordinal",
});
jaRules.resolvedOptions();
// {
//     "locale": "ja",
//     "type": "ordinal",
//     "minimumIntegerDigits": 1,
//     "minimumFractionDigits": 0,
//     "maximumFractionDigits": 3,
//     "pluralCategories": ["other"],
//     "roundingIncrement": 1,
//     "roundingMode": "halfExpand",
//     "roundingPriority": "auto",
//     "trailingZeroDisplay": "auto"
// }
```

### オプション

Intl.PluralRules に指定できるオプションも見ていきましょう。全コンストラクタプロパティ共通の `localeMatcher` を除くと Intl.PluralRules に指定できるオプションは２種類に分けられます。

- `type` オプション
- 小数点以下の桁数と丸めに関するオプション

#### `type` オプション

`type` オプションは対象とする数値が基数か序数かを指定するオプションです。`'cardinal'`(基数) と `'ordinal'`(序数) のいずれかの値をとります。デフォルトは `cardinal` ですが、`ordinal` を指定することで序数の時の複数形のルールを取得できます。

```ts
new Intl.PluralRules("en-US", { type: "cardinal" }).select(2); // 'other'
new Intl.PluralRules("en-US", { type: "ordinal" }).select(2); // 'two'
```

#### 小数点以下の桁数に関するオプション

Intl.PluralRules の `select()` / `selectRange()` メソッドは小数も受け付けるため、Intl.NumberFormat が受け取るオプションと同じ小数点以下の桁数と丸めに関するオプションを受け取ります。

- `minimumIntegerDigits`
- `minimumFractionDigits`
- `maximumFractionDigits`
- `minimumSignificantDigits`
- `maximumSignificantDigits`
- `roundingPriority`
- `roundingIncrement`
- `roundingMode`

これらは挙動も Intl.NumberFormat 同様ですが、 Intl.NumberFormat において「`{style: "decimal" ,notation: "standard"}` を指定した時」と同じ前提で動くことになります。(つまり `notation` が `"compact"` の時のややこしい挙動は気にしなくて良いです。)

これらの「小数点以下の桁数と丸めに関するオプション」については詳しくの記事で解説しているので、そちらを参照してください。

https://zenn.dev/sajikix/articles/intl-advent-calendar-24-15

### ユースケース

`Intl.PluralRules` のユースケースとしてはやはり、基数や序数に応じた文言の出しわけが一番大きいでしょう。例えば、以下のようなコードを書くことで、基数と序数に応じた文言を出しわけることができます。

```ts
const enRules = new Intl.PluralRules("en-US");
switch (enRules.select(count)) {
  case "one":
    return `There is ${count} item.`;
  case "other":
    return `There are ${count} items.`;
}
```

もちろん序数に応じた文言を出しわけることもできます。

```ts
const enRules = new Intl.PluralRules("en-US", { type: "ordinal" });
switch (enRules.select(count)) {
  case "one":
    return `You are the ${count}st visitor.`;
  case "two":
    return `You are the ${count}nd visitor.`;
  case "few":
    return `You are the ${count}rd visitor.`;
  case "other":
    return `You are the ${count}th visitor.`;
}
```

もちろん、基数と序数による文法パターンは言語によって異なるので厳密に行うならサポートする言語ごとにこれらのロジックを用意する必要があります。それでもパターンの判別を `Intl.PluralRules` に任せることができるのは一定のメリットがあるでしょう。

## まとめと次回予告

この記事では `Intl.PluralRules` について前提となる知識や基本的な使い方、API について解説しました。今ままで複数形や序数の扱いで困っていた方は、ぜひ `Intl.PluralRules` を使ってみてください。

次回[18 日目]()の記事ではリストの表記(`A,B and C` みたいなもの)をロケールに合わせて書式化してくれる　`IntlListFormat`　について解説します。
