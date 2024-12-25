---
title: "parseResource() で便利になるIntl.MessageFormatとその未来(#25)"
emoji: "🔜"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: false
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 25 日目(最終日)の記事です。

今回は MessageFormat v2 の文言リソースファイルを Intl 側で利用しやすくする提案、`Intl.MessageFormat.parseResource()` Proposal について解説します。その上で Intl.MessageFormat が標準化された時のメリットや懸念点についても考えていきます。

## `Intl.MessageFormat.parseResource()`

[23 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-23)で解説した通り、Intl.MessageFormat は ICU MessageFormat v2 に準拠した形で仕様の策定を進めています。しかし ICU MessageFormat v2 はあくまで「１つ 1 つの文言のフォーマット」の仕様です。また Intl.MessageFormat 自体も「1 つの文言フォーマットを読み込んで書式化する」機能を提供するもので、複数の文言を管理したりまとめて書式化するための仕組みは提供していません。

### `Intl.MessageFormat.parseResource()` のモチベーション

web アプリケーションなどで実際に国際化対応をする場合、各言語ごとに何十、何百といった文言を用意し一意な key などで管理することが一般的です。また文言管理ライブラもこのような文言のリソースファイルを読み込んで key から適切な文言を取得する機能を提供しています。(詳しくは[23 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-23)を参照してください。)

Intl.MessageFormat.parseResource() proposal はこのような一般的な文言管理ライブラリにある複数の文言を管理するための仕組みを Intl.MessageFormat で提供する提案です。

https://github.com/tc39/proposal-intl-message-resource

### `Intl.MessageFormat.parseResource()` の API

例えば以下のような MessageFormat v2 の形式で書かれた文言リソースファイルがあるとします。

```ini
greeting = {Hello {$name}!}

new_notifications =
  match {$count}
  when 0   {You have no new notifications}
  when one {You have {$count} new notification}
  when *   {You have {$count} new notifications}
```

Intl.MessageFormat.parseResource() はこのような文言リソースファイルを読み込む静的メソッドです。引数としては文言リソースファイルの文字列と、その文言リソースファイルのロケールを指定します。

```ts
const resource = ... // 上記のような文言リソースファイルを読み込む
const messageResource = Intl.MessageFormat.parseResource(resource,"en");
```

`parseResource()` の返り値は MessageResource というオブジェクトで、key に文言 key を持ち、値に Intl.MessageFormat インスタンスや MessageResource 自体を持つ `Map` オブジェクトです。

```ts
type MessageResource = Map<string, Intl.MessageFormat | MessageResource>;
```

そのため、文言の key で `get` することで、その文言に対応する Intl.MessageFormat インスタンスを取得できます。

```ts
const greetingInstance = messageResource.get("greeting");
```

もちろん取得した Intl.MessageFormat インスタンスは通常の Intl.MessageFormat インスタンスと同様に `format()` メソッドを使ってメッセージを得ることができます。

```ts
greetingInstance.format({ name: "Saji" }); // "Hello Saji!"
```

このように `Intl.MessageFormat.parseResource()` が実装されることで、Intl.MessageFormat に足りていなかった「文言リソースファイルを一括で読み込みパースする」機能が提供されることになります。まだまだ Stage1 の提案ではありますが、実際の利便性を考えると Intl.MessageFormat とセットで欲しい機能だと思います。

## Intl.MessageFormat が使える未来

最後に MessageFormat v2 や Intl.MessageFormat で使えるようになることでどのようなメリットがあるのか、またどのように使えそうか考えてみます。(筆者の私見が多分に含まれます。)

### MessageFormat v2 の嬉しいところ

MessageFormat v2 は上記の通り、他の文言フォーマットと比べても豊富な機能を持っています。特に関数の呼び出し機能などは多言語対応やフォーマットの柔軟性を高めるために非常に便利だと感じています。

またマークアップ機能などはかなり web で利用することを踏まえた機能となっていそうで、今まで誰しもが悩むであろう「文言に HTML を埋め込む」問題を一定解決してくれるのではないかという期待があります。

さらに、どこまで複雑な構文を利用するかはチームによりそうですが、パターンマッチなどは言語ごとにその文法に合わせた文言の出しわけが行えるという点で非常に強力な機能です。

### Intl で使えるメリット

何より Intl で MessageFormat v2 が使えることにより、ライブラリやフレームワークに頼らない文言の管理とフォーマットが標準機能で可能になります。また現状ライブラリやプラットオームごとに独自に決めている文言のフォーマットが MessageFormat v2 にまとまっていくことで、周辺ツールやサービスが発達する可能性もあると思っています。

しかも MessageFormat v2 で標準的に使える関数などは Intl 側の実装を利用する方になるでしょうから、それまで Intl の機能を部分的に使っていた人でも移行しやすい形になるのではないかと思います。

このような点から、Intl で利用できるようになった暁には、web アプリケーションにおける文言フォーマットの第１選択肢として選ばれるようになるのではと期待しています。

### 考えないといけないこと

このような強力な MessageFormat v2 ですが、その機能の豊富さゆえに複雑な構文への理解や複雑化させないための規約などが必要になるかもしれません。(筆者も「大量のパターンマッチして何十行にもなってる文言」はみたくはないです...)

また文言内で受け取れる変数や利用できる関数は、Intl 側(JS 側)で指定したろ実装したりする必要があります。そのため専門のライターさんや翻訳者さんが文言を管理している場合、今まで以上にエンジニア(実装サイド)との連携が必要になることが予想されます。そういった意味でも、利用している関数や変数が適切に JS 側で渡されているかのチェックツールや Lint ルールなどは今後現れてくれるとありがたいですね。

## まとめ

今回の記事では Intl 側で文言リソースを利用しやすくする `Intl.MessageFormat.parseResource()` Proposal について解説しました。また Intl.MessageFormat が標準化されるメリットや懸念点についても考えていきました。

この記事をもって、[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555) は終了です。最後までお読みいただきありがとうございました！
