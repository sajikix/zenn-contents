---
title: "Intl.Collator(#19)"
emoji: "🔜"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: false
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 19 日目の記事です。

今回は言語を考慮した文字列の比較を可能にする `Intl.Collator` について解説します。

## ロケールによる文字列の比較の違い

どのアプリケーションでも文字列の比較はよく使われる操作です。例えば、ユーザーが入力した文字列をソートする際や、検索する際には文字列の比較が必要になります。

しかし、そもそも文字列の比較方法は 1 つでなく、ロケールによって異なることがあります。例えば、英語の場合だけをとっても以下のような方法が考えられます。

- 大文字・小文字を区別するかどうか。区別するならどちらが先になるのか。
- 数字部分を数値として比較するか文字列として比較するか。
- 句読点を無視するかどうか。

さらにロケールによっては記号付きの文字を考慮する必要があったり、そもそも文字の並び順がいわゆるアルファベット順とは異なることがあります。例えば、ドイツ語では `ä`, `ö`, `ü` といったウムラート付きの文字や ß がアルファベットとして扱われます。これらもロケールに応じて適切に比較して並び替える必要があります。

## `Intl.Collator`

`Intl.Collator` はこのようなロケールに応じた文字列の比較をするための機能です。

### Intl.Collator の基本的な使い方

他の Intl のコンストラクタプロパティ同様、第１引数にロケール(ロケール識別子 or Intl.Locale オブジェクト)を第２引数にフォーマットのオプションを渡して初期化することで、Intl.Collator インスタンスを生成できます。

```ts
const collator = new Intl.Collator("en-US", {
  // オプションを指定
});
```

生成した Intl.Collator インスタンスは `compare()` メソッドが生えており(詳しくは記述)、このメソッドに比較したい文字列をそれぞれ渡すことで、ロケールに合わせた文字列の比較が可能です。

```ts
collator.compare("a", "b"); // -1
```

### メソッド

Intl.ListFormat インスタンスには以下の 2 つのメソッドがあります。

- `compare()`
- `resolvedOptions()`(他の Intl オブジェクトと同様なので省略)

#### `compare()`

`compare()` メソッドは２つの文字列を比較し、そのロケールにおいてソートしたときにどちらが前に来るべきかを判別するメソッドです。返り値は `-1`,`0`,`1` のいずれかで、それぞれ以下のような意味を持ちます。

- `-1`: 第 1 引数が第 2 引数よりも前に来るべき
- `0`: 第 1 引数が第 2 引数の順番は変わらない
- `1`: 第 1 引数が第 2 引数よりも後に来るべき

```ts
// a,b,c の順に並べるのがロケールにおいて正しい順番なので...
new Intl.Collator("en-US").compare("a", "b"); // -1
new Intl.Collator("en-US").compare("a", "a"); // 0
new Intl.Collator("en-US").compare("b", "a"); // 1
```

このルールは Array.prototype.sort() などにおける比較関数と同じ挙動です。

`compare()` メソッドの引数に文字列以外の値を指定したり、何も値を指定しなかったりした場合、それらの値は全て文字列に変換されてから比較されます。(例えば、`null` は `"null"` に変換される)

```ts
new Intl.Collator("en-US").compare(undefined, null); // 1
// ↑ は以下と同じ意味になる
new Intl.Collator("en-US").compare("undefined", "null");
```

### `Intl.Collator` のオプション

Intl.Collator の初期化で指定できるオプションは以下の 6 つです。(全コンストラクタプロパティ共通の `localeMatcher` を除く)。

- `usage`
- `collation`
- `numeric`
- `caseFirst`
- `sensitivity`
- `ignorePunctuation`

それぞれのオプションについて詳しく見ていきましょう。

#### `usage` オプション

`usage` オプションは文字列の比較の目的を指定するオプションです。指定可能な値は　`"sort"` と `"search"` の２つです。

このようなオプションが用意されているのは並び替えと検索(文字列の一致判断)では最適な比較のアルゴリズムが微妙に変わるためです。言語によってはあるアルファベットと記号のついたアルファベットに対して、「並び替えの際は広く同じ(or 近い)文字として扱いたいが、検索の際は全く違う文字として区別したい」といった場合があるようです。(チェコ語では、`'a'` と `'á'` などは[その例らしい](https://unicode.org/reports/tr35/#UnicodeCollationIdentifier)です。)

デフォルトは `"sort"` で、文字列の並び替えを目的とした比較方法が適用されますが、もし文字列の一致検索を目的とする場合は `"search"` を指定した方がいいでしょう。

#### `collation` オプション

`collation` オプションは文字列比較に使用するパターンを個別に指定するオプションです。Intl.Collator はロケールを加味するだけでなく、このオプションで明示的特定のロケール向けの比較方法を指定できます。具体的には絵文字の並び替えに対応するための `emoji` や中国語向けのピンイン順序に対応する `pinyin` 、漢字の部首の画数順に並べ替える `unihan` などが指定できます。

```ts
new Intl.Collator("ja-JP", { collation: "unihan" }).compare("魚", "草"); //1
// 魚の部首は「さかな(11画)」で草の部首は「くさかんむり(３画)」なので
```

#### `numeric` オプション

`numeric` オプションは文字列の比較において数値部分を数値として比較するか文字列として比較するかを指定するオプションです。真偽値を取り、`false` の場合は「文字列」として比較し、`true` だと「数値の大小」として比較します。

```ts
new Intl.Collator("en-US", { numeric: false }).compare("a11", "a2"); // -1
// 文字列として比較すると２文字が「1」の a11 が a2 よりも前に来る
new Intl.Collator("en-US", { numeric: true }).compare("a11", "a2"); // 1
// 数値として比較すると 2 < 11 なので a2 が a11 よりも前に来る
```

#### `caseFirst` オプション

`caseFirst` オプションは文字列の並べ替え時に大文字小文字どちらを先とするかを指定するオプションです。`"upper"`, `"lower"`, `"false"` のどれかを取り、`"upper"` の場合は大文字が先に、`"lower"` の場合は小文字が先に、`"false"` のロケールのデフォルトの順序を保持します。

```ts
new Intl.Collator("en-US", { caseFirst: "upper" }).compare("a", "A"); // 1
new Intl.Collator("en-US", { caseFirst: "lower" }).compare("a", "A"); // -1
```

#### `sensitivity` オプション

`sensitivity` オプションは文字列の比較においてどの程度の違いを無視するかを指定するオプションです。指定可能な値は `"base"`, `"accent"`, `"case"`, `"variant"` の 4 つです。それぞれアクセント記号(その他発音記号含む)と大文字小文字の区別をするかどうかで以下のような挙動の違いがあります。

|             | アクセント記号 | 大文字小文字 |
| ----------- | -------------- | ------------ |
| `"base"`    | 区別しない     | 区別しない   |
| `"accent"`  | 区別する       | 区別しない   |
| `"case"`    | 区別しない     | 区別する     |
| `"variant"` | 区別する       | 区別する     |

#### `ignorePunctuation` オプション

`ignorePunctuation` オプションは文字列の比較において句読点を無視するかどうかを指定するオプションです。真偽値を取り、`false` の場合は句読点を考慮し、`true` だと句読点を無視します。

```ts
new Intl.Collator("en-US").compare("a.c", "ab"); // -1
new Intl.Collator("en-US", { ignorePunctuation: true }).compare("a.c", "ab"); // 1
```

デフォルト値はタイ語のみ `true` で、他のロケールでは `false` です。

### ユースケース

`Intl.Collator` の代表的なユースケースとしては、文字列の一致検索と文字列のソートが挙げられます。

#### 文字列の一致検索

Intl.Collator は上記のオプションで説明した通り、大文字小文字やアクセント記号の違いなど、微妙な表記の揺れを加味した一致を判断できます。これにより、いわゆる初歩的な曖昧検索のような挙動を実現できます。

```ts
const looseMatch = (str1: string, str2: string) => {
  return (
    new Intl.Collator("en-US", {
      sensitivity: "base",
      usage: "search",
    }).compare(str1, str2) === 0
  );
};
```

またこの挙動は `sensitivity` オプションで切り替えられるので、ユーザー側で検索の厳密さを調整するような機能を提供する場合でも柔軟に対応できます。

#### 文字列のソート

`compare` メソッドの部分で説明した通り、Intl.Collator の `compare` メソッドは配列の `sort()` などが callback として受け取る比較関数と同じ挙動をします。つまり Intl.Collator の `compare` メソッドはそのまま配列の `sort()` メソッドや `toSotted()` メソッドなどに渡すことができます。

```ts
const fruits = ["Banana", "apple", "orange"];
fruits.toSorted(new Intl.Collator("en-US").compare);
// ["apple", "Banana", "orange"]
```

これはあらゆるリストにおいてロケールを加味しソートを行えるという点で非常に便利です。

## まとめと次回予告

この記事では言語を考慮した文字列の比較を可能にする `Intl.Collator` について基本的な使い方、API について解説しました。

次回[20 日目]()は単語や見た目の１文字、文単位で文字列を分割できる `Intl.Segmenter` について解説します。お楽しみに！

## 参考文献

- LDML: Part 1 Core, Unicode Collation Identifier
  - https://unicode.org/reports/tr35/#UnicodeCollationIdentifier
- Unicode Technical Standard #10 Unicode Collation Algorithm, Searching and Matching.
  - https://unicode.org/reports/tr10/#Searching
