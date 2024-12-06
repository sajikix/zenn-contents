---
title: "Intlを支えるUnicodeの仕様・データ・ライブラリ(#6)"
emoji: "🗂️"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: true
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 6 日目の記事です。

今回は Intl の仕様や実装を知る上で欠かせない Unicode の持つ仕様、ロケールデータのデータベース、国際化ライブラリについてそれぞれ解説します。

## Intl の仕様と Unicode

Intl は国際化機能という性質上、他の標準仕様やデータベースを参照する部分が多いという特徴を持っています。その中でも特に Unicode コンソーシアムの持つ仕様やデータベースとは切っても切り離せない関係にあります。

例えば、以下のような機能についてはすべて Unicode 側の仕様を参照する形で ECMA-402 仕様書が書かれています。

- 使用する文字セット
- 文字の区切り方に関する仕様
- 文字自体の比較・並び替えに関する仕様

これ以外にも Unicode コンソーシアムは様々な国際化に関する仕様や文献、データベースを持っています。さらに、これらの仕様に準拠した国際化ライブラリを公開しており、JavaScript エンジンの多くはこのライブラリを利用しています。

### Unicode コンソーシアムの持つものと技術文書

Unicode というと一般的には Unicode コンソーシアムが定めた文字コードの業界標準規格を指します。しかし、Unicode コンソーシアムでは文字コード以外にも以下のような仕様・文書・データベース・ライブラリなどを管理しています。

- Unicode Technical Reports: ソフトウェアアルゴリズムやデータ構造の仕様や技術文書
- CLDR: ロケールの構造化データを策定するプロジェクト（後述）
- ICU: 国際化と地域化に関する機能を持つライブラリ（後述）

Unicode Technical Reports には Unicode テキストの正規化、テキスト内の改行の機会やその他の分節境界のような文字コードや国際化に関する仕様・文書が公開されています。Unicode Technical Reports は次のように分類されます。

- Unicode Standard Annex (UAX)
  - Unicode（文字コードの）標準とは別の文書だが不可欠な仕様
  - 例: Unicode Standard Annex #29: Unicode Text Segmentation
- Unicode Technical Standard (UTS)
  - Unicode（文字コードの）標準とは独立した仕様
  - 例: Unicode Technical Standard #10: Unicode Collation Algorithm
- Unicode Technical Report (UTR)
  - 上記以外の有益な資料

特に UAX と UTS は ECMA-402 仕様書からも参照されることが多いです。例えば、UAX #29 の「Unicode Text Segmentation」は `Intl.Segmenter` の仕様で、UTS #10 の「Unicode Collation Algorithm」は `Intl.Collator` の仕様でそれぞれ参照されています。

また、UTS #35 の Unicode Locale Data Markup Language (LDML) は ECMA-402 の多くの部分で参照される Intl 仕様書を読む上で外せない仕様です。

https://unicode.org/reports/tr35/

次章ではこの LDML についてもう少し詳しく解説します。

## LDML

LDML は Locale Data Markup Language という名前の通り、ロケールや国際化に関するあらゆるデータを記録するための統一的な XML によるフォーマットです。

LDML では以下のようなロケールや国際化に関する情報の取りうる値やデータの形、分類などを定めており、その種類は多岐に渡ります。

- ロケール識別子について
- 数値のフォーマットについて
  - 単位・通貨がどう使われどう表示されるか
- 日時のフォーマットについて
  - カレンダーの種類や時刻の表記方法
- 複数形や序数について
- など

このように分類のカバーする範囲が多いことから、仕様自体も Part 1 から Part 9 までに分かれています。

- Part 1: Core (languages, locales, basic structure)
- Part 2: General (display names & transforms, etc.)
- Part 3: Numbers (number & currency formatting)
- Part 4: Dates (date, time, time zone formatting)
- Part 5: Collation (sorting, searching, grouping)
- Part 6: Supplemental (supplemental data)
- Part 7: Keyboards (keyboard mappings)
- Part 8: Person Names (person names)
- Part 9: MessageFormat (message format)

(LDML Part 1 より引用)

ただ、これだけでは具体的にどのような形式でデータが記述されているのか分かりにくいと思います。そこで具体的に命数法（Numbering System）の記述形式を例にとって仕様を見てみましょう。

命数法に関する仕様は Part 3 に記述されています。

https://unicode.org/reports/tr35/tr35-numbers.html#numbering-systems

この仕様によると、命数法は以下のような形式で記述されます。

```
<!ELEMENT numberingSystems ( numberingSystem* ) >
<!ELEMENT numberingSystem EMPTY >
<!ATTLIST numberingSystem id NMTOKEN #REQUIRED >
<!ATTLIST numberingSystem type ( numeric | algorithmic ) #REQUIRED >
<!ATTLIST numberingSystem radix NMTOKEN #IMPLIED >
<!ATTLIST numberingSystem digits CDATA #IMPLIED >
<!ATTLIST numberingSystem rules CDATA #IMPLIED >
```

これは `numberingSystem` というタグ名で各命数法の情報を記述でき、空白を挟んで属性には `id` / `type` / `radix` / `digits` / `rules` を持つことを示しています。属性名の次はその属性の型を示しており、`#REQUIRED` は必須、`#IMPLIED` はものによって必要か分かれることを示しています。

具体的な例を見てみましょう。いわゆるアラビア数字の命数法は以下のように記述されます。

```xml
<numberingSystem id="latn" type="numeric" digits="0123456789"/>
```

このように LDML では多くのロケール固有な値を XML で構造的に管理できるようなフォーマットを定義しており、ECMA-402 仕様書でもこの LDML に準拠することを明記されている部分があります。

- Intl.PluralRules
  - 複数形のパターン・ルールは LDML の言語複数ルールに準拠
- Intl.ListFormat
  - 内部的なデータの形式として LDML の ListPattern に準拠すること

このように LDML は Intl の仕様を調べたり理解する上で押さえておきたい仕様の 1 つです。

## CLDR

LDML は「ロケールや国際化に関するデータを統一的に管理するためのフォーマット」でした。この LDML の形式で世界中のロケールデータを集め、公開しているのが [Common Locale Data Repository (CLDR)](https://cldr.unicode.org/) です。また、この取り組み自体を CLDR Project と呼びます。CLDR は GitHub で公開されており、誰でも参照できます。

https://github.com/unicode-org/cldr

ほとんどの JS エンジンはバージョンや内容の差こそあれ、この CLDR のデータを元に国際化機能を提供しています。例えば、Intl の以下のようなデータはすべて CLDR のデータを元にしています。

- ロケール識別子でサポートする言語・文字体系・地域などの一覧
- サポートする単位・通貨の一覧
- サポートするカレンダーの一覧
- 文字並び替えの種類
- 言語による文法的な様々なルール（複数形/序数など）
- サポートするタイムゾーン

また、ECMA-402 仕様書でも各所で「実装において CLDR を利用することを推奨」しています。（仕様で指定しているわけではなく、あくまでも NOTE として推奨する形）。具体的には以下のような部分です。

- Intl.DateTimeFormat でのロケールやカレンダー
- Intl.DisplayNames で利用するデータ
- Intl.ListFormat の ListPattern のデータ
- など

従って、LDML 同様 CLDR も Intl の仕様を読む上で知っておきたい知識となります。

## ICU

ロケールデータの統一的フォーマットである LDML とそのフォーマットを元にした実データである CLDR があることで、ランタイム間で統一的な国際化機能を提供するのが楽になりました。一方で、ロケールデータのフォーマットやデータベースがあっても、依然として国際化機能のロジックはロケールによって例外も多く複雑です。

そのため、Unicode コンソーシアムでは一般的な国際化機能をサポートするオープンソースのライブラリ、International Components for Unicode (ICU) を公開しています。ICU には C/C++ 実装の ICU4C、Java 実装の ICU4J、Rust 実装の ICU4X が存在しており、利用する言語によって使い分けることができます。

- ICU4C/ICU4J: https://github.com/unicode-org/icu
- ICU4X: https://github.com/unicode-org/icu4x

これらの ICU は実装言語によって多少異なるものの、以下のような機能を提供しています。

- 符号化されたテキストと Unicode との変換
- 数値、日時、通貨などの書式化
- 時刻に関する演算
- グレゴリオ暦およびその他の暦の取り扱い
- タイムゾーンに関する演算
- Unicode のサポート
- Unicode 正規化、大文字・小文字の処理、その他 Unicode Standard に基づく処理
- 正規表現
- 双方向テキストに関する処理
- テキスト境界（単語、文、段落などの境界）の判別
- 行折り返しと単語折り返し（英語版）のための処理

多くの JavaScript エンジンは Intl のような国際化機能を実装する際、この ICU（大抵は ICU4C か ICU4X）を内部的に利用しています。例えば、Intl.Segmenter の機能は ICU の BreakIterator の機能を利用して実装されています。実際どのように利用しているかはコードを読んだ記事を書いているので、ぜひ参考にしてみてください。

https://zenn.dev/cybozu_frontend/articles/explore-intl-segmenter

## まとめと次回予告

この記事では Unicode コンソーシアムの提供する仕様やデータライブラリ群によって Intl の仕様や実装が支えられていることを解説しました。次回 [7 日目]()からは Intl.DateTimeFormat について解説していきます。

## 参考文献

- Unicode Technical Reports について
  - https://www.unicode.org/reports/about-reports.html
- LDML
  - https://unicode.org/reports/tr35/
- CLDR Project
  - https://cldr.unicode.org/
- ICU Documentation
  - https://unicode-org.github.io/icu/
