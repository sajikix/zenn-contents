---
title: "世界の暦に対応するための Proposal(#10)"
emoji: "🗓️"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: true
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 10 日目の記事です。

今回は世界中の暦をより柔軟にサポートするために提案されている、eraDisplay option for Intl.DateTimeFormat と Intl Era and MonthCode Proposal について解説します。

## 世界の暦は一つじゃない

現在日本を含め多くの国ではグレゴリオ暦が使われていますが、世界中には様々な暦が存在します。例えば、中国では中国暦、イスラム教圏ではヒジュラ暦、ユダヤ教圏ではヘブライ暦(ユダヤ暦)がグレゴリオ暦と共存しつつ使われています。また、日本でも伝統行事や一部官公庁のシステムなどで和暦が使われています。

従って、日時の書式化並びに日時の操作ではこれらの暦をサポートすることが求められます。基本的な書式化機能に関しては Intl.DateTimeFormat の `calendar` オプションや `era`　オプションで対応できますが、それでも不十分な場合や仕様上不明確な部分が存在しているのが現状です。(2024/12 現在)

これらの問題を解決するために、現在提案されているのが以下の 2 つの Proposal です。

- `eraDisplay` option for Intl.DateTimeFormat
- Intl Era and MonthCode Proposal

## eraDisplay option for Intl.DateTimeFormat

この Proposal は Intl.DateTimeFormat における暦の表示方法を柔軟に指定できるようにするという現在 Stage2 の提案です。

https://github.com/tc39/proposal-intl-eradisplay

現在の Intl.DateTimeFormat では、`era` オプションを指定することで、`era` をどのように表示するかどうかを指定できます。

```ts
const date = new Date(-1000, 0, 1);
const formatter = new Intl.DateTimeFormat("en-US", { era: "short" });
formatter.format(date); //'1/1/1001 BC'
```

しかし、この `era` オプションは年号の表記(`"long"`/`"short"`/`"narrow"`)を指定するオプションであり、`era` オプションを指定した時点で**必ず年号部分**は表記されてしまいます。逆に言えば指定しない限り `era` が現在と異なっても表示されず、例えば紀元前紀元後の区別はつかなくなります。

```ts
const date = new Date(-1000, 0, 1);
const formatter = new Intl.DateTimeFormat("en-US");
formatter.format(date); //'1/1/1001' ← 1001 年に見えちゃう
```

このように現状 Intl.DateTimeFormat には「`era` 自体をいつ表示するか」指定するオプションが存在しません。より具体的には以下のようなユースケースに対応できていないことになります。

- 紀元前の時だけ BC をつけるといった指定をしたい
  - `era` オプションを指定した時点で紀元後でも `AC` がついてしまう
  - 逆に `era` オプションを指定しないと紀元前でも `BC` がつかない
- 和暦だけど元号は表示しないといった指定をしたい
  - `{calendar: "japanese"}` を指定した時点で、`era` オプションを指定しなくても元号が表示される

このような課題に対し、この Proposal では新しく `eraDisplay` というオプションを Intl.DateTimeFormat に追加することを提案しています。`eraDisplay` は以下の３つの値のうちいずれかを取ります。

- `"never"`: 常に `era` を表示しない
- `"always"`: 常に `era` を表示する
- `"auto"`: 解決されたカレンダーで、年号が表示される & 日付の年号が今日の日付と異なる場合に `era` を表示する

この `eraDisplay` オプションを使うことで上記のユースケースも柔軟に対応できるようになります。

```ts
// 紀元前の時だけ BC をつける
const enFormatter = new Intl.DateTimeFormat("en-US", { eraDisplay: "auto" });
enFormatter.format(new Date(-1000, 0, 1)); // '1/1/1001 BC'
enFormatter.format(new Date(1000, 0, 1)); // '1/1/1001'

// 和暦だけど元号は表示しない
const jpFormatter = new Intl.DateTimeFormat("ja-JP-u-ca-japanese", {
  eraDisplay: "never",
});
jpFormatter.format(new Date(2024, 0, 1)); //'6年1月1日'
```

特に `eraDisplay: "auto"` はなるべく `era` の表記を省略しつつも、`era` の表記がないと分かりにくい場合は表示するという柔軟な挙動に対応しており便利そうですね。

## Intl Era and MonthCode Proposal

eraDisplay option は `era` の表示方法を柔軟に指定できるよう、 Int.DateTimeFormat を拡張する具体的な提案ですが、この Proposal は少し趣の異なる提案です。

https://github.com/tc39/proposal-intl-era-monthcode

この Proposal の目的は iso8601 以外のカレンダーや UTC 以外のタイムゾーンにおける `era` や `eraYer`, `monthCode` などの仕様をより詳細に定義することです。これらの仕様は Temporal Proposal で基本的な部分がカバーされていますが、今後の国際化機能の充実のためにもより詳細な仕様の策定と ECMA402 仕様書への統合が求められています。

ここでは具体的に現状のどこが問題なのか、この Proposal がどのような仕様を追加で提供するのかを見ていきましょう。

### 問題 1: era と eraYear の定義を明確化したい

現状 Temporal Proposal では `era` と `eraYear` を指定した日時の生成ができ、逆に特定の日時データから `era` や `eraYear` を取得できるよう提案されています。

```ts
const date = Temporal.PlainDate.from({
  calendar: "japanese",
  era: "meiji",
  eraYear: 1,
  month: 1,
  day: 1,
});
```

一方でこれらに関して具体的に以下のような部分の仕様が不明確なままであると指摘されています。

- それぞれの暦で有効な `era` と `eraYear` の範囲
- `era` を指定する文字列(=eraCode)の定義が厳密に定まっていない
- 暦によって `era` が複数ある場合のない場合が存在し、特に単数の時の挙動が曖昧
- そもそも `era` を指定しなかったときのフォールバック・デフォルトの挙動が不明確
- etc...

### 問題 2: 暦によって異なる月に対応する MonthCode の仕様

特定の暦、時に太陽太陰暦を採用している暦では 1 年が 12 ヶ月でなかったり、閏月(調整のため２回同じ月が来る)が存在したります。Temporal ではこれらの暦に対応するため `monthCode` を指定した日時の生成ができ、逆に特定の日時データから `era` や `eraYear` を取得できるよう提案されています。具体的には `M${month}` の形で指定し、閏月の場合は `L` をつけます。

```ts
const date = Temporal.PlainDate.from({
  calendar: "hebrew",
  year: 5779,
  monthCode: "M05L",
  day: 1,
});
```

ただこの仕様に関しても「どの暦でどの `monthCode` のパターンがあり得るのか」などが不明確なままであると指摘されています。

### この Proposal のゴール

これらの問題を解決するべく、この Proposal では各暦において取りうる `era` と `eraYear` の範囲、`era` のコード、`monthCode` のパターンなどを明確に定義し、ECMA402 に統合することを目指しています。

具体的には暦ごとに以下のような表を提供し、各暦における `era` と `eraYear` の範囲、`era` のコードや alias を定義し、入力値がこれらの要件を満たすか検証する抽象操作(Abstract Operation)を提供します。

![era aliases and range of eraYear](/images/intlAdventCalendar24_10/eraTable.png)

https://tc39.es/proposal-intl-era-monthcode/#table-eras

合わせて CLDR にも上記の `era` のコードや alias について修正を出しています。(CLDR については[6 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-06)を参照のこと)

https://github.com/unicode-org/cldr/pull/2665/files

また、`monthCode` に関しても各暦におけるパターンを定義し、入力値がこれらの要件を満たすか検証する抽象操作(Abstract Operation)を提供します。

![Additional Month Codes in Calendars](/images/intlAdventCalendar24_10/monthCode.png)
https://tc39.es/proposal-intl-era-monthcode/#sec-temporal-isvalidmonthecodeforcalendar

このほかにも上記の仕様追加を踏まえた Temporal の仕様の修正や、ECMA402 仕様書の変更・追加が提案されています。

## まとめと次回予告

今回は世界中の暦をサポートするために提案されている Proposal について解説しました。これらの Proposal が実装されることで、Temporal と Intl.DateTimeFormat がより柔軟に世界中の暦をサポートできるようになることが期待されます。

次回[11 日目](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-11)では、相対的な日時の書式化をサポートする Intl.RelativeTimeFormat について解説します。

## 参考文献

- eraDisplay option for Intl.DateTimeFormat
  - github: https://github.com/tc39/proposal-intl-eradisplay
- Intl Era and MonthCode Proposal
  - github: https://github.com/tc39/proposal-intl-era-monthcode
  - draft spec : https://tc39.es/proposal-intl-era-monthcode/
- Temporal Proposal
  - spec : https://tc39.es/proposal-temporal
  - github : https://github.com/tc39/proposal-temporal
