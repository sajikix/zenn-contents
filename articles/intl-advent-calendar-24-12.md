---
title: "Intl.DurationFormat Proposal(#12)"
emoji: ""
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: false
publication_name: "cybozu_frontend"
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 11 日目の記事です。

今回は Intl.DurationFormat Proposal について解説します。

## Intl.DurationFormat

Intl.DurationFormat Proposal は「１時間３分」のような期間を書式化するために提案されている Proposal です。この Proposal は今年(2024 年)の 12 月の TC39 ミーティングで Stage4 に上がったため、2025 年には正式な Intl の仕様として策定され、ほとんどの環境で使えるようになる予定です。

### 基本的な IF と使い方

他の Intl のコンストラクタプロパティ同様、第１引数にロケール(ロケール識別子 or Intl.Locale オブジェクト)を第２引数にフォーマットのオプションを渡して初期化することで、Intl.DurationFormat インスタンスを生成できます。

```ts
const enFormatter = new Intl.DurationFormat("en-US", {
  // オプションを指定
});
const jaFormatter = new Intl.DurationFormat("ja-JP", {
  // オプションを指定
});
```

生成した Intl.DurationFormat インスタンスには Intl.DateTimeFormat や Intl.RelativeTimeFormat 同様 `format()` / `formatToParts()` のようなメソッドが生えており(詳しくは記述)、このメソッドに{}を渡すことで書式化できます。

### オプション

### メソッド

## ユースケース

## まとめと次回予告

## 参考文献
