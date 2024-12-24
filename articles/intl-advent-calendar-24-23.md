---
title: "Intl.MessageFormat 基礎(#23)"
emoji: "👔"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: true
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 23 日目の記事です。

今回は Intl で統一的な文言フォーマットを扱えるようにしようと言う野心的な試み、Intl.MessageFormat Proposal についてその基本となる部分を解説します。

## 文言フォーマット

### 「文言フォーマット」とは

そもそもここで言う「文言フォーマット」とはどういったものでしょうか？

アプリケーションの多言語対応をする場合、以下のようにして UI や説明文として表示する文言を管理・表示・切り替えることが一般的です。

- 管理
  - 表示する文言ごとに key のような識別子をつける
  - 言語ごとに、各 key に対応する文言プールを用意する
- 表示
  - 表示側は文言を key で指定する
  - 表示するべき言語で文言プールを選び、 key から必要な文言を取得して表示する
- 切り替え
  - 表示言語を切り替える場合、key を探す文言プールを切り替える

例えば React を用いたアプリケーションでしばしば利用される国際化ライブラリ `i18next / react-i18next` では、以下のような JSON 形式で文言プールを管理します。

```json
{
  "en": {
    "greeting": "Hello, World!"
  },
  "ja": {
    "greeting": "こんにちは、世界！"
  }
}
```

このような文言プールを事前に読み込んでおき、表示する際に key で指定することで、表示言語に応じた文言を表示できます。

```tsx
function App() {
  const { t } = useTranslation();
  // key を指定することで現在使用する言語プールから文言を取得する
  return <h2>{t("greeting")}</h2>;
}
```

このように単純な文言であれば表示も簡単ですが、実際はもう少し複雑な機能が文言管理には求められます。例えばよくあるユースケースとして、「文言内にプレースホルダを埋め込みたい」というものがあります。この場合「どこに指定したプレースホルダが埋め込まれれるか」を文言リソース内で表現しなくてはなりません。

例えば、`react-i18next` では以下のように、`{{ }}` で囲った部分がプレースホルダとして扱われます。

```json
{
  "en": {
    "greeting": "Hello, {{name}}!"
  },
  "ja": {
    "greeting": "こんにちは、{{name}}さん！"
  }
}
```

この場合利用する側はオプションでプレースホルダの値を指定します。

```tsx
function App() {
  const { t } = useTranslation();
  return <h2>{t("greeting", { name: "Saji" })}</h2>;
  // => "こんにちは、Sajiさん！"
}
```

このように。「`{{ }}` で囲った部分がプレースホルダ key になる」やといった文言表記のルールをここでは「文言フォーマット」と呼びます。

### 文言フォーマットの多様性

web に限らず、アプリケーションの文言管理方法や文言のフォーマットは多岐に渡ります。例えば iOS アプリで利用されていた Localizable Stringsdict 形式は以下のようなフォーマットを持ちます。

```stringsdict
"messages" = "Hello %s";
```

この形式では `%s` のような文字がプレースホルダとして扱われます。

また Android アプリで利用される文言リソースは XML ファイルで、以下のようなフォーマットで管理されます。

```xml
<string name="greeting">Hello, %1%s!</string>
```

このように、文言フォーマットはアプリケーションのプラットフォームやライブラリによって様々なものが存在しています。

### 文言フォーマットに求められる機能

文言フォーマットに求められる機能はプレイスホルダー機能にとどまりません。例えば以下のような機能(記法)は文言のフォーマットとして求められがちな機能です。

- 文法的な条件分岐
- 数値や日付部分のフォーマット
- マークアップ埋め込み

#### 文法的な条件分岐

[17 日の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-17)でも触れた通り、多くの言語では単数複数の違いや人称の違いによって文言が変わります。このような場合、文法的な条件分岐処理を文言フォーマットに組み込みたくなります。

例えば、「N 件の未読通知があります」と言う文言を英語に翻訳する場合、未読通知の件数によって以下のように文言を変える必要があります。

- 1 件の場合 : `"You have 1 unread notification."`
- 2 件以上の場合 : `"You have N unread notifications."`
- 0 件の場合 : `"You have no unread notifications."`

これらを愚直に文言として登録すると以下のようになります。

```json
{
  "en": {
    "unreadNotification:one": "You have 1 unread notification.",
    "unreadNotification:other": "You have {{count}} unread notifications.",
    "unreadNotification:zero": "You have no unread notifications."
  }
}
```

一方で日本語は単数複数の違いがないため、以下のように登録するだけで済んでしまいます。

```json
{
  "ja": {
    "unreadNotification": "未読通知が{{count}}件あります。"
  }
}
```

これだど言語によって必要な key が変わってしまい、文言の管理は煩雑になってしまいます。可能であれば「key はどの言語も一意で、言語によってはいい感じに条件分岐できる」フォーマットが欲しいところです。このようなニーズに応えるため、文言フォーマットによっては単数系・複数系の条件分岐をサポートしているものがあります。例えば Android で利用される XML 形式では以下のように与えられ数値によって文言の条件分岐を記述できます。

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <plurals name="numberOfSongsAvailable">
        <!-- 単数の時 -->
        <item quantity="one">%d song found.</item>
        <!-- 複数の時 -->
        <item quantity="other">%d songs found.</item>
    </plurals>
</resources>

```

#### 数値や日付部分のフォーマット

この Intl Advent Calendar でも触れてきたように、数値や日付のフォーマットはロケールによって適切な形が異なりロジックも複雑になりがちです。そのため数値や日付部分に関しては「値を渡すだけでロケールに合わせてフォーマットし文言に埋め込んでくれる」ような機能があると便利です。

#### マークアップ埋め込み

web アプリケーションにおいては、文言を表示するだけでなく部分的な装飾やリンクを埋め込みたいというニーズが多いです。そのため文言フォーマットによっては、マークアップを埋め込む機能が提供されていることがあります。例えば Android で利用される XML 形式では以下のように XML 内で HTML のマークアップを記述できます。

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="welcome">Welcome to <b>Android</b>!</string>
</resources>
```

## Intl.MessageFormat

ここまで、文言フォーマットの指すものと、文言フォーマットの多様性、文言フォーマットに求められる機能について解説してきました。Intl.MessageFormat Proposal はこのようなニーズに対して、「**JS としての標準の文言フォーマットを策定**」し、「**標準化したフォーマットを扱う API も標準で提供**」しようという野心的な Proposal です。

### Intl.MessageFormat 定める文言フォーマット

ややこしいことに、Intl.MessageFormat が提案する文言のフォーマットは単に Intl が独自で策定しているわけではありません。Intl.MessageFormat とサポートする文言フォーマットに至るまでの経緯を理解するためには、ICU MessageFormat という文言フォーマットの存在を知る必要があります。

#### ICU MessageFormat

[6 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-06)で解説した通り、Unicode は ICU と言う国際化機能をまとめたライブラリを提供しています。この ICU には独自の文言フォーマットである ICU MessageFormat が存在します。

https://unicode-org.github.io/icu/userguide/format_parse/messages/

ICU MessageFormat は ICU が定義・運用している文言フォーマットで、20 年以上現役で業界でも広く使われるフォーマットの 1 つです。もちろんこのフォーマットを利用する機能も ICU に備わっており多くの言語やプラットフォームで利用されています。また、ICU MessageFormat はプレイスホルダの埋め込みや条件分岐、数値や日付のフォーマットなど多くの機能を持っているのも特徴です。

一方で ICU MessageFormat が提供されてから 20 年以上経過し、様々な課題も見えてきています。

- 拡張性の問題
  - 既存の文法を拡張することが難しい
  - 独自のフォーマット関数を提供できない
  - 既存のフォーマット関数を廃止できない
- ICU の他機能への追従や更新の遅れ
  - 新しく増えた ICU の機能を MessageFormat に反映できていない
  - 条件分岐の複雑さや実装事情が文言側に漏れてしまっている

#### ICU MessageFormat v2 と Intl.MessageFormat

このような課題を受けて、ICU では新しい MessageFormat を策定する機運が高まっていました。同時期に ECMAScript でも Intl による個別の国際化機能が整備され始め、Intl の各機能を統合した文言全体の国際化機能が求められるようになりました。

このような背景から、ICU と ECMAScript で協力しつつ策定されることになったのが「ICU MessageFormat v2」と呼ばれる文言フォーマットです。

https://github.com/unicode-org/message-format-wg

この ICU MessageFormat v2 の構文自体は Unicode の MessageFormat Working Group によって策定されており、Intl.MessageFormat はこの ICU MessageFormat v2 に準拠する形で標準化が進められています。

ICU MessageFormat v2 の詳しい構文に関しては、次回[24 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-24)で詳しく解説します。

### Intl.MessageFormat に求められる機能

ここまでの話を踏まえると、Intl.MessageFormat は ICU MessageFormat v2 に準拠した文言フォーマットを標準で扱うための API と言うことになります。 ICU MessageFormat v2 をサポートするには以下のような機能をサポートする必要があります。

- MessageFormat v2 内で埋め込まれる変数を渡す機能
- MessageFormat v2 で利用される関数を定義する機能

MessageFormat v2 では他の文言フォーマット同様、`{$[変数名]}` のような形で変数を埋め込むことができます。

```
Hello, {$name}!
```

そのため、Intl.MessageFormat 側ではフォーマット時に変数を渡す機能が必要です。また、MessageFormat v2 では `{$[変数名] :[関数名]}` のような形で変数を関数の引数として渡しつつその結果を埋め込むことができます。

```
Today is {$date :datetime}.
```

そのため、Intl.MessageFormat 側では MessageFormat v2 で利用される関数を定義する機能も必要になります。

もちろん、ロケールの指定やメッセージ自体の受け取りは Intl の他の機能と同様に行う必要があります。

### Intl.MessageFormat の API

現在提案されている Intl.MessageFormat の API を簡単に紹介します。

他の Intl のコンストラクタ同様、Intl.MessageFormat は第１引数にロケール(ロケール識別子 or Intl.Locale オブジェクト)を受け取ります。一方で第 2 引数には書式化オプションではなく MessageFormat v2 形式のメッセージ文字列を受け取ります。(書式化のオプションは第 3 引数として受け取る形になります。)

```ts
const message = "Hello, {$name}!";
const mf = new Intl.MessageFormat("en-US", message);
```

生成した Intl.MessageFormat インスタンスは `format()` メソッドを持ち、このメソッドに必要であれば変数を渡すことでフォーマットされたメッセージを取得できます。

```ts
mf.format({ name: "Saji" }); // "Hello, Saji!"
```

また、MessageFormat v2 で利用される関数を定義する場合は、Intl.MessageFormat 初期化時の第３引数でオプションの 1 つとして渡します。

```ts
const message = "Logged in at {$date :myDateTime}.";
const mf = new Intl.MessageFormat("en-US", message, {
  functions: {
    myDateTime: () => {
      // 何らかの日付フォーマット処理
    },
  },
});
mf.format({ date: new Date() });
```

さらに Intl.MessageFormat インスタンスは `formatToParts()` メソッドも持ち、フォーマットされた文言を「ハードコードされていた場所 / プレースホルダ部分 / 関数呼び出し結果」のような形で分類して返すことができます。

```ts
const mf = new Intl.MessageFormat("en", "Hello {$place}!");
mf.formatToParts({ place: "world" });
/* [
  { type: 'text', value: 'Hello ' },
  { type: 'string', source: '$place', value: 'world' },
  { type: 'text', value: '!' }
] */
```

またここでは詳しく触れませんが、Intl.MessageFormat は第 2 引数として、 MessageFormat v2 形式の文字列だけでなく「`MessageData` オブジェクト」も受け取れるようになっています。この `MessageData` オブジェクトは MessageFormat v2 で定義されている構文などをプログラム上で扱いやすいデータ形式で表現したものです。詳しくは MessageFormat 2.0 Data Model を参照してください。

https://github.com/unicode-org/message-format-wg/tree/main/spec/data-model

## まとめと次回予告

今回は、文言フォーマットとは何かという点から始まり、文言フォーマットに求められる機能、Intl.MessageFormat が ICU MessageFormat v2 に準拠した形で策定されるまでの経緯を解説しました。また、Intl.MessageFormat の基本的 API についても紹介しました。

一方この記事では Intl.MessageFormat/MessageFormat v2 の以下のような点にについては詳しく触れていません。

1. ICU MessageFormat v2 の構文
2. 多くの文言プールをどう扱うか

1 に関しては次の[24 日目の記事]()で、2 に関してはその次の[25 日目の記事]()でそれぞれ解説します。

## 参考文献

- Intl.MessageFormat Proposal
  - https://github.com/tc39/proposal-intl-messageformat
- ICU MessageFormat
  - https://unicode-org.github.io/icu/userguide/format_parse/messages/
- ICU MessageFormat v2
  - https://github.com/unicode-org/message-format-wg
- Android : 文字列リソース
  - https://developer.android.com/guide/topics/resources/string-resource?hl=ja
- react-i18next
  - https://react.i18next.com/
