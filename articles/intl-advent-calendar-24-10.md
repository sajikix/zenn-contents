---
title: "世界の暦に対応するための Proposal(#10)"
emoji: ""
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: false
publication_name: "cybozu_frontend"
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 9 日目の記事です。

今回は世界中の暦をより柔軟にサポートするために提案されている、eraDisplay option for Intl.DateTimeFormat と Intl Era and MonthCode Proposal について解説します。

## 世界の暦は一つじゃない

現在日本を含め多くの国ではグレゴリオ暦が使われていますが、世界中には様々な暦が存在します。例えば、中国では中国暦、イスラム教圏ではヒジュラ暦、ユダヤ教圏ではヘブライ暦(ユダヤ暦)がグレゴリオ暦と共存しつつ使われています。また、日本でも伝統行事や一部官公庁のシステムなどで和暦が使われています。

従って、日時の書式化並びに日時の操作では、これらの暦をサポートすることが求められます。基本的な書式化機能に関しては Intl.DateTimeFormat の `calendar` オプションや `era`　オプションで対応できますが、それでも不十分な場合や仕様上不明確な部分が存在しているのが現状です。(2024/12 現在)

これらの問題を解決するために、現在提案されているのが以下の 2 つの Proposal です。

- `eraDisplay` option for Intl.DateTimeFormat
- Intl Era and MonthCode Proposal

## eraDisplay option for Intl.DateTimeFormat

この Proposal は Intl.DateTimeFormat における暦の表示方法を柔軟に指定できるようにするという現在 Stage2 の提案です。

現在の Intl.DateTimeFormat では、`era` オプションを指定することで、`era` をどのように表示するかどうかを指定できます。

```ts
const date = new Date(-1000, 0, 1);
const formatter = new Intl.DateTimeFormat("en-US", { era: "short" });
formatter.format(date); //'1/1/1001 BC'
```

しかし、この `era` オプションは年号の表記(`"long"`/`"short"`/`"narrow"`)を指定するオプションであり、`era` オプションを指定した時点で必ず年号部分は表記されてしまいます。逆に言えば指定しない限り `era` が現在と異なっても表示されず、例えば紀元前紀元後の区別はつかなくなります。

```ts

```

このように現状 Intl.DateTimeFormat には「`era` 自体をいつ表示するか」指定するオプションが存在しません。より具体的に以下のようなユースケースに現状対応できていないことになります。

- 紀元前の時だけ BC をつけるといった指定をしたい
  - `era` オプションを指定した時点で紀元後でも `AC` がついてしまう
  - 逆に `era` オプションを指定しないと紀元前でも `BC` がつかない
- 和暦だけど元号は表示しないといった指定をしたい
  - `{calendar: "japanese"}` を指定した時点で、`era` オプションを指定しなくても元号が表示される

このような課題に対して、この Proposal は、新しく `eraDisplay` というオプションを Intl.DateTimeFormat に追加することを提案しています。`eraDisplay` は以下の３つの値のうちいずれかを取ります。

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

## Intl Era and MonthCode Proposal

eraDisplay option は `era` の表示方法を柔軟に指定できるよう、 Int.DateTimeFormat を拡張する具体的な提案ですが、この Proposal は少し趣の異なる提案です。

この Proposal の目的は特定の iso8601 カレンダーや UTC 以外のタイムゾーンにおける `era` や `eraYer`, `monthCode` などの仕様をより詳細に定義することです。これらの使用は Temporal Proposal で基本的な部分がカバーされていますが、今後の国際化のためにもより詳細な仕様の策定と ECMA402 への統合が求められています。

### era と eraYear の定義を明確化する

### 暦によって異なる月に対応する MonthCode

## まとめと次回予告

## 参考文献

<!-- 世界の暦に対応するための Proposal -->

```

```
