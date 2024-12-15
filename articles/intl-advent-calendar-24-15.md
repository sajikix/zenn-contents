---
title: "NumberFormatの新しいオプションとCurrency Display Choices Proposal(#15)"
emoji: "💲"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: true
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 15 日目の記事です。

今回は Intl.NumberFormat の比較的新しいオプションと、Currency Display Choices Proposal について解説します。

## NumberFormat の新しいオプション

Intl.NumberFormat は最初の ECMA-402 策定時から度々機能がアップデートされています。その中でも ES2023 に正式に仕様として標準化された「Intl.NumberFormat v3」という提案は新しくメソッドや様々なオプションを追加する大きなアップデートでした。この「Intl.NumberFormat v3」もとい、ES2023 から追加された Intl.NumberFormat の仕様群はその後各ランタイムで実装が進み、2024 年 12 月現在全てのブラウザで利用可能になっています。

具体的に、ES2023 での大幅アップデートで以下のような機能・メソッド・オプションが追加されました。

- `formatRange()` / `formatRangeToParts()` メソッドの追加
- `format` 系のメソッドで BigInt や数値文字列を受け付けるように
- 表示桁数や値の丸めに関する細かいオプションの追加
- 桁区切りのグルーピングに関するオプションや、数値の正負の表現に関するオプションの追加

今回はその中でも「表示桁数や値の丸めに関する細かいオプション」と「桁区切りのグルーピングに関するオプションや数値の正負の表現に関するオプション」について詳しく解説します。(`formatRange()` / `formatRangeToParts()` メソッドや `format` 系のメソッドで BigInt を扱えることついては、まとまりの関係上すでに[13 日目](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-13)で解説済みです。)

### 表示桁数や値の丸めに関するオプション

NumberFormat v3 で追加された表示する桁数や値の丸めに関するオプションは以下の 2 つに分類できます。

- 整数部分や小数部分の桁数に関するオプション
- 数値の丸め挙動に関するオプション

#### 整数部分や小数部分の桁数に関するオプション

桁数指定のオプションに関しては２つの「モード」があり、それぞれのオプションがどちらかのモードに属しているというように考えると理解しやすいです。(あくまで著者が勝手に名前をつけているだけで正式に存在する名称ではないので注意してください。)

- 個別桁数指定モード : 整数部分と小数部分の桁数をそれぞれ指定する
  - `minimumIntegerDigits` : 整数部分の最小桁数
  - `minimumFractionDigits` : 小数部分の最小桁数
  - `maximumFractionDigits` : 小数部分の最大桁数
- 全体桁数指定モード : 整数部分と小数部分合わせた桁数を指定する
  - `minimumSignificantDigits` : 全体の最小桁数
  - `maximumSignificantDigits` : 全体の最大桁数

##### 個別桁数指定モード の場合

個別桁数指定モードに属するオプションでは、特に少数部分の桁数を固定したいような場合に便利です。例えば以下のように `minimumFractionDigits` と `maximumFractionDigits` を指定することで、小数部分を必ず 2 桁にできます。

```ts
const formatter = new Intl.NumberFormat("en", {
  maximumFractionDigits: 2, // 小数部分の最大桁を2桁
  minimumFractionDigits: 2, // 小数部分の最小桁を2桁
});
formatter.format(123.456); // → 123.46
formatter.format(123.4); // → 123.40
```

ただし `style` が `"currency"` の場合は通貨の規定によって小数部分の桁数が指定通りにならないことがあるので場合は注意が必要です。この場合、基本的に [ISO 4217](https://www.iso.org/iso-4217-currency-codes.html)の通貨コード一覧における minor unit(副単位 : セントとか銭とか) に従って小数部分の桁数が決まり、何も情報がない場合は 2 桁になります。

##### 全体桁数指定モード の場合

全体桁数指定モードに属するオプションでは、全体の桁数を調整できるので、表示スペースが限られている場合などに便利です。例えば以下のように `minimumSignificantDigits` と `maximumSignificantDigits` を指定することで、全体の桁数を必ず 5 桁にできます。

```ts
const formatter = new Intl.NumberFormat("en", {
  minimumSignificantDigits: 5, // 全体の最大桁を5桁
  maximumSignificantDigits: 5, // 全体の最小桁を5桁
  useGrouping: false, // ","による桁区切りを無効化
});
formatter.format(123.4567); // → 123.46
formatter.format(1234567); // → 12346
```

##### モード の優先順位と `roundingPriority` オプション

ではこの個別桁数指定モードと全体桁数指定モードで矛盾した値をした場合どちらが優先されるのでしょうか？

この桁数指定における「モードの選択」を制御するオプションとして、`roundingPriority` オプションがあります。`roundingPriority` オプションは以下のいずれかを指定でき、デフォルトは `"auto"` です。

- `"auto"`
- `"morePrecision"`
- `"lessPrecision"`

まずは分かりやすい、`"morePrecision"` と `"lessPrecision"` から説明します。`"morePrecision"` が指定された場合、個別桁数指定モードと全体桁数指定モードで**より精度が高くなる方**が選択されます。

```ts
const formatter = new Intl.NumberFormat("en", {
  minimumFractionDigits: 2, // 小数部分の最小桁数
  maximumFractionDigits: 2, // 小数部分の最大桁数
  minimumSignificantDigits: 5, // 全体の最小桁数
  maximumSignificantDigits: 5, // 全体の最大桁数
  roundingPriority: "morePrecision", // より精度が高い方を選択
});
// 全体桁数を優先した方が精度が高い場合 → 全体桁数の方のオプションを選択
formatter.format(1.2345); // → 1.2345
// 小数部分の桁数を優先した方が精度が高い場合 → 少数桁数の方のオプションを選択
formatter.format(1234.5); // → 1234.50
```

逆に `"lessPrecision"` が指定された場合、個別桁数指定モードと全体桁数指定モードで**より精度が低くなる方**が選択されます。

```ts
const formatter = new Intl.NumberFormat("en", {
  minimumFractionDigits: 2, // 小数部分の最小桁数
  maximumFractionDigits: 2, // 小数部分の最大桁数
  minimumSignificantDigits: 5, // 全体の最小桁数
  maximumSignificantDigits: 5, // 全体の最大桁数
  roundingPriority: "lessPrecision", // より精度が低い方を選択
});
// 小数部分の桁数を優先した方が精度が低い場合 → 少数桁数の方のオプションを選択
formatter.format(1.2345); // → 1.23
// 全体桁数を優先した方が精度が低い場合 → 全体桁数の方のオプションを選択
formatter.format(1234.5); // → 1234.5
```

最後にデフォルト値の `"auto"` についてです。`"auto"` の場合は以下のオプションの値(指定されているかどうか含め)の組み合わせによって選択されるモードが変わります。

- `minimumSignificantDigits`
- `maximumSignificantDigits`
- `minimumFractionDigits`
- `maximumFractionDigits`
- `notation`

具体的に以下の 3 パターンに挙動を分けることができます。

1. `minimumSignificantDigits` / `maximumSignificantDigits` が少なくとも１つ指定されている場合
2. １のどちらも指定されておらず、 `minimumFractionDigits` / `maximumFractionDigits` が少なくとも１つ指定されている場合
3. `minimumSignificantDigits` / `maximumSignificantDigits` / `minimumFractionDigits` / `maximumFractionDigits` どれも指定されていない場合

1 の場合、全体桁数指定モードが選択され、2 の場合、個別桁数指定モードが選択されます。つまり、`roundingPriority` オプションが `"auto"` の場合(=何も指定していない時も含む) は全体桁数指定モードが優先されることになります。逆に言えば、`roundingPriority` オプションが `"auto"` で個別桁数指定モードを選択したい場合は全体桁数指定モードのオプションを指定してはいけません。

3 の場合、さらに `notation` オプションによって選択されるモードが変わります。

- `notation` が `"compact"`
  - 各オプションの値が `{ minimumFractionDigits: 0, maximumFractionDigits: 0, minimumSignificantDigits: 1, maximumSignificantDigits: 2 }`
  - その上で `roundingPriority` が強制的に `"morePrecision"` になる(`"auto"` と指定していても)。
- `notation` が `"compact"` 以外 : `minimumFractionDigits` / `maximumFractionDigits` のデフォルト値で個別桁数指定モードになる

非常にややこしいですが、以下のように押さえておけば基本的に問題ないでしょう。

- `roundingPriority` に `"auto"` 以外を指定したらそのオプション通りの挙動
- `roundingPriority` が無指定 / `"auto"` の場合
  - 基本的に全体桁数指定モードが優先される
  - 桁数系のオプションを何も指定せずかつ `notation` が `"compact"` の場合はそれぞれに独自のデフォルト値が適用される

#### 数値の丸め挙動に関するオプション

次に数値の丸めに関するオプションを見ていきましょう。数値の丸めに関するオプションは以下の 3 つがあります。

- `roundingMode` : 丸め方法を指定する
- `roundingIncrement` : 丸めの単位を指定する

##### `roundingMode`

`roundingMode` は数値の丸め方に関するオプションで、全部で９種類存在します。

`"ceil"`/ `"floor"`/ `"expand"`/ `"trunc"` はともに切り捨て or 切り上げをする方法で以下のような違いがあります。

- `"ceil"` : 常に +∞ 側に丸める
- `"floor"` : 常に -∞ 側に丸める
- `"expand"` : 常に 0 から離れる方向に丸める
- `"trunc"` : 常に 0 に近い方向に丸める

`"halfCeil"`/ `"halfFloor"`/ `"halfExpand"`/ `"halfTrunc"`/ `"halfEven"` はより近い方の丸め値に丸める方法です。ちょうど真ん中の値(整数で丸めようとした時の `.5` の状態)の場合にどちらで丸めるかの違いで以下のように分かれています。

- `"halfCeil"` : ちょうど真ん中の値は +∞ 側に丸める
- `"halfFloor"` : ちょうど真ん中の値は -∞ 側に丸める
- `"halfExpand"` : ちょうど真ん中の値は 0 から離れる方向に丸める
- `"halfTrunc"` : ちょうど真ん中の値は 0 に向かう方向で丸める
- `"halfEven"` : ちょうど真ん中の値は偶数になる方向で丸める

ちなみにいわゆる「四捨五入」は　`"halfExpand"` を指定することで実現でき、`roundingMode` のデフォルト値も `"halfExpand"` です。一方、ISO/JIS の規則や浮動小数点の仕様などでも採用されている、コンピュータ分野でよく使われる丸め方法は `"halfEven"` なので一応注意しておきましょう。(基本的に `"halfEven"` の方が「丸め誤差が偏らない」というメリットがあるので多くの処理や仕様で採用されていると考えられますが、書式化ではより馴染みの深い？形を優先したのではないかと筆者は勝手に考えています。)

##### `roundingIncrement`

`roundingIncrement` はどの単位で丸めるかのオプションで、デフォルトは `1` です。使うことは少ないかもしれませんが、クォーターで丸めたい場合なども考慮されており、1, 2, 5, 10, 20, 25, 50, 100, 200, 250, 500, 1000, 2000, 2500, and 5000 のいずれかの値を指定できます。

```ts
const formatter = new Intl.NumberFormat("en", {
  roundingIncrement: 25, // クォーターで丸める
});
formatter.format(123456); // → 123.50
```

### 桁区切りや数値の正負の表現に関するオプション

それぞれ `signDisplay` オプションと `useGrouping` オプションが新たに追加されました。

#### `signDisplay` オプション

`singDisplay` オプションは数値の正負の表現を指定するオプションです。以下の５つの値のいずれかを指定でき、デフォルトは `"auto"` です。

- `"auto"` : 数値が負の場合のみ `-` が付く。(`-0` を含む)
- `"always"` : 正負に関わらず `+/-` が付く
- `"exceptZero"` : 0 以外の数値で `+/-` が付く
- `"negative"` : 数値が負の場合のみ `-` が付くが、`-0` は `0` と表示される
- `"never"` : 正負に関わらず `+/-` が付かない

#### `useGrouping` オプション

また「`1,000,000`」のようにカンマで桁を区切るグループ化の挙動を指定する `useGrouping` オプションも追加されました。`useGrouping` オプションは以下の 5 つの値(文字列 or 真偽値)のいずれかを指定できます。

- `"auto"` : ロケールに応じてグループ化する
- `"always"` : 常にグループ化する(ロケールに関わらず)
- `"min2"` : グループ内に少なくとも 2 桁の数字がある場合にグループ化する
- `true` : `"always"` と同じ
- `false` : グループ化しない(ロケールに関わらず)

### NumberFormat の新しいオプションまとめ

このように新しい Intl.NumberFormat のオプションでは、数値の表示に関するより細かい調整が可能になりました。特に桁数の指定や丸め方に関するオプションは、これまで数値の表示に関する処理を自前で書いていた部分を Intl.NumberFormat に任せることができるようになり、非常に便利です。

## Currency Display Choices Proposal

次に、現在 Intl.NumberFormat に提案されている新しい通貨表示に関するオプションの提案、Currency Display Choices Proposal について解説します。

前回[14 日目](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-14)で紹介した通り、通貨に関するオプションは `style` が `"currency"` の場合にのみ有効で以下の 3 つがありました。

- `currency` : 通貨コード
- `currencyDisplay` : 通貨の表示方法
- `currencySign` : 通貨におけるマイナスの表現方法

Currency Display Choices Proposal はこの `currencyDisplay` オプション(つまり通貨の表記方法)に対して、より細かい表示方法を指定できるようにする提案です。

### 現状の `currencyDisplay` オプション挙動と問題

`currencyDisplay` オプションで `"symbol"` を指定した場合、同じ通貨を指定してもロケールによって表記が変わることがあります。例えば同じ `currency: "USD"` を指定しても、`"en-US"` では `$1.00` と表記されるのに対して、`"en-CA"` では `US$1.00` と表記されます。

```ts
new Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "USD",
  currencyDisplay: "symbol",
}).format(1); // → "$1.00"
new Intl.NumberFormat("en-CA", {
  style: "currency",
  currency: "USD",
  currencyDisplay: "symbol",
}).format(1); // → "US$1.00"
```

これは `"en-CA"` ロケール、つまりカナダでは米ドルとは別に、同じ通貨記号のカナダドル(`CAD`)があるため、`"US$"` のように表示しないとしないと区別がつかなくなってしまうためです。

これに対し、`currencyDisplay` オプションで `"narrowSymbol"` を指定するとこのようなロケールによる考慮を無視して、常に通貨記号だけを表示できます。

```ts
new Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "USD",
  currencyDisplay: "narrowSymbol",
}).format(1); // → "$1.00"
new Intl.NumberFormat("en-CA", {
  style: "currency",
  currency: "USD",
  currencyDisplay: "narrowSymbol",
}).format(1); // → "$1.00" (上と同じ)
```

しかしながら、「常に通貨記号だけを表示」するオプションはあっても「同じ通貨記号の通貨が複数ある場合常にロケール部分も入れる」というオプションはありません。そのため「同じページ(1 つのロケール)で同じ通貨記号を持つ複数の通貨を同時に表記する」といったユースケース(料金比較画面とかでありそうですね)では区別がつきづらくなってしまい困ることになります。

また、通貨記号を表示しないというオプションもないため、「UI の他の部分などで通貨を指定しつつ、数値のフォーマットでは通過記号を表示したくない」といったケースにも対応できません。

### Currency Display Choices Proposal の提案内容

そこで Currency Display Choices Proposal では、`currencyDisplay` オプションに以下の 2 つの値を追加することを提案しています。

- `"formalSymbol"("wideSymbol")` : どのロケールでも正式な広い形の通貨記号を表示する
  - 例 : `USD` → `US$1.00`
- `"never"` : 通貨記号を表示しない
  - 例 : `USD` → `1.00`

```ts
new Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "USD",
  currencyDisplay: "formalSymbol",
}).format(1); // → "US$1.00"
new Intl.NumberFormat("en-CA", {
  style: "currency",
  currency: "USD",
  currencyDisplay: "never",
}).format(1); // → "1.00"
```

これにより、上記のようなユースケースにも対応できるとしています。

## まとめと次回予告

この記事では Intl.NumberFormat の比較的新しいオプション群とその挙動について詳しく解説しました。また現在提案されている Currency Display Choices Proposal についても紹介しました。

次回[16 日目](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-16)の記事では数値フォーマットの中でも単位のフォーマットに関して新しい機能を提供しようとする２つの提案、Smart Unit Preferences Proposal と Representing Measures Proposal について解説します。
