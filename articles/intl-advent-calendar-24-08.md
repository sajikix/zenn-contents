---
title: "Intl.DateTimeFormatの主要オプションを抑える(#8)"
emoji: "🕐"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: true
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 8 日目の記事です。

今回は Intl.DateTimeFormat のオプションについてその挙動を解説します。

## Intl.DateTimeFormat のオプション

Intl.DateTimeFormat では、どのように日時を書式化するかについて、初期化する際のオプションで指定できます。

```ts
const formatter = new Intl.DateTimeFormat("ja-JP", {
  calendar: "japanese", // カレンダーを指定するオプション
});
```

従って「どのようなオプションがあるか」、「各オプションを指定した時にどのような挙動をするか」を把握することが Intl.DateTimeFormat を使いこなす上で重要になります。

### 日時文字列を構成する各要素のオプション

Intl.DateTimeFormat では書式化した日時文字列を構成する各要素(=年月日時分秒...など)に対して、それぞれ「どう表示するか」のオプションが用意されています。

- `month`
  - `numeric` : 数値で表示
  - `2-digit` : 2 桁の数値で表示
  - `narrow` : 短縮形で表示
  - `short` : 短い形で表示
  - `long` : 長い形で表示
- `year` / `day` / `hour` / `minute` / `second`
  - `numeric` : 数値で表示
  - `2-digit` : 2 桁の数値で表示
- `fractionalSecondDigits` (秒の小数点以下の桁数)
  - 0 〜 3 の数値で指定
- `weekday`(曜日) / `era`(元号・BC/AD など) / `dayPeriod`(AM/PM 等)
  - `narrow` : 短縮形で表示
  - `short` : 短い形で表示
  - `long` : 長い形で表示
- `timeZoneName`(タイムゾーン名)
  - `short` : 短い形で表示
  - `long` : 長い形で表示
  - `shortOffset` : 短い形でオフセットも表示
  - `longOffset` : 長い形でオフセットも表示
  - `shortGeneric` : 短い形の一般名で表示
  - `longGeneric` : 長い形の一般名で表示

これらのオプションを個別に指定することで、書式化の結果を細かく制御できます。

```ts
const formatter = new Intl.DateTimeFormat("en-US", {
  year: "numeric", // 数値で表示
  month: "long", // 長い形で表示
  day: "2-digit", // 2 桁の数値で表示
  weekday: "long", // 長い形で表示
  era: "short", // 短い形で表示
});
formatter.format(new Date()); // 'Sunday, December 08, 2024 AD'
```

各オプション値における細かい表示例などに関しては MDN を眺めながら実際に試してみると良いでしょう。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat/DateTimeFormat

### 各オプションを省略した時の挙動

日時文字列を構成する各要素に対してスタイルを指定するオプションがあることは分かりましたが、すべてのオプションを指定することは稀です。これらのオプションを省略した場合の挙動はどうなるでしょうか？

:::message
ここで説明する挙動は、「実装がありとあらゆる書式化パターンに対応している」という理想的な状態での挙動です。実際にはブラウザの実装やロケールによってユーザーが指定した通りの書式化パターンが存在しない場合もあります。この時なるべく近い書式化パターンをどう選択するのかに関しては後述の「`formatMatcher` オプションについて」を参照してください。
:::

#### 1 : 全てのオプションを省略した場合

各要素に対するオプションを一切指定しない場合、以下のような初期値をとります。

- `year` / `month` / `day` が `numeric`
- それ以外の要素は表示しない

つまり、年月日は数値で表示され、それ以外の要素は表示されないという挙動になります。

#### 2: `era` / `timeZoneName` 以外のどれかを指定した場合

`year` / `month` / `day` / `hour` / `minute` / `second` / `weekday` / `fractionalSecondDigits` のどれかを指定した場合、指定した要素のみが書式化されて返されます。

```ts
const formatter1 = new Intl.DateTimeFormat("en", { year: "2-digit" });
formatter.format(new Date()); // '24' のように年のみ表示される

const formatter2 = new Intl.DateTimeFormat("ja", {
  second: "numeric",
  year: "2-digit",
});
formatter.format(new Date()); // '24年 10' のように年と秒のみ表示される
```

#### 3: `era` / `timeZoneName` のみ指定した場合

`era` / `timeZoneName` のみ指定した場合、指定した要素 + 年月日が表示されます。この年月日の部分は全て `numeric` で表示されます。(1 の時と同じ)

```ts
const formatter = new Intl.DateTimeFormat("ja", { era: "long" });
formatter.format(new Date()); // '西暦2024/12/8'
```

#### 4: 1~3 以外の場合

これは、`era` / `timeZoneName` と他の要素を併用した場合なので、指定した要素のみが表示されます。

```ts
const formatter = new Intl.DateTimeFormat("ja", {
  era: "long",
  month: "long",
});
formatter.format(new Date()); // '西暦 12月'
```

### まとめてスタイルを指定したい場合

ここまでの例では、各要素に対して個別にスタイルを指定する方法やそれぞれの要素に対するオプションが省略されたときの挙動を見てきました。一方で全ての要素に対して細かくオプションを指定するのは少し面倒です。

このようなユースケースに応えるため、`dateStyle` / `timeStyle` というオプションが用意されています。これはそれぞれ日付部分と時刻部分に対して一括でスタイルを指定するオプションです。

- `dateStyle` / `timeStyle`
  - `full` : 完全な形式で表示
  - `long` : 長い形式で表示
  - `medium` : 中間の形式で表示
  - `short` : 短い形式で表示

このように４種類のスタイルが用意されており、各要素の細かいスタイルは指定できないものの、一括で簡単にスタイルを指定できます。

```ts
const formatter = new Intl.DateTimeFormat("ja", {
  dateStyle: "full",
  timeStyle: "short",
});
formatter.format(new Date()); // '2024年12月8日日曜日 12:00'
```

ただし `dateStyle` / `timeStyle` には**個別要素の書式化オプションと併用できない**という制約があるので注意しましょう。例えば以下のような指定はエラーになります。

```ts
const formatter = new Intl.DateTimeFormat("ja", {
  dateStyle: "full",
  minute: "2-digit",
});
// Uncaught TypeError: Invalid option : option
```

あくまでも `dateStyle` / `timeStyle` は一括でスタイルを簡単に指定するためのもので、各要素をよりカスタマイズしたくなった場合は丁寧に要素ごとに書式を指定してあげましょう。

### そのほかの表記に関連するオプション

ここまでは日時文字列を構成する各要素の書式化オプションに焦点を当てていましたが、他にも日時文字列の表記に関連するオプションは用意されています。

#### `calendar`

カレンダーの種類を指定するオプションです。デフォルトは `gregory` ですが、他にも `buddhist`, `chinese`, `coptic`, `ethioaa`, `hebrew`, `indian`, `islamic`, `japanese`, `persian`, `roc` などが指定できます。

#### `numberingSystem`

命数法を指定するオプションです。デフォルトは `latn` ですが、それ以外にも様々なロケールの命数法が指定できます。

#### `hourCycle`

時刻の表示で「24 時制 / 12 時制どちらか」と「０時と表記するか 12 時(24 時)と表記するか」を指定するオプションです。`"h11"`, `"h12"`, `"h23"`, `"h24"` の 4 パターンが指定できますが、`"h24"` は基本的に採用されていません。

- `"h11"` : 0 ~ 11 で表記する 12 時制
- `"h12"` : 1 ~ 12 で表記する 12 時制
- `"h23"` : 0 ~ 23 で表記する 24 時制
- `"h24"` : 1 ~ 24 で表記する 24 時制

#### `timezone`

使用するタイムゾーンを指定するオプションです。デフォルトは `undefined` でデフォルトのタイムゾーンが使われます。

### 対応する Unicode 拡張

これまで紹介したオプションの一部はロケール識別子における Unicode 拡張で指定できます。(ロケール識別子と Unicode 拡張については[2 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-02)を参考にしてください。)

- `u-nu-[value]` : `numberingSystem` と同じ値を指定できる
- `u-ca-[value]` : `calendar` と同じ値を指定できる
- `u-hc-[value]` : `hourCycle` と同じ値を指定できる

そのため以下の formatter1 と formatter2 は同じオプションを指定していることになります。

```ts
const formatter1 = new Intl.DateTimeFormat("ja-JP-u-ca-japanese");
const formatter2 = new Intl.DateTimeFormat("ja-JP", { calendar: "japanese" });
```

### `formatMatcher` オプションについて

Intl.DateTimeFormat には `formatMatcher` というオプションも用意されています。

Intl.DateTimeFormat では各ロケールにおいて「実装がサポートする書式化パターン」の中から「ユーザーが指定した書式化パターン」に最も近いものを選択するようになっています。この `formatMatcher` オプションは、その書式化パターンの選択方法を指定するもので　`"basic"` か `"best fit"` のどちらかを取ります。

`"basic"` の挙動は ECMA402 仕様書で BasicFormatMatcher という抽象操作(Abstract Operations)として厳密に定義されています。

https://tc39.es/ecma402/#sec-basicformatmatcher

この方式では各オプションと利用可能な書式化パターンとの差分をスコアとして計算し一番スコアの良いものを `bestFormat` として解決します。

一方 `"best fit"` の挙動は「各ランタイムの実装に依存する」と ECMA-402 仕様書で定義されています。

https://tc39.es/ecma402/#sec-bestfitformatmatcher

ただし、[localeMatcher の時](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-04)と同様、`"best fit"` を指定した場合「選択されたロケールの典型的なユーザーが、BasicFormatMatcher によって返されるものと同じかそれ以上に良いと認識するものを返す」ことが求められています。

## Intl.DateTimeFormat を実際に使うときに(私見)

ここからは実際に Intl.DateTimeFormat を使う際に意識できると良さそうな点と気をつけるべき点について私見を紹介します。

### まずは `dateStyle` / `timeStyle` でスタイルを指定してみる

上記の通り、`dateStyle` / `timeStyle` は日付部分と時刻部分に対して一括でスタイルを指定できるため、まずはこれらを使って希望する書式になるかを確認してみると良いでしょう。デザイン上の要求や長さの制約などが厳しくない場合は、この２つのオプションで十分なことが多いです。(実際の書式化例を見ても各ロケールで標準的な出力をしてくれているように思えます。)

### 個別で要素の書式を指定する際は丁寧に

`dateStyle` / `timeStyle`　での指定で満足いく結果が得られない場合は、各要素に対して個別にスタイルを指定することになります。この場合、完全に自分でカスタマイズして組み合わせることになるので、各要素がどう表記されるのか、そもそもその要素が表示されるのかを確認しながら指定していく必要があります。

この際、Intl.DateTimeFormat にある `resolvedOptions()` メソッドを使うと、現在適応される各要素へのオプション(表示されない場合はちゃんと `undefined` になる)を確認できとても便利です。(詳しくは[7 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-07)を参照のこと。)

### ロケールごとにオプションを変えるのもあり

ロケールごとに目指す書式化の形がある場合、ロケールごとにオプションを変えるのも 1 つの手です。これは個人的な感覚ですが、ロケールごとに各要素のオプション値が意味するものは変わることもある(`en-US` と `ja-JP` でほんとに `"long"` の意味合いが同じなのか？等)ので、ロケールごとにオプションを変えて柔軟に対応するのも良いと思います。

もちろん利用可能なロケールごとに Intl.DateTimeFormat インスタンスを作成するロジックは必要になることには注意が必要ですが。

```ts
switch (locale) {
  case "ja-JP":
    return new Intl.DateTimeFormat("ja-JP", { month: "long" });
  case "en-US":
    return new Intl.DateTimeFormat("en-US", { month: "short" });
  default:
    return new Intl.DateTimeFormat(locale);
}
```

### どうしても満足できない場合は `formatToParts()` を使う

個別で要素の書式を指定したり、ロケールごとにオプションを変えても満足できない場合は、`formatToParts()` メソッドを使って各要素を取得・編集して自分で組み合わせるという方法もあります。(これも詳しくは[7 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-07)を参照のこと。)

一方で、書式化の挙動が全く変わらないことを保証するのは「国際化機能」という性質上難しいため、標準とはいえ挙動の代わりうる要素に対してハードコーディングしているという意識は持っておく必要があります。

## まとめ

この記事では Intl.DateTimeFormat の主要なオプションとその挙動について解説しました。また合わせて実際に使う際の私見も紹介しました。次回 [9 日目]()は Intl.DateTimeFormat と日時に関する Proposal :Temporal との関連について解説します。

## 参考文献

- ECMA-402(DateTimeFormat Objects)
  - https://tc39.es/ecma402/#datetimeformat-objects
- MDN Intl.DateTimeFormat
  - https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat/DateTimeFormat
