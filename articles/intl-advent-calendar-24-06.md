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

### Unicode コンソーシアムの持つもの

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

## LDML

## CLDR

## ICU

## 参考文献

https://www.unicode.org/reports/about-reports.html
