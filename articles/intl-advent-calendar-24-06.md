---
title: "Intl を支える LDML と CLDR と ICU(#6)"
emoji: ""
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: false
publication_name: "cybozu_frontend"
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 6 日目の記事です。

今回は Intl の仕様や実装を知る上で欠かせない LDML と CLDR、ICU について解説します。

## Intl の仕様と Unicode

そもそも Intl は国際化機能という性質上、他の標準仕様やデータベースを参照する部分が多いという特徴を持っています。その中でもとりわけ Unicode コンソーシアムの持つ仕様やデータベースとは切っても切り離せない関係にあります。

例えば以下のような機能についてはみな Unicode 側の仕様を参照する形で ECMA402 仕様書が書かれています。

- 文字セット
- 文字の区切り方に関する仕様
- 文字自体の比較・並び替えに関する仕様

これ以外にも Unicode コンソーシアムは様々な国際化に関する仕様や文献、データベースを持っています。さらにこれらの仕様に準拠した国際化ライブラリを公開しており、JavaScript エンジンの多くはこのライブラリを利用しています。

### Unicode コンソーシアムの持つものと技術文書

Unicode というと一般的には Unicode コンソーシアムが定めた、文字コードの業界標準規格のこと指します。しかし Unicode コンソーシアムでは文字コード以外にも以下のような仕様・文書・データベース。ライブラリなどを管理しています。

- Unicode Technical Reports : ソフトウェアアルゴリズムやデータ構造の仕様や技術文書
- CLDR : ロケールの構造化データを策定するプロジェクト(後述)
- ICU : 国際化と地域化に関する機能を持つライブラリ(後述)

Unicode Technical Reports には Unicode テキストの正規化、テキスト内の改行の機会やその他の分節境界のような文字コードや国際化に関する仕様・文書が公開されています。Unicode Technical Reports は次のように分類されます。

- Unicode Standard Annex(UAX)
  - Unicode (文字コードの)標準とは別の文書だけど不可欠な仕様
  - 例 : Unicode Standard Annex #29: Unicode Text Segmentation
- Unicode Technical Standard (UTS)
  - Unicode (文字コードの)標準とは独立した仕様です
  - 例 : Unicode Technical Standard #10: Unicode Collation Algorithm
- Unicode Technical Report (UTR)
  - 上記以外の有益な資料

特に UAX と UTS は ECMA402 仕様書からも参照される事が多いです。例えば UAX#29 の「Unicode Text Segmentation」は Intl.segmenter の仕様で、UTS#10 の「Unicode Collation Algorithm」Intl.Collator の仕様でそれぞれ参照されています。

また UTS#35 の LDML は ECMA402 の多くの部分で参照される Intl 仕様書を読む上で外せない仕様です。この LDML は CLDR や ICU とも関わってくるので次章で解説します。

## LDML

LDML は Locale Data Markup Laguage の略でその名前の通り、ロケールや国際化に関するあらゆるデータを記録するための統一的なフォーマットです。具体的 LDML では以下のようなロケール情報の取りうる値やデータの形、分類などを定めています。

- a
- b

ここでは具体的にのデータの仕様を見てみましょう。

<!-- 具体例 -->

このように LDML では多くのロケール固有な値を管理するために設計しており、ECMA402 仕様書でも以下のような場所で参照されています。

- a
- b

## CLDR

LDML は「ロケールや国際化に関するデータを統一的管理するためのフォーマット」でした。この LDML の形式で世界中のロケールデータを集め公開しているのが CLDR() です。またこの取り組み自体を CLDR Project と呼びます。CLDR は github で公開されており、誰でも参照できます。

<!-- URL -->

現在皆さんが利用している JS エンジンはバージョンや内容の差こそあれ、この CLDR のデータを元に国際化機能を提供しています。

<!-- 例えば -->

## ICU

ロケールデータの統一的フォーマットである LDML とそのフォーマットを元にした実データである CLDR があることで、ランタイム間で統一的な国際化機能を提供するのが楽になりました。一方でデータのフォーマットやデータがあっても以前として国際化機能のロジックはロケールなどによって例外などが多く複雑です。

そのため Unicode コンソーシアムでは一般的な国際化機能をサポートするライブラリ、ICU()を公開しています。ICU には C/C++実装の ICU4C、Java 実装の ICU4J、Rust 実装の ICU4X があり、それぞれ以下のような機能を提供しています。

- a
- b

多くの JavaScript エンジンは Intl のような国際化機能を実装する際、この ICU(大抵は ICU4C か ICU4X)を利用しています。例えば Intl.Segmenter の機能は ICU の BreakIterator の機能を利用して実装されています。

<!-- zennのsegmenterの記事 -->

## まとめと次回予告

この記事では Unicode コンソーシアムの提供する仕様やデータライブラリ群によって Intl の仕様や実装が支えられていることを解説しました。次回 7 日目からは Intl.DateTimeFormat について解説していきます。

## 参考文献

https://www.unicode.org/reports/about-reports.html
