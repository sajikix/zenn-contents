---
title: "Intl.Segmenterはどうやって単語分割しているのか"
emoji: "🕵️‍♂️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["JavaScript", "frontend", "Intl", "i18n", "Unicode"]
publication_name: "cybozu_frontend"
published: false
---

## Intl.Segmenter についておさらい

JavaScript には Intl と呼ばれる国際化 API があり、日時や数値のフォーマットを始めとする国際化に便利な機能が揃っています。Intl.Segmenter はこの Intl の一機能で、文字・単語・文章単位での文字列分割を可能にします。

文字単位での分割では複数のコードユニットやコードポイントを持った文字を考慮し、正確に見た目上の１文字（書記素）で分割できるので、絵文字を含んだ文字数のカウントなどに便利です。

```ts
const segmenter = new Intl.Segmenter("ja", { granularity: "grapheme" });
console.log("🇯🇵👨🏻‍💻".length); // ❌ 11
console.log([..."🇯🇵👨🏻‍💻"].length); // ❌ 6
console.log([...segmenter.segment("🇯🇵👨🏻‍💻")].length); // ✅ 2
```

また単語単位の分割では、日本語・中国語のように単語間にスペースがない言語でも分割できます。

```ts
// 日本語の文を単語単位で分割するsegmenterの例
const segmenter = new Intl.Segmenter("ja", { granularity: "word" });
console.log([...segmenter.segment("吾輩は猫である")]);
/* 
[
    { "segment": "吾輩", ... "isWordLike": true },
    { "segment": "は", ... "isWordLike": true },
    { "segment": "猫", ... "isWordLike": true },
    { "segment": "で", ... "isWordLike": true },
    { "segment": "ある", ... "isWordLike": true }
]*/
```

:::message
Intl.Segmenter は 2024 年 2 月現在では Firefox でサポートされておらず、残念ながらブラウザ上で動くプロダクトに利用できる状況ではありません。しかし今後 FireFox v125 でサポートされる予定となっており、今年の前半には主要な JS エンジンでの実装が揃うこととなりそうです。
:::

## 単語でも分割できる！でもどうやって？

初めて Intl.Segmenter の機能を知った人の中には単語単位で分割ができることを知って驚かれる方もいたのではないでしょうか？（筆者の所感ですが実際 Segmenter の機能の中でも特にフィーチャーされることが多い機能だと思っています。）

しかし本来日本語や中国語のような単語間にスペースがない言語の単語分割は容易でなく、何らかの辞書や学習データを持つ形態素解析器のようなものが必要になります。ということは各 JavaScript エンジンにそのような機能が実装されているのでしょうか？

## ECMAScript 仕様書ではどうなっているのか

まず確認すべきなのは JavaScript の仕様書である ECMAScript の仕様書でしょう。`Intl` は JavaScript の拡張仕様として ECMA-402 という仕様番号で管理されおり、仕様書も JavaScript 本体の仕様書(ECMA-262 仕様書)と分かれています。

https://tc39.es/ecma402/

Segmenter についての仕様は [18 章](https://tc39.es/ecma402/#segmenter-objects)に書かれており、この中で実際に境界を探す処理は [`FindBoundary` と呼ばれる Abstract Operation](https://tc39.es/ecma402/#sec-findboundary)として定義されています。この `FindBoundary` ですが、定義の下に以下のような注釈がついています。

> Boundary determination is implementation-dependent, but general default algorithms are specified in Unicode Standard Annex #29. It is recommended that implementations use locale-sensitive tailorings such as those provided by the Common Locale Data Repository (available at https://cldr.unicode.org).

「一般的には Unicode の仕様書に従う」としつつ、はっきりと「実装による」と書かれていますね。つまり ECMAScript 仕様書自体は Intl.Segmenter における単語境界の検出方法を指定していないことになります。

## 実装を見てみよう

「実装による」と書かれている以上実装を見るしかありません。今回は主要ブラウザ並びにランタイムが利用している以下の JS エンジンの実装を見てみます。

- V8
- JavaScriptCore
- SpiderMonkey

### V8 での Intl.Segmenter の実装

V8 のコードは [Google Git のページ](https://chromium.googlesource.com/v8/v8/+/refs/heads/main) から閲覧可能です。検索やコードジャンプなどは使えないので clone してきた方が読みやすいでしょう。(コードを読むだけなら [github の公式ミラーリポジトリ](https://github.com/v8/v8)から clone だけしても良いですが、ビルドなどをしたい場合は手順に従ってください。)

https://v8.dev/docs/source-code

`FindBoundary` Abstract Operation は `Intl.Segments.prototype.containing()` と `IntlSegmentIteratorPrototype.next()` の二箇所で参照されていますが、今回は Segmenter が持つ Iterator の `next()` メソッドが呼ばれるところから実装を追ってみます。V8 の実装では仕様書の step 表記をそのままコメントとして採用していることが多いので「`FindBoundary`」のように Abstract Operation の名前で検索しても見つけることができます。今回だと[/js-segment-iterator.cc](https://chromium.googlesource.com/v8/v8/+/refs/heads/main/src/objects/js-segment-iterator.cc#79)の 79 行目あたりが該当します。

```cpp
// 6. Let endIndex be ! FindBoundary(segmenter, string, startIndex, after).
int32_t end_index = icu_break_iterator->next();
```

`icu_break_iterator` 自体は 75 行目で `JSSegmentIterator::Next` の引数の `segment_iterator` から得たポインタ変数です。

```cpp
icu::BreakIterator* icu_break_iterator =
    segment_iterator->icu_break_iterator()->raw();
```

よって　`JSSegmentIterator::Next` の呼び出し元で `segment_iterator` を渡しているはずなので...という形で追っていくと、最終的に[`src/objects/js-segmenter.cc` の 86 行目](https://chromium.googlesource.com/v8/v8/+/refs/heads/main/src/objects/js-segmenter.cc#86)から `granularity_enum`("grapheme" / "word" / "sentence")に応じて `icu_break_iterator` (スマートポインタ)の `reset` 関数で `icu::BreakIterator::create〇〇Instance` メソッドを代入しているところに行き着きます。

```cpp
  switch (granularity_enum) {
    case Granularity::GRAPHEME:
      icu_break_iterator.reset(
          icu::BreakIterator::createCharacterInstance(icu_locale, status));
      break;
    case Granularity::WORD:
      icu_break_iterator.reset(
          icu::BreakIterator::createWordInstance(icu_locale, status));
      break;
    case Granularity::SENTENCE:
      icu_break_iterator.reset(
          icu::BreakIterator::createSentenceInstance(icu_locale, status));
      break;
  }
```

この `icu::BreakIterator` は同じファイルの 22 行目で `unicode/brkiter.h` からインクルードしているのがわかります。

```cpp
#include "unicode/brkiter.h"
```

つまり最終的には ICU の [BreakIterator(common/brkiter.cpp)](https://github.com/unicode-org/icu/blob/9515e82741875b09fbd223573fb5e6d23fad6023/icu4c/source/common/brkiter.cpp) を利用しているということがわかりました。

### JavaScriptCore での Intl.Segmenter の実装

同様に JavaScriptCore での実装も見てみましょう。JavaScriptCore の実装は以下の github で公開されています。

https://github.com/WebKit/WebKit/tree/main/Source/JavaScriptCore

V8 同様に Segmenter が持つ Iterator の `next()` メソッドが呼ばれるところから実装を追ってみます。SegmentIterator の `next()` メソッドの実装は [Source/JavaScriptCore/runtime/IntlSegmentIterator.cpp の 71 行目](https://github.com/WebKit/WebKit/blob/dffba3d2838bbede6722f804db91807f79afe63b/Source/JavaScriptCore/runtime/IntlSegmentIterator.cpp#L71)にからになります。この中で `FindBoundary` Abstract Operation に該当するのは次の部分になります。

```cpp
int32_t endIndex = ubrk_next(m_segmenter.get());
```

この `ubrk_next` は [Source/JavaScriptCore/runtime/IntlSegmenter.h](https://github.com/WebKit/WebKit/blob/dffba3d2838bbede6722f804db91807f79afe63b/Source/JavaScriptCore/runtime/IntlSegmenter.h) 経由(`IntlSegmentIterator` が `IntlSegmenter` をインクルードしている)で `unicode/ubrk.h` からインクルードされています。

よって JavaScriptCore でも内部的に ICU の BreakIterator を利用していることがわかりました。

:::message
あとで unicode/ubrk.h と unicode/brkiter.h の違いを書く
:::

### SpiderMonkey での Intl.Segmenter の実装

最後に SpiderMonkey で現在実装が進められている部分も見てみましょう。SpiderMonkey のコードは以下のページで読むことができます。

https://searchfox.org/mozilla-central/source/js/src

SpiderMonkey の場合、Segmenter の実装は[builtin/intl/Segmenter.cpp](https://searchfox.org/mozilla-central/source/js/src/builtin/intl/Segmenter.cpp)にまとまっています。ここの [`FindSegmentBoundaries` というメソッド](https://searchfox.org/mozilla-central/source/js/src/builtin/intl/Segmenter.cpp#857)が境界の検知を行なっており、実際 Granularity の種類によって異なる関数を呼び出しています。

```cpp
switch (segments->getGranularity()) {
  case SegmenterGranularity::Grapheme: {
    boundaries = GraphemeBoundaries(segments, index);
    break;
  }
  case SegmenterGranularity::Word: {
    boundaries = WordBoundaries(segments, index);
    break;
  }
  case SegmenterGranularity::Sentence: {
    boundaries = SentenceBoundaries(segments, index);
    break;
  }
}
```

`WordBoundaries` を調べると最終的には `FindBoundaryFrom` という関数でもらったイテレーターの next()メソッドを呼び出しています。この Segment Iterator を生成しているのが 490 行目からの部分になります。

```cpp
switch (granularity) {
  // ... 略
  case SegmenterGranularity::Word: {
    auto* seg = CreateSegmenter<WordSegmenter>(cx);
    if (!seg) {
      return false;
    }
    segmenter->setSegmenter(seg);
    break;
  }
  // ... 略
}
```

この WordSegmenter 構造体の定義を見ると次のようになっています。

```cpp
struct WordSegmenter {
  using Segmenter = capi::ICU4XWordSegmenter;
  using BreakIteratorLatin1 =
      SegmenterBreakIteratorType<WordSegmenterBreakIteratorLatin1>;
  using BreakIteratorTwoByte =
      SegmenterBreakIteratorType<WordSegmenterBreakIteratorTwoByte>;

  static constexpr auto& create = capi::ICU4XWordSegmenter_create_auto;
  static constexpr auto& destroy = capi::ICU4XWordSegmenter_destroy;
};
```

ここで利用されている ICU4XWordSegmenter の定義元に飛ぶと [source/intl/icu_capi/c/include/ICU4XWordSegmenter.h](https://searchfox.org/mozilla-central/source/intl/icu_capi/c/include/ICU4XWordSegmenter.h)で定義されています。このファイルのように `source/intl/icu_capi/c` 配下に定義されているヘッダーファイルは ICU4X のコードを FFI 経由で C や C++から呼び出すための生成されたヘッダーファイルです。

https://searchfox.org/mozilla-central/source/intl/icu_capi/c/README.md

> This folder contains the C FFI for ICU4X.

つまり SpiderMonkey では ICU4X の Segmenter を利用していることがわかりました。（ICU4X については後述します。）

https://github.com/unicode-org/icu4x/tree/main/components/segmenter

## ICU ってなんだっけ？

ここまで各 JS ランタイムが ICU を利用していることがわかりましたが、「そもそも ICU ってなんだっけ？」という方向けに簡単に ICU について説明します。

ICU は International Components for Unicode の略で Unicode 文字自体の取り扱いや国際化・地域化を行うオープンソースのライブラリです。現在 C/C++　向けの ICU4C と Java 向けの ICU4J 、Rust で書かれより広い環境での実行を目指す ICU4X などが開発されています。

https://icu.unicode.org/

ICU は具体的には以下のような機能を持っています。

- 他の文字セットやエンコーディングと Unicode との変換
- 文字列の比較
- 数値・日時などの書式化
- 日時の計算
- Unicode 仕様に基づく Unicode 正規化、大文字小文字の折りたたみなど
- 正規表現
- 双方向テキストの処理
- テキスト境界の判別

V8 並びに JavaScriptCore はこの中でもテキスト境界の判別の機能である BreakIterator の C/C++ 実装版（ICU4C）を使っていることになります。

一方 SpiderMonkey は ICU4C ではなく、ICU4X の Segmenter を利用しています。ICU4X は Rust で書かれ、FFI などを通じて Rust 以外の様々な言語で利用されるよう設計された比較的新しい ICU の派生ライブラリです。

https://blog.unicode.org/2022/09/announcing-icu4x-10.html

今までの ICU4C・ICU4J の知見や Intl での知見を活かし、モジュール化・プラグイン可能なロケールデータなどの特徴を持ったライブラリになっています。

:::message
SpiderMonkey で ICU4X を利用するにあたり ICU4C との挙動の違いなどの問題はあったようで 2024 年 02 月現在でも Firefox ではこれらの挙動差の調整などがリリースに向けて進んでいそうです。
https://bugzilla.mozilla.org/show_bug.cgi?id=1423593
:::

## ICU はどうやって単語レベルの分割をしているのか

そうなると ICU4C の BreakIterator や ICU4X の Segmenter がどうやって単語レベルの分割を行なっているのかが気になるところです。

### ICU4C の BreakIterator の場合

ICU4C の BreakIterator 実装に関しては ICU の UserGuide に記述があります。

https://unicode-org.github.io/icu/userguide/boundaryanalysis/break-rules.html#details-about-dictionary-based-break-iteration

> For Chinese and Japanese text, on the other hand, we have a unified dictionary (due to the fact that both use some of the same characters, it is difficult to distinguish them) that contains information about word frequencies. The algorithm to match text then uses dynamic programming to find the set of breaks it considers “most likely” based on the frequency of the words created by the breaks.

ここにある通り、辞書にある単語データとそれぞれの出現頻度から動的計画法を利用して単語の分割を行なっています。具体的な実装は以下の 1299 行目から続く部分になります。

https://github.com/unicode-org/icu/blob/b8271577b66f0a29409d40b686505e917538c1e8/icu4c/source/common/dictbe.cpp#L1299

利用している辞書データは [icu/main/icu4c/source/data/brkitr/dictionaries/cjdict.txt](https://raw.githubusercontent.com/unicode-org/icu/main/icu4c/source/data/brkitr/dictionaries/cjdict.txt) に日本語と中国語のセットとして存在し、単語とその出現頻度が並んでいます。このファイルの冒頭に書かれている通り、日本語部分に関しては形態素解析ツールである ChaSen に内包されていた IPA 辞書を利用しており、場合によって単語を追加しているようです。（近年だと「令和」などが追加されている）

### ICU4X の Segmenter の場合

ICU4X の wordSegmenter の実装は以下になります。

https://github.com/unicode-org/icu4x/blob/main/components/segmenter/src/word.rs

この実装によると [159 行目からのコメント](https://github.com/unicode-org/icu4x/blob/main/components/segmenter/src/word.rs#L159)にある通り、日本語と中国語は LSTM モデルのデータを持っていないため、辞書ベースで分割するようです。辞書ベースでの分割の実装は [components/segmenter/src/complex/dictionary.rs](https://github.com/unicode-org/icu4x/blob/main/components/segmenter/src/complex/dictionary.rs#L36) で実装されています。こちらに使われる辞書は[intl/icu_segmenter_data/data/macros](https://searchfox.org/mozilla-central/source/intl/icu_segmenter_data/data/macros)に生成されており、元を辿ると [icu_datagen crate](https://crates.io/crates/icu_datagen) で ICU4C の BreakIterator で使われているものと同じ辞書データから生成されています。

https://github.com/unicode-org/icu4x/blob/main/docs/tutorials/data_management.md#2-generating-data

従って ICU4X における単語分割も ICU4C とほとんど変わらなさそうです。

### ICU での単語レベルの分割

結論としては IPA 辞書を利用してその頻度情報からある程度簡易的な単語の分割を行なっているようです。一方で他の形態素解析ツールのようにより文法を意識した解析やより特化したアルゴリズムは利用されておらず、辞書も常に新しく更新されているわけではないので、専用の形態素解析ツールと比べると精度や新しい語彙への対応で劣る部分がありそうです。

ちなみに辞書に関しては Intl.Segmenter でカスタム辞書を利用可能にする提案が行われてはいるものの、採用されるかはわからない状況です。

https://github.com/tc39/proposal-intl-segmenter/issues/133

## まとめ

- Intel.Segmenter の分割の仕組みはある程度実装に任されている。
- 主要ブラウザの JS ランタイムでは ICU の BreakIterator あるいは ICU4X の Segmenter を利用している。
- ICU の BreakIterator 　並びに ICU4X の Segmenter は IPA 辞書を利用したある程度簡易的な単語の分割を行なっている。

## 参考文献

この記事を書くにあたって参考にしたページです。

- ECMAScript
  - [18 Segmenter Objects](https://tc39.es/ecma402/#segmenter-objects)
- V8
  - [Checking out the V8 source code](https://v8.dev/docs/source-code)
  - [chromium/v8/v8/refs/heads/main (Google Git)](https://chromium.googlesource.com/v8/v8/+/refs/heads/main)
  - [V8 JavaScript Engine (github.com)](https://github.com/v8/v8)
- SpiderMonkey
  - [mozilla-central](https://searchfox.org/mozilla-central/source/js/src)
- JavaScriptCore
  - [Webkit JavaScriptCore](https://github.com/WebKit/WebKit/tree/main/Source/JavaScriptCore)
- ICU
  - [ICU Boundary Analysis / Break Rules](https://unicode-org.github.io/icu/userguide/boundaryanalysis/break-rules.html)
  - [ICU github.com](https://github.com/unicode-org/icu)
  - [ICU4X github.com](https://github.com/unicode-org/icu4x)
