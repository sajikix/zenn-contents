---
title: "MessageFormat v2 の記法をみてみよう(#24)"
emoji: "🔎"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: true
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 24 日目の記事です。

今回は Intl.MessageFormat Proposal が採用する ICU MessageFormat v2 について解説します。

## ICU MessageFormat v2

前回[23 日目](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-23)の記事で解説した通り Intl.MessageFormat は ICU MessageFormat v2 の準拠した形で仕様の策定を進めています。そこでここでは ICU MessageFormat v2 の記法について解説し、利用する際のイメージを持ってもらいます。

:::message
ICU MessageFormat v2 は現在 Final Candidate として提案されており、今後仕様が変更される可能性があります。
:::

### ICU MessageFormat v2 の基本

ICU MessageFormat v2 は元々存在していた ICU MessageFormat の課題を解決すべく多くの新しい記法などが追加された文言のフォーマット仕様です。具体的な課題などに関しては詳しくは下記の記事が参考になります。

https://github.com/unicode-org/message-format-wg/blob/main/docs/why_mf_next.md

ICU MessageFormat v2 は現在 ICU の MessageFormat Working Group によって策定されており、それらの仕様やドキュメント、issue の管理などは以下の github 上で行われています。

https://github.com/unicode-org/message-format-wg

ICU MessageFormat は XML 形式などではなく、独自の記法を持って、プレイスホルダの埋め込みや条件分岐、数値や日付のフォーマットなど多くの機能を持っています。

```
Hello, {name}!
```

## ICU MessageFormat v2 の記法を知る

では実際に ICU MessageFormat v2 の記法を見て行きましょう。

### ICU MessageFormat v2 における「Message」の分類

ICU MessageFormat v2 には多くの記法が存在しますが、利用される記法やフォーマットの構造によっては大きく「Simple Message」と「Complex Message」の２つに分類されます。

Simple Message はそのまま表示されるテキストで構成されるようなものや、表示するテキストの一部に `{$ name}` のような形で変数を埋め込むようなものです。例えば以下のようなメッセージはそれぞれ Simple Message に分類されます。

```
Hello, world!
Hello, {$ name}!
```

一方 Complex Message はそもそも全体が条件分岐などの MessageFormatV2 の構文で書かれていて、その中に「`{{ }}` で囲われた表示したいテキスト部分」が存在しているような記法たちのことです。以下は 1 つの Complex Message の例です。(詳しい記法は後で説明するので一旦これで 1 つの「Message」であることに注目してください。)

```
.local $userId= {$id : integer}
{{user: {$userId} has logged in}}
```

Complex Message を初めて見る方は「これで１つのメッセージ？」「メッセージというよりプログラムでは？」と思うかもしれませんが、MessageFormatV2 はこのような記法をサポートすることで、より複雑な条件分岐や変数の埋め込みをサポートしています。

少し雑な例えではありますが、JavaScript でいうところの、

- Template Literal 内で色々頑張ってる文字列 → Simple Message
- 最終的文字列を返す関数や if 文などの式 → Complex Message

のようなイメージを持ってもらえるとわかりやすいかもしれません。

#### 記法による空白や改行の扱いの違い

MessageFormat v2 においては読んでいる部分が MessageFormat v2 の構文部分か、表示するためのテキストなのかを区別すする必要があります。これは両者で以下のように空白や改行の扱いなどが変わるためです。

- MessageFormat v2 の構文内 : 改行や空白は構文を壊さない限り無視できる
  - つまり見やすく複数行に整形してもいいし、スペースを入れてもいい
- 表示するテキスト : 改行や空白はそのまま表示される
  - でないと改行や空白を表現できなくなるため

Simple Message の場合 `{}` で囲まれる部分だけが MessageFormat v2 の構文でそれ以外は表示するテキストとして扱われます。一方 Complex Message の場合は全体が MessageFormat v2 の構文であり、その中で `{{ }}` で囲まれた部分が表示するテキストとして扱われます。

```
// Simple Message = {} で囲まれた部分は「構文」なのでスペースとか入れて良い
Hello, {$name}!
// ↑と同じ
Hello, { $name }!

// Complex Message = 全体が構文なので{{}}内以外は好き改行とかして良い
.local $userId= {$id : integer}
{{user: {$userId} has logged in}}
// ↑と同じ
.local $userId= {$id : integer} {{user: {$userId} has logged in}}
```

### Simple Message に分類される記法

まずは Simple Message に分類される記法について解説します。

#### 単純な文字列と変数の埋め込み

特に MessageFormat v2 の構文を使わずに、文字列を書いた場合、それはそのまま表示されます。

```
Hello, world!
```

また `{$[変数名]}` のような形でフォーマット時に後から渡す変数を文字列内に埋め込むことができます。

```
Hello,{$userName}.
```

#### 構文内でのエスケープ・文字列リテラル

MessageFormat v2 の SimpleMessage では `{}` で囲まれた部分が構文として扱われてしまいます。このような `{}` の中などで文字列を表したい場合、`||` で囲います。

```
Hello, {|Saji|}.
```

ただ基本的に文字列をエスケープするくらいなら普通に表示する文字として構文の外で書いたほうが楽なのであまり使うことはないかもしれません。(続く構文で使う場合があるので先に説明しましたが。)

#### 関数の呼び出し

MessageFormat v2 では `{$[変数 or リテラル] :[関数名]}` のような形で変数の値を関数に渡しつつ呼び出した結果を埋め込むことができます。

```
Today is {$date :datetime}.
```

また後ろに `[引数名]=[引数値]` のような形で追加の引数を渡すこともできます。

```
Today is {$date :datetime weekday=long}.
```

関数はフォーマット時に登録した関数を呼び出すこともできますし、デフォルトで提供される(ように仕様で決まっている)関数を呼び出すこともできます。デフォルトで利用できる関数は以下のようなものがあります。

- `number` : 数値のフォーマット (Intl.NumberFormat と似た感じ)
- `Integer` : 整数のグルーピング(単数・複数形のため)と整数のフォーマット
- `datetime` : 日付/時刻の値をフォーマットする
- `date` / time : 日付/時刻の値を別々でフォーマット
- `string` : 文字列への変換

ちなみに `{$[変数 or リテラル] :[関数名]}` と説明した通り、関数の前に書けるのは変数だけでなく数値や文字列といったリテラルも使えます。`{}` 内では、数値はそのまま、文字列は先ほど説明したように `||` で囲むことでリテラルとして扱うことができ、関数に渡すことができます。

```
The total was {0.5 :number style=percent}.
Define a constant {|maxNum| :toUpperSnake}.
```

また変数名部分を省略した場合、ただの関数呼び出しとして扱われます。

```
You are browsing from {$ :platform}.
```

#### マークアップ

MessageFormat v2 では HTML などのマークアップを埋め込むこともできます。具体的には `{#[開始タグ]}[中身]{/[終了タグ]}` の形で表現できます。

```
{#button}Submit{/button} // <button>Submit</button>
```

また、`{#[開始タグ] [属性名]=[属性値]}[中身]{/[終了タグ]}` のように属性も指定できますし、`{#[開始タグ] /}` のように終了タグの省略もできます。

```
{#img alt=|Cancel| /} // <img alt=“cancel”>
```

ただし、これはあくまでも MessageFormat v2 側は「ここの部分がマークアップでこういう情報が入っている」という情報を持つだけで、実際この情報をどう書式化するのかはフォーマットする側の処理に任されています。一般的には HTML 文字列にフォーマットするでしょうが、ブラウザで実行される JS ランタイムであればこの情報から直接 DOM を作成しても良いわけです。

### Complex Message に分類される記法

ここからは Complex Message に分類される記法について解説します。そのため `{{ }}` で囲まれた部分以外は基本 MessageFormat v2 の構文として扱われ、`{{ }}` で囲まれた部分が表示するテキストとして扱われます。

#### 変数の宣言

MessageFormat v2 では文言の先頭部分で変数を宣言できます。変数の宣言には `input-definition` と `local-definition` の 2 つがあります。

`input-definition` は外部から渡される変数を保持しておくもので、`.input {$[変数名]}` のように宣言します。

```
.input {$name}
{{Hello, {$name}! This is {$name}'s profile.}}
```

`local-definition` では `.input {$[変数名] : [関数名]}` のように変数の値を関数に渡しつつその結果も保持できます。

```
.input {$name : toUpper}
{{Hello, {$name}!}}
```

`local-definition` は `input-definition` と違い、式の解決された値を変数として保持する構文です。 `.local $[変数名]= [式]` のように宣言します。

```
.local $userId= {$id : integer}
{{user: {$userId} has logged in}}
```

もちろん `input-definition` で受け取った変数を使って `local-definition` でさらに変数を宣言できます。

```
.input {$id : integer}
.local $userName= {$id : getUserName}
{{Hello, {$userName}! , your id is {$id}.}}
```

上の例では `id` という名前で変数を受け取り、それを `getUserName` という関数に渡してその結果を `userName` という変数に保持しています。

#### パターンマッチ

MessageFormat v2 では渡された式や宣言された変数の値によってメッセージを出し分けるパターンマッチの記法があります。具体的には `.match {[式]} [パターン1] {{[メッセージ]}} [パターン2] {{[メッセージ]}}` のような形で記述します。

以下の例では `count` という変数の値によって「0 の場合」「1 の場合」「それ以外の場合」でメッセージを出し分けています。(`"*"` は全てをキャッチするという意味を持つ特殊な文字)

```
.match {$count :integer}
0   {{You have no notifications.}}
one {{You have {$count} notification.}}
*   {{You have {$count} notifications.}}
```

この時、`.match` の後ろに指定できるのは以下のいずれかになります。

- 変数に関数を適用した式
  - 例: `{$count :integer}`
- 単純な関数の呼び出し
  - 例: `{$ :platform}`
- 関数を適用した `input-definition` と `local-definition`
  - 例: `.input {$count :integer}`

さらに、パターンマッチの構文にはマッチする対象を複数にして全てのパターンを列挙する文法もあります。例えば変数として渡された `numLikes` と `numShares` それぞれの値によってメッセージを出し分ける場合、以下のように記述できます。

```
.input {$numLikes :integer}
.input {$numShares :integer}
.match {$numLikes} {$numShares} // match条件が2つ
0   0   {{Your item has no likes and has not been shared.}}
0   one {{Your item has no likes and has been shared …}}
0   *   {{Your item has no likes and has been shared …}}
// … 以下全パターン列挙(この例だと3x3の9パターン)
```

このようにかなり複雑な文言の条件分岐もパターンマッチで記述できます。

### MessageFormat v2 の記法まとめ

このように、MessageFormat v2 にはプレイスホルダーのような機能以外にも、関数呼び出しやマークアップの記述、変数宣言やパターンマッチなど多くの機能があります。またここで紹介したもの以外にも「Private-Use Annotation」や「Namespace」といった機能もあるので詳しくは是非以下の記法の仕様書を参照してください。

https://github.com/unicode-org/message-format-wg/blob/main/spec/syntax.md

## まとめと次回予告

今回は Intl.MessageFormat で利用される MessageFormat v2 のその記法について細かく紹介しました。
次回[25 日目]()はこの MessageFormat v2 の文言を Intl 側でより効率的に管理・フォーマットするための提案、`Intl.MessageFormat.parseResource()` について解説し、Intl.MessageFormat が標準化された時のメリットや懸念点についても考えていきます。
