---
title: "Intl.DisplayNames(#22)"
emoji: "📺"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: true
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 22 日目の記事です。

今回はは言語、地域、文字体系などの表示名の一貫した翻訳をサポートする Intl.DisplayNames について解説します。

## Intl.DisplayNames

#### 「言語、地域、文字体系の表示名」　とは？

「言語、地域、文字体系の表示名などの一貫した翻訳」をもう少し具体的に言うと「特定のロケールで言語名・ロケール名・文字体系などをどう表記するか」ということです。

例えば地域名の翻訳を考えてみましょう。地域名は言語によって異なる表記があります。

- 英語で「アメリカ」 → `"United States"`
- 英語で「日本」 → `"Japan"`
- 日本語で「アメリカ」 → `"アメリカ合衆国"`
- 中国語で「アメリカ」 → `"美国"`

このように、言語、地域、文字体系の表示は「翻訳するロケール数」×「表示したい言語・地域・文字体系」の数だけパターン存在するため、全てを自前でサポートするのは到底不可能です。

### Intl.DisplayNames の基本的な使い方

他の Intl のコンストラクタプロパティ同様、第１引数にロケール(ロケール識別子 or Intl.Locale オブジェクト)を第２引数にフォーマットのオプションを渡して初期化することで、Intl.DisplayNames インスタンスを生成できます。

```ts
const regionNamesInEnglish = new Intl.DisplayNames(["en"], {
  type: "region",
  // オプションを指定
});
```

生成した Intl.DisplayNames インスタンスは `of()` メソッドが生えており(詳しくは記述)、このメソッドに言語コード・地域コードなどを渡すことで、そのロケールでの表示名を取得できます。

```ts
regionNamesInEnglish.of("US"); // "United States"
```

### Intl.DisplayNames のメソッド

Intl.DisplayNames インスタンスには以下の 2 つのメソッドがあります。

- `of()`
- `resolvedOptions()`(他の Intl オブジェクトと同様なので省略)

#### `of()` メソッド

`of()` メソッドは言語コード・地域コードなどを渡すことで、そのロケールでの表示名を取得するためのメソッドです。

```ts
new Intl.DisplayNames(["en"], { type: "region" }).of("US");
// "United States"
```

渡したコードがフォーマットとして不正なものであったり、そもそも文字列でなかった場合は RangeError がスローされます。

```ts
// RangeError: invalid_argument
new Intl.DisplayNames(["en"], { type: "region" }).of("xxxx");
new Intl.DisplayNames(["en"], { type: "region" }).of(3);
```

引数として受け取るコードは次に解説する `type` オプションによって異なります。

### Intl.DisplayNames のオプション

Intl.DisplayNames のオプションは全コンストラクタプロパティ共通の `localeMatcher` を除くと以下の 4 つがあります。

- `type`
- `style`
- `fallback`
- `languageDisplay`

#### `type` オプション

`type` オプションは翻訳する対象を指定するオプションで、オプションの中で唯一必須のオプションです。とりうる値は以下の 6 つです。

- `"language"`
- `"region"`
- `"script"`
- `"currency"`
- `"calendar"`
- `"dateTimeField""`

この `type` オプションによって後述する `of()` メソッドに渡す引数のフォーマットが変わります。

##### `"language"` を指定した場合

`"language"` を指定した場合、Intl.DisplayNames インスタンスは言語名の表示名を取得するためのインスタンスとなります。

```ts
const languageNamesInEnglish = new Intl.DisplayNames(["en"], {
  type: "language",
});
languageNamesInEnglish.of("ja"); // "Japanese"
```

この場合、`of()` メソッドに渡す引数は言語コードとなります。 言語コードは LDML の [Unicode Language Id](https://unicode.org/reports/tr35/#unicode_language_id)に従ったロケール識別子の変化形サブタグまでの部分です。ロケール識別子については[2 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-02)を参照してください。

##### `"region"` を指定した場合

`"region"` を指定した場合、Intl.DisplayNames インスタンスは地域名の表示名を取得するためのインスタンスとなります。

```ts
new Intl.DisplayNames(["en"], { type: "region" }).of("US");
// "United States"
```

この場合、`of()` メソッドに渡す引数は地域コードとなります。地域コードは ２文字の大文字アルファベットからなる[ISO 3166 国別コード](https://www.iso.org/iso-3166-country-codes.html)か３文字の数字からなる[UNSD M49 地域コード](https://unstats.un.org/unsd/methodology/m49/)を指定します。

##### `"script"` を指定した場合

`"script"` を指定した場合、Intl.DisplayNames インスタンスは文字体系名の表示名を取得するためのインスタンスとなります。

```ts
new Intl.DisplayNames(["en"], { type: "script" }).of("Hans");
// "Simplified"
```

この場合、`of()` メソッドに渡す引数はスクリプトコードとなります。スクリプトコードは 4 文字のアルファベットからなる[ISO 15924 スクリプトコード](https://www.unicode.org/iso15924/iso15924-codes.html)を指定します。

##### `"currency"` を指定した場合

`"currency"` を指定した場合、Intl.DisplayNames インスタンスは通貨名の表示名を取得するためのインスタンスとなります。

```ts
new Intl.DisplayNames(["en"], { type: "currency" }).of("USD");
// 'US Dollar'
```

この場合、`of()` メソッドに渡す引数は通貨コードとなります。通貨コードは ３文字からなる[ISO 4217 通貨コード](https://www.iso.org/iso-4217-currency-codes.html)を指定します。

##### `"dateTimeField"` を指定した場合

`"dateTimeField"` を指定した場合、Intl.DisplayNames インスタンスは各日時フィールド(年月日..)の表示名を取得するためのインスタンスとなります。

```ts
new Intl.DisplayNames(["ja"], { type: "dateTimeField" }).of("day");
// '日'
```

この場合、`of()` メソッドに渡す引数は日時フィールドタイプとなります。日時フィールド名は `"era"`, `"year"`,`"quarter"`, `"month"`, `"day"`, `"weekday"`, `"dayPeriod"`, `"hour"`, `"minute"`, `"second"`,`"timeZoneName"` のいずれかを指定します。

##### `"calendar"` を指定した場合

`"calendar"` を指定した場合、Intl.DisplayNames インスタンスはカレンダー名の表示名を取得するためのインスタンスとなります。

```ts
new Intl.DisplayNames(["ja"], { type: "calendar" }).of("gregory");
// '西暦(グレゴリオ暦)'
```

この場合、`of()` メソッドに渡す引数はカレンダータイプとなります。カレンダータイプは Intl でサポートするカレンダータイプのいずれかを指定します。これは Intl.DateTimeFormat の `calendar` オプションで指定できるカレンダータイプ (= Intl Locale Info Proposal で提案されている `Intl.Locale.prototype.getCalendars().` で所得できるカレンダータイプ) です。

#### `style` オプション

`style` オプションは表示名のスタイルを指定するオプションです。とりうる値は以下の 3 つです。デフォルトは `"long"` です。

- `"narrow"` : 可能な限り短い形で表示
- `"short"` : 短い形・通称などで表示
- `"long"` : 省略しない完全な形で表示

```ts
new Intl.DisplayNames("ja-JP", { type: "region", style: "short" }).of("US");
// 「アメリカ」
new Intl.DisplayNames("ja-JP", { type: "region", style: "long" }).of("US");
// 「アメリカ合衆国」
```

#### `fallback` オプション

`fallback` オプション `of()` メソッドで指定したコードが見つからなかった場合のフォールバックを挙動指定するオプションです。`of()` メソッドは上記の通り、コードのフォーマットが不正な場合は RangeError をスローしますが、フォーマットが正しくてもそのコードが見つからない場合は `fallback` オプションで指定したコードを返します。それぞれ以下のような挙動をします。

- `"code"` : コードそのものを返す
- `"none"` : `undefined` を返す

```ts
// type : "region" で XX は存在しないが、
// 「２文字の大文字アルファベット」と言うフォーマットは満たしてる
new Intl.DisplayNames("ja-JP", { type: "region", fallback: "code" }).of("XX");
// "XX"
new Intl.DisplayNames("ja-JP", { type: "region", fallback: "none" }).of("XX");
// undefined
```

#### `languageDisplay` オプション

`languageDisplay` オプションは `type` オプションが `"language"` の場合にのみ有効なオプションです。このオプションは言語名の表示方法を指定するオプションで、とりうる値は以下の 2 つです。

- `"dialect"` : 特別な名前がある場合はその地域独自の名称を使って表示する。(デフォルト)
- `"standard"` : 標準の言語名と地域名のセットで表示する。

```ts
new Intl.DisplayNames("ja-JP", {
  type: "language",
  languageDisplay: "dialect",
}).of("en-US"); // "アメリカ英語"
new Intl.DisplayNames("ja-JP", {
  type: "language",
  languageDisplay: "standard",
}).of("en-US"); // "英語 (アメリカ合衆国)"
```

## Intl.DisplayNames のユースケース

### 翻訳言語の選択 UI

多言語対応がされているウェブアプリケーションや Web ページでは、ユーザーが表示言語を選択する UI がヘッダーや設定画面に用意されていることが多いです。またこのような UI では、言語名を「その言語で表示」することが一般的です。

![MDNの言語選択UI](/images/intlAdventCalendar24_22/mdnLangSelect.png)

このような UI を作成する場合、Intl.DisplayNames を使うと便利です。具体的にはサポートする言語コードを Intl.DisplayNames の初期化時と `of()` メソッドに渡すことで、その言語コードに対応する言語名を取得できます。

```ts
const languageCodes = ["de", "fr", "ja", "ko"];
const localeLanguageNames = languageCodes.map((code) =>
  new Intl.DisplayNames(code, { type: "language", style: "narrow" }).of(code)
);
console.log(localeLanguageNames);
//  ['Deutsch', 'français', '日本語', '한국어']
```

### ロケールに合わせた通貨選択の表示など

上記のユースケースは「各言語で自分の言語を表記する」ものでしたが、もちろんページのロケールに合わせた表示も Intl.DisplayNames のユースケースです。

特に通貨名などはいちいち文言として通貨名を用意しなくても簡単にロケールに合わせた通貨名を表示できまて便利です。

```ts
const availableCurrencies = ["USD", "JPY", "EUR"];
const currencyNames = availableCurrencies.map((code) =>
  new Intl.DisplayNames(navigator.language, { type: "currency" }).of(code)
);
console.log(currencyNames); // ['米ドル', '日本円', 'ユーロ']
```

## まとめと次回予告

この記事では言語、地域、文字体系などの表示名の一貫した翻訳をサポートする Intl.DisplayNames についてその API やユースケースを解説しました。

次回[23 日目]()では、Intl で統一的な文言フォーマットを策定しようと言う野心的な試み、Intl.MessageFormat Proposal についてその基本となるん部分を解説します。

## 参考文献

- Unicode Language Identifier (LDML)
  - https://unicode.org/reports/tr35/#unicode-language-identifier
