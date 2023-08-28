---
title: "typescript-eslintで新しい構文をサポートする"
emoji: "🌲"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript", "eslint", "frontend"]
publication_name: "cybozu_frontend"
published: false
---

## 新しい構文がやってくる

ご存知の通り、JavaScript の標準仕様である ECMAScript では毎年新しい記法や構文が提案され、採択されています。もちろん JavaScript のスーパーセットである TypeScript もその例外ではありません。
TypeScript では基本的に ECMAScript Proposal の Stage3 になった仕様から順次サポートするという方針があります。最近であれば v5.0 に入った `Decorators` や v5.2 で導入される `using Declarations`(`Explicit Resource Management`)などが該当します。

ここまでのことは普段 TypeScript を利用している方であれば知っている方も多いでしょう。しかし実際に TypeScript で新しい構文がサポートされた後、typescript-eslint 並びに typescript-eslint の提供する機能に依存するツールチェインがどのように新しい構文に対応するのかまで追っている人は少ないかと思います。

この記事では typescript-eslint で `using Declarations` をサポートしたケースをもとに、新しい構文にどう対応していくのかを紹介します。

## JS/TS の AST バリエーション

eslint などの lint ツールや prettier などの format ツールはコードをチェックしたり、書き換えるために AST(Abstract Syntax Tree: 抽象構文木) というデータ構造を介してソースコードを解析・変更することが一般的[^1]です。

JavaScript の場合、この AST に対して標準化団体が策定するような厳密な標準仕様はありません。そのため JavaScript のコードから生成される AST には parser などによってバリエーションや系統があります。数ある AST の中でメジャーな AST の仕様が [ESTree](https://github.com/estree/estree) です。Babel や ESLint、typescript-eslint などの JavaScript ツールチェインの多くはこの ESTree 互換の[^2] AST 並びにそのような AST を生成する parser を利用しています。

この辺りの話は以下のような記事が詳しいのでぜひ併せて読んでみてください。

- [ESTree とは](https://sosukesuzuki.dev/advent/2022/06/)
- [[2015-02] 最近の JavaScript AST 標準化の動き](https://efcl.info/2015/02/26/recent-js-ast/)

また、ESTree にある程度互換のある AST を生成する parser に関して以下の Zenn scrap でまとまっているので、興味があれば見てみると面白いでしょう。

https://zenn.dev/sosukesuzuki/scraps/fa4d48f9098d66

### typescript-eslint の場合

さて、TypeScript の Compiler API も AST を生成する機能を持っていますが、この AST は ESTree 互換ではありません。一方 typescript-eslint では ESLint 同様、ESTree に互換性のある AST を利用したいという事情があります。そのため typescript-eslint では TypeScript の Compiler API が生成した AST をさらに ESTree 互換の形に変換することで ESTree 互換の AST を生成しています。これを行なっているのが typescript-eslint のリポジトリ内にある
[`@typescript-eslint/typescript-estree`](https://github.com/typescript-eslint/typescript-eslint/tree/main/packages/typescript-estree)です。

![ESTree互換のASTを生成するまで](/images/tsEslintNewSyntax/astConvert.png)

また変換された結果得られる ESTree 互換の AST の型情報は `@typescript-eslint/ast-spec` にあります。

https://github.com/typescript-eslint/typescript-eslint/tree/main/packages/ast-spec

## `using Declarations` 構文についておさらい

今回の記事では `using Declarations` 構文への対応を例に話すので、この構文についても軽くおさらいをしておきましょう。すでに知っている方などはスキップしてもらって大丈夫です。

`using Declarations` は [`Explicit Resource Management`](https://github.com/tc39/proposal-explicit-resource-management)と呼ばれる proposal に含まれる構文で、2023/08 現在 Stage3 の Proposal です。TypeScript としては v5.2 からサポートされた機能になります。

ここでは `Explicit Resource Management` 自体への詳しい解説は避けますが、簡単にいうと「リソースの確保と開放を、変数の初期化と破棄に紐付けるような挙動」をサポートする機能です。`using Declarations` 構文はこの中で明示的にリソースを確保・開放したいデータを代入する変数を宣言するとき利用されます。

### `using` と `await using`

`using Declarations` は `const` や `let` などと同じような使われ方をイメージするとわかりやすいでしょう。実際、以下の例のように変数の宣言をする際、変数名の前につけて利用します。

```typescript
using handle = open("./path/to/file.txt");
```

また派生として `await using` という記法もあります。こちらは非同期にリソースの確保・破棄をする必要がある場合に使われます。ただの `await` 演算子に似て見慣れないかもしれませんが、 `await using` で 1 まとまりです。

```typescript
await using handle = await open("./path/to/file.txt");
```

また `using Declarations` は `for-of` 文でも利用できます。これも `let` や `const` と同じですね。具体的には以下のような使い方になります。

```typescript
for (using x of iter){
  // ...
}
for (await using x of iter){
  // ...
}
```

### `using` 構文の制約

このように `const` や `let` と同じような記法で利用できる `using Declarations` ですが、`const` や `let` で許されていて、`using Declarations` では許されていない記法が多くあります。今回はその中でも次の項で説明する「typescript-eslint での対応」に関係しそうな制約を挙げます。

#### 必ず初期化しないといけない

以下の例のように初期化しないような記法は `using Declarations` では許可されていません。

```typescript
using x; // ❌
```

#### 分割代入はできない

`using Declarations` を利用する場合、分割代入は許可されません。以下の例はいずれも仕様上間違った記法になります。

```typescript
using {x,y} = hoge(); // ❌
using [x,y] = fuga(); // ❌
```

#### `for-in` では利用できない

`using Declarations` は `for-of` で利用可能ですが、`for-in` では利用できません。

#### typescript の `declare` 宣言と併用できない

これは ECMAScript の仕様としてではなく、TypeScript 独自の記法との兼ね合いですが、 `using Declarations` の前に `declare` 宣言をつけることはできません。例えば以下の例は TypeScript として間違った記法になります。

```typescript
declare using y = hoge(); // ❌
```

以上が `using Declarations` のおおまかな使用例と制約についてでした。これらの前提をもとに、typescript-eslint で `using Declarations` に対応する流れを次の項で説明します。

## typescript-eslint で新しい構文に対応する

ここからが記事の本題です。これまで説明した事情を踏まえると、typescript-eslint で新しい構文をサポートするには以下の対応が必要になります。

- typescript-eslint が定義する「ESTree 互換の AST」の型を更新する
- TypeScript Compiler API が生成する AST を「ESTree 互換の AST」に変換するロジックを更新する
  - 場合によっては構文的に間違った AST でエラーを投げるなどの処理も追加する
- AST の変更を検知するスナップショットテストを追加・更新する

それぞれ順番に `using Declarations` 対応を例に取りながら見ていきましょう。

### ESTree 互換の AST の型を更新する

まずは `@typescript-eslint/typescript-estree` が変換し、実際に typescript-eslint のルールが利用する「ESTree 互換の AST」の型定義を更新する必要があります。

#### ESTree で定義された AST の仕様を読み解く

このために ESTree で `using Declarations` 利用時の AST がどう定義されているのか見にいきましょう。ESTree では ECMAScript Proposal の stage3 になった仕様から[専用のディレクトリ](https://github.com/estree/estree/tree/master/stage3)を切って AST の仕様を公開しています。今回対応する `Explicit Resource Management` に関しても仕様が公開されています。

https://github.com/estree/estree/blob/master/stage3/explicit-resource-management.md

この仕様によれば、`VariableDeclaration` Node の kind プロパティに新しく `"using"` と `"await using"` を足す形で AST の定義を変更するようです。ただ注意書きにあるように AST に対して以下の２点の新しい制約が課されています。

##### 1: If `kind` is `"using"` or `"await using"`, for every declarator `d` of `declarations`, `d.id` must be an Identifier.

`VariableDeclaration` Node には `declarations` と呼ばれるプロパティがあり、この中には宣言された変数に関する情報を持つ Node (`VariableDeclarator`) の配列が入ります。(`const a,b;` のように一度に２つ変数宣言ができたりするので配列になっています。)

```
interface VariableDeclaration extends Declaration {
    type: "VariableDeclaration";
    declarations: [ VariableDeclarator ]; // ←ここ!
    kind: "var";
}
```

ESTree の仕様では、`"using"` や `"await using"` に限り、 `declarations` に入っている各 Node(`VariableDeclarator`) の id プロパティが必ず `Identifier` Node であることを求めています。

`id` プロパティには変数名などの情報が入った AST Node が格納されます。`Identifier` Node は `const a = 1;` のような一般的な変数宣言で現れる Node です。一方 `let` や `const` では `Identifier` Node 以外にも　`BindingPattern(ArrayPattern | ObjectPattern)` Node が `id` プロパティに入る Node として許されています。`BindingPattern(ArrayPattern | ObjectPattern)` は `const {a} = hoge;` や `const [a] = hoge;` のような分割代入の構文で `id` プロパティに入る Node です。

つまり、最初の制約を言い換えると、「`using Declarations` では分割代入ができない」と言うことになります。これは前の項で説明した通りですね。

##### 2: If the variable declaration is the `left` of a ForOfStatement, `d.init` must be `null`, otherwise `d.init` must be an Expression.

これは for-of 文中の `using Declarations` では `declarations` に入っている各 Node(`VariableDeclarator`) の `init` プロパティが必ず `null` になり、それ以外の時は `null` にならないと言う制約です。

`VariableDeclarator` Node の `init` プロパティには名前の通り「変数を初期化した時の情報」が入ります。よって `init` プロパティが `null` ということは宣言時の初期化がされていないことを意味します。

つまり、この制約を言い換えると以下のようになります。

- for-of 文の中では `using Declarations` の初期化はされない。
- for-of 文の中以外で `using Declarations` を使う際は必ず初期化しないといけない。

これらの制約も前の項で説明した仕様上許されない記法や構文の通りです。

このように見ると 1,2 どちらの制約も仕様上許されない構文の制約を AST に落とし込んだものと言えるでしょう。

#### 実際に型定義を変更する

typescript-eslint のルールが利用する「ESTree 互換の AST」の型は [`@typescript-eslint/ast-spec`](https://github.com/typescript-eslint/typescript-eslint/tree/main/packages/ast-spec)にあるので、こちらの `src/declaration/VariableDeclaration/spec.ts` を起点に編集していきます。

一番のポイントは `VariableDeclaration` Node の `kind` プロパティに今までの `let | const | var` に加えて `using | await using` が加わるという変更です。

ただ、今回は上記の制約を型で表現することにより強固な AST の型絞り込みを行いたいというレビューをもらいました。そのため最終的に `VariableDeclaration` Node を以下の 3 つの AST Node の型の Union にすることとなりました。

- `LetOrConstOrVarDeclaration` : 名前の通り今までの let/const/var による `VariableDeclaration` Node の型です。`kind` プロパティは `let | const | var`。
- `UsingInForOfDeclaration` : for-of 文で利用された時の `using Declarations` の Node の型です。`kind` プロパティは `using | await using`。
- `UsingInNormalContextDeclaration` : for-of 文以外で利用された時の `using Declarations` の Node の型です。 `kind` プロパティは `using | await using`。

さらに、`UsingInForOfDeclaration` 型と `UsingInNormalContextDeclaration` にはそれぞれ対応する `Declarator` Node(`declarations` に入っている各 Node)の型を用意して、上記の制約を型に盛り込んでいます。また `UsingInForOfDeclaration` 型と `UsingInNormalContextDeclaration` 型はともに `declare` プロパティが false になっているのもポイントです。(typescript の `declare` 宣言と併用できない制約からです。)

https://github.com/typescript-eslint/typescript-eslint/blob/23ac49944e4f4067f89123fddd4a80c629270b4c/packages/ast-spec/src/declaration/VariableDeclaration/spec.ts

for-of 文の AST node (`ForOfStatement`) に関しても、`ForOfInitialiser` 部分が `LetOrConstOrVarDeclaration | UsingInForOfDeclaration` のみを受け付けるように(= `UsingInNormalContextDeclaration` を受け付けないように) 変更しました。これで、for-of における `using Declarations` の利用パターンでも AST をより厳密に絞り込めます。

### 「ESTree 互換の AST」に変換するロジックを更新する

次に、TypeScript Compiler API が生成する AST を「ESTree 互換の AST」に変換するロジックを更新する必要があります。この変換ロジックは[`@typescript-eslint/typescript-estree`](https://github.com/typescript-eslint/typescript-eslint/tree/main/packages/typescript-estree)内の [`src/convert.ts`](https://ygithub.com/typescript-eslint/typescript-eslint/blob/23ac49944e4f4067f89123fddd4a80c629270b4c/packages/typescript-estree/src/convert.ts) が担っています。

この `src/convert.ts` は 3000 行以上ある非常に大きなファイルですが、やっていることはそこまで複雑ではありません。基本的に以下の処理を繰り返しているだけです。

1. TypeScript Compiler API が生成する AST の Node type ごとに処理を分岐する。
2. 分岐されたそれぞれの case で ESTree 互換の AST に変換するロジックを書く。
3. AST が入れ子になっている場合(終端 Node 以外は基本入れ子になっている)は再起的にこの変換ロジックを呼ぶ(以降終端の Node に至るまで再起的に 1.が呼び出されていく...)

今回の例で言えば `VariableDeclaration` Node を生成する部分に処理を追加する必要がありそうです。具体的に[`VariableDeclaration` Node を生成する部分](https://github.com/typescript-eslint/typescript-eslint/blob/c11e05c97ef80d36fd194ac15952c339c1612b9e/packages/typescript-estree/src/convert.ts#L984-L991)を見ると、`getDeclarationKind` というメソッドで `kind` プロパティを判定しているようです。

#### `let | const | var | using | await using` を判定する

`VariableDeclaration` Node の `kind`(`let` か `const` か...) を判別する際は基本的に以下の 2 つの「ビット論理積」を取ることで判別できます。

- TypeScript Compiler API が生成する AST の `VariableDeclaration` Node にある flags プロパティの値
- TypeScript で Node ごとに決まっている Flag の値

これらのフラグは 2 の冪乗で設定されている(=2 進数で表現した時に特定の決まった桁だけ 1 になるように設定されている)ため、同じ種類の Flag 同士のビット論理積だけ 0 より大きくなり、それ以外は 0 となるようになっています。

```js
const LetFlag = 0b001;
const ConstFlag = 0b010;

LetFlag & LetFlag; // → 0b001(1)
ConstFlag & ConstFlag; // → 0b010 (2)
ConstFlag & LetFlag; // → 0b000 (0)
```

しかしながら `await using` だけは [`Const` の Flag と `Using` の Flag のビット論理和をとると言う定義](https://github.com/microsoft/TypeScript/blob/c3c5abb3a7d960de6e2fb75bc2a74fb90d9109b6/src/compiler/types.ts#L787)になっており、単純にビット論理積をとると `await using` の Flag 以外でも 0 より大きくなってしまいます。(例えば `Const` のフラグ `0b0010` と　`await using` の Flag `0b0110` のビット論理積は `0b010` になってしまいます。違う Flag なのに...)

このため今回の実装では[先に `await using` の可能性を排除した上で、`Const` や `Using` の Flag とビット論理積を取る](https://github.com/typescript-eslint/typescript-eslint/blob/c11e05c97ef80d36fd194ac15952c339c1612b9e/packages/typescript-estree/src/node-utils.ts#L344-L353)ような実装にする必要がありました。

#### 仕様上許されていない構文の AST が生成されそうになった時になるべくエラーを投げる。

TypeScript Compiler API は仕様上許されていないような構文に関してもある程度寛容にパースできるようになっています。そのため場合によっては変換処理中で不正な AST に対してエラーを投げる必要が出てきます。

具体的には今回 4 つのケースでエラーを投げるように実装しました。

- [`using Declarations` で初期化していることのチェック](https://github.com/typescript-eslint/typescript-eslint/blob/c11e05c97ef80d36fd194ac15952c339c1612b9e/packages/typescript-estree/src/convert.ts#L1001-L1006)
- [`using Declarations` で分割代入をしていないことのチェック](https://github.com/typescript-eslint/typescript-eslint/blob/c11e05c97ef80d36fd194ac15952c339c1612b9e/packages/typescript-estree/src/convert.ts#L1007-L1012)
- [`for-of` 文で `using Declarations` を使った時に初期化されていないことのチェック](https://github.com/typescript-eslint/typescript-eslint/blob/c11e05c97ef80d36fd194ac15952c339c1612b9e/packages/typescript-estree/src/convert.ts#L1037C10-L1042)
- [`for-of` 文内の `using Declarations` で分割代入をしていないことのチェック](https://github.com/typescript-eslint/typescript-eslint/blob/c11e05c97ef80d36fd194ac15952c339c1612b9e/packages/typescript-estree/src/convert.ts#L1043-L1048)

### スナップショットテストを追加する。

最後に生成する AST が変わっていないことを保証するためにスナップショットテストを追加します。スナップショットテストのケースは以下のように `@typescript-eslint/ast-spec` 内の AST 型定義と並列で置かれている `fixtures` 配下に書かれています。

```
├── fixtures
│   ├── _error_              // 異常系のテスト
│   │   ├── foo              // テストケース
│   │   │   ├── fixture.ts   // テストしたい構文の書かれたファイル
│   │   │   └── snapshots    // エラーのスナップショット(自動生成)
│   │   └── ...
│   ├── bar                  // 正常系のテストケース
│   │   ├── fixture.ts       // テストしたい構文の書かれたファイル
│   │   └── snapshots        // 生成されたASTのスナップショット(自動生成)
│   └── ...
└── spec.ts                  // ASTの型定義

```

今回であれば [`src/declaration/VariableDeclaration/fixtures/`](https://github.com/typescript-eslint/typescript-eslint/tree/c11e05c97ef80d36fd194ac15952c339c1612b9e/packages/ast-spec/src/declaration/VariableDeclaration/fixtures) にテストケースを足していきます。具体的には以下の 6 点のテストケースを追加しました。

- 正常系
  - [`await using` で複数の変数の初期化を行うパターン](https://github.com/typescript-eslint/typescript-eslint/blob/c11e05c97ef80d36fd194ac15952c339c1612b9e/packages/ast-spec/src/declaration/VariableDeclaration/fixtures/await-using-multiple-declarations/fixture.ts)
  - [`using` で複数の変数の初期化を行うパターン](https://github.com/typescript-eslint/typescript-eslint/blob/c11e05c97ef80d36fd194ac15952c339c1612b9e/packages/ast-spec/src/declaration/VariableDeclaration/fixtures/using-multiple-declarations/fixture.ts)
  - [`await using` で初期化を行うパターン](https://github.com/typescript-eslint/typescript-eslint/blob/c11e05c97ef80d36fd194ac15952c339c1612b9e/packages/ast-spec/src/declaration/VariableDeclaration/fixtures/await-using-with-value/fixture.ts)
  - [`using` で初期化を行うパターン](https://github.com/typescript-eslint/typescript-eslint/blob/c11e05c97ef80d36fd194ac15952c339c1612b9e/packages/ast-spec/src/declaration/VariableDeclaration/fixtures/using-with-value/fixture.ts)
- 異常系
  - [`using` で分割代入をしているパターン](https://github.com/typescript-eslint/typescript-eslint/blob/c11e05c97ef80d36fd194ac15952c339c1612b9e/packages/ast-spec/src/declaration/VariableDeclaration/fixtures/_error_/object-binding-patterns-in-using/fixture.ts)
  - [`using` で初期化をしていないパターン](https://github.com/typescript-eslint/typescript-eslint/blob/c11e05c97ef80d36fd194ac15952c339c1612b9e/packages/ast-spec/src/declaration/VariableDeclaration/fixtures/_error_/missing-value-in-using/fixture.ts)

また for-of 文での動作もある程度保証したかったので、[for-of 文の AST 型定義がある場所](https://github.com/typescript-eslint/typescript-eslint/tree/c11e05c97ef80d36fd194ac15952c339c1612b9e/packages/ast-spec/src/statement/ForOfStatement/)にも正常系のテストケースを１件追加しています。

### `using Declarations` 対応完了

以上の３つの変更で基本的に新しい構文へのサポートは完了です。

今回は ESTree ですでに決まっている仕様通りだったことや既存の構文・AST の拡張扱いだったのでそこまで大きな変更にはなりませんでした。全く持って新しい AST を定義するような構文対応だともう少し修正部分や影響範囲も大きそうです。

また、ESTree はあくまで **JavaScript の AST 標準**であり、 TypeScript 独自の部分は定義されていません。そのため新しい TypeScript 独自の記法に対応する際などは、主に Babel との互換性を気にしながら[^3] AST を拡張する必要があります。

## 終わりに・謝辞

これで、typescript-eslint 並びに `@typescript-eslint/typescript-estree` を利用しているツールは `using Declarations` をパースできるようになりました。今回は自分が対応する機会を得ましたが、typescript-eslint では新しい構文が TypeScript に入るたびにこのような対応を行なっています。(しかも RC が出てから対応する方針なので結構忙しい時もあるのではと個人的には心配になります。) この記事を元にツールの裏側の仕組みやツールを支えている方々の活動にも目をむけてくれる人が増えれば嬉しい限りです。

またこの `using Declarations` 対応は [sousuke 氏](https://twitter.com/__sosukesuzuki)の勧めとサポートのおかげで行うことができました。本当にありがとうございます。

[^1]: Rome など、AST ではなく CST（Concrete Syntax Tree: 具象構文木）を採用しているツールもあるので必ず AST を使っているとは限りません。
[^2]: あくまでも厳密な標準仕様ではなく、ツール間の互換性のために守られている約束に近いので parser ごとで微妙に違ったり、独自の拡張をしていることが多いです。
[^3]: 現状明確な合意などではなく、ある程度揃えようと協調して努力している形です。ちなみに typescript-eslint 側では、Babel の AST と差分があったテストを snapshot に記録しています。
