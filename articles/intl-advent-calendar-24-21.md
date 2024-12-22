---
title: "Intl.Segmenter v2 Proposal(#21)"
emoji: "↩️"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: true
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 21 日目の記事です。

今回は Intl.Segmenter に新しいオプションを追加する Intl.Segmenter v2 proposal について解説します。

## Intl.Segmenter v2 Proposal

Intl.Segmenter は前回[20 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-20)で解説した通り、ロケールを考慮しつつ、指定された粒度で文字列を分割するための API です。Intl.Segmenter v2 proposal はこの Intl.Segmenter に新しいオプションを追加する Stage1 の提案です。

https://github.com/tc39/proposal-intl-segmenter-v2

### モチベーション

Intl.Segmenter v2 proposal では新しく文字列を分割する粒度として「行」をサポートしようとしています。

適切な行での分割はテキストのレンダリングにおいて重要な要素であり、特に以下のような場所でニーズがあります。

- Canvas や SVG などの内部で Text をレンダリングする場合
- CLI やコード埋め込みなどにおける適切なレンダリング : 例) github のソース diff 表示
- HTML の文脈外のテキストをレンダリングする場合 : 例) PDF の生成
- WebGL や WebXR でのテキストレンダリング

このような処理に対して都度、行分割のためのライブラリをバンドルすることは容量の観点からしても現実的ではありません。(実際多くのロケールをサポートしようとするとかなりの容量になるようです。)

そのため Intl.Segmenter の 1 機能(=JS の標準機能)として行分割をサポートすることが求められています。

::: message

実は Chromium にはすでにこれに近い機能(オプション)を有する `Intl.v8BreakIterator` という API が存在してます。ただしこの API は V8 が実装する Chromium 独自の API で非標準なものです。

```ts
const itr = new Intl.v8BreakIterator(["ja-jp"], { type: "line" });
```

さらに、この機能は 2024 年 11 月に非推奨とすることが提案され、ゆくゆくは Chromium 並びに v8 からも削除されることが予定されています。

https://chromestatus.com/feature/4973414383878144

:::

### 提案されている実際の API

Intl.Segmenter v2 proposal では API の変更として以下の２点が提案されています。

- `granularity` オプションに `"line"` を追加
- `lineBreakStyle` オプションを新たに追加

`granularity` オプションに `"line"` を指定することで分割単位を「行」に指定できます。

```ts
const segmenter = new Intl.Segmenter("ja-JP", {
  granularity: "line",
});
```

またこの時 Intl.Segmenter インスタンスの `segment()` メソッドが返すセグメントのデータには `isHardBreak` というプロパティが追加され、そのセグメントがハードブレーク(強制改行)であるかどうかを示すようになります。

```ts
{
  "segment": "こんにちは",
  "index": 0,
  "input": "こんにちは、世界。",
  "isHardBreak": false // 🆕
}
```

これは `granularity` オプションに `"word"` を指定した場合の `isWordLike` プロパティと同様の機能です。

また `lineBreakStyle` オプションは `"strict"`, `"normal"`, `"loose"` の 3 つの値を取ります。これは CSS Text Level 3 で中国語・日本語・韓国語（CJK）テキストの改行方法を指定する `line-break` プロパティに指定できる値と同様に以下のような挙動をすることが提案されています。

- `"strict"`：最も厳格な改行規則を用いてテキストを改行する。
  - 長い行によく使われる。
- `"normal"`：最も一般的な改行規則を用いてテキストを改行する。
  - 書籍や文書によく使われる。
- `"loose"`：最も制約の少ない改行規則
  - 新聞などの短い行で使われる

従って `"strict"` > `"normal"` > `"loose"` の順で本当に改行して良さそうなところでしか「改行できる」とマークしなくなります。

詳しい改行のルールは Unicode Standard Annex #14 Unicode Line Breaking Algorithm に従うようなのでそちらも参照すると良いでしょう。

https://unicode.org/reports/tr14/

## まとめと次回予告

この記事では行単位での文字列分割をサポートする Intl.Segmenter v2 proposal についてそのモチベーションや提案されている API などを解説しました。

次回[22 日目]()では言語、地域、文字体系の表示名の一貫した翻訳をサポートする Intl.DisplayNames について解説します。

## 参考文献

- Intl Segmenter v2 Proposal
  - https://github.com/tc39/proposal-intl-segmenter-v2
- Unicode Standard Annex #14 Unicode Line Breaking Algorithm
  - https://unicode.org/reports/tr14/
