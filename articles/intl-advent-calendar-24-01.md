---
title: "Intlの全体像と基本となるIF(#1)"
emoji: ""
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: false
publication_name: "cybozu_frontend"
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 1 日目の記事です。

アドベントカレンダーの初回である今回は JS の国際化 API「Intl」の仕様的な位置付けと基本となる構成要素・IF を紹介します。

## Intl とは

Intl は ECMAScript(JavaScript[^1]) にある名前空間で、`Intl` 配下にある API は「国際化」に関連する機能を提供しています。

### 国際化 API のやること

Intl では具体的に以下のような国際化機能を提供しています。

- 日時のフォーマット
- 数値のフォーマット
- 複数形や除数など各言語の数に対する文法上の分類判別
- 見た目上の一文字(書記素)や単語、文レベルの分ち書き

### 仕様の位置付けと策定

Intl は一般的な ECMAScript の仕様である ECMA262 とは別に ECMA402 という仕様番号で管理されています。ECMA402 は ECMA262 の拡張仕様という位置付けとなっており、 ECMA262 と ECMA402 を合わせた集合が ECMAScript 仕様であると考えて良いでしょう。

#### ECMA262 と分かれているもの

仕様番号の他にも Intl と他の ECMAScript 仕様ではいくつか異なる部分があります。

- 仕様書
- meeting とその議事録
- Proposal の一覧

仕様番号が違うということはもちろん仕様書も ECMA262 とは別に記述されています。

<!-- TODO 仕様書のリンク -->

また Intl(ECMA402) の仕様は TC39[^2] の中でも専用の TaskGroup(分科会？)で議論されていることが多いです。そのため ECMAScript 全体の meeting 以外にも Ecma402 だけで(ほぼ)月次の meeting が行われており、議事録も別で残されています。

https://github.com/tc39/ecma402/tree/main/meetings

もちろん ECMAScript 全体の meeting でも ECMA402 の Proposal について話されることはありますが、Proposal の詳細や近況を調べたい場合はこちらも参照する必要があるかもしれません。

さらに細かい違いですが、提案されている Proposal の一覧も ECMA262 に対するものとは別のパスで管理されているので探す時には注意です。

https://github.com/tc39/proposals/blob/HEAD/ecma402/README.md

## Intl の基本となる IF

### コンストラクタプロパティとメソッド

Intl は多くの機能をコンストラクタプロパティとして保持しており、2024 年 12 月現在 9 つのコンストラクタプロパティを持っています。

- List
<!-- todo list -->

またそれ以外にロケールや Intl 自体の機能に関するメタ的な情報を返すメソッドが 2 つ存在しています。

- List
<!-- todo list -->

### コンストラクタの基本となる IF

Intl の主要な機能はコンストラクタプロパティとして提供されるので使う際はそれぞれのコンストラクタプロパティを初期化 (`new`)　して使うことになります。初期化する際の引数はオプションの違いこそありますが、どのコンストラクタプロパティも比較的共通した形を持っています。

具体的に、各コンストラクタプロパティは皆以下の２つの引数を受け取ります。

1. ロケール文字列 or Intl.Locale オブジェクト
2. 初期化する際のオプション

[^1]: JavaScript の仕様に言及する場合は原則 ECMAScript の名称を使用します、
[^2]: 様々な仕様を策定する Ecma International の中で ECMAScript の策定・議論をする専門委員会(Technical Committee : TC)のこと
