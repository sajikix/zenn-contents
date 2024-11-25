---
title: "Intl.Locale のプロパティと Intl Locale Info proposal(#3)"
emoji: ""
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: false
publication_name: "cybozu_frontend"
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 3 日目の記事です。

今回は Intl.Locale オブジェクトのプロパティ・メソッドと Intl Locale Info proposal について解説します。

## Intl.Locale のプロパティ・メソッド

[2 日目の記事]()で解説した通り Intl.Locale オブジェクトを初期化する際には様々なオプションが指定できます。

この時作成された Intl.Locale オブジェクトはオプションで指定した値すべてをオプションと同じ key 名で参照できます。

```ts
const jpLocale = new Intl.Locale("ja-JP", { hourCycle: "h12" });
console.log(jpLocale.language); // "ja"
console.log(jpLocale.region); // "JP"
console.log(jpLocale.hourCycle); // "h12"
```

初期化時のオプションで指定しなかった物については `numeric` を覗き `undefined` として記録されています。(`numeric` は default 値が `false`)

これら初期化時にオプションとして存在した 9 つのプロパティに加え Intl.Locale オブジェクトは以下のプロパティとメソッドを持っています。

- `baseName`
  - この `baseName` プロパティは初期化オプションにおける `language`、`script`、`region` を `"-"` で繋いだ文字列を返します。
- ## `maximize()`
- `minimize()`
- `toString()`
  - `toString()` は `baseName` で得られる文字列とは異なり、

## Intl Locale Info proposal

### 提案されている Locale Info とユースケース
