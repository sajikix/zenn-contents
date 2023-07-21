---
title: "typescript-eslit v6アップデートガイド"
emoji: "⏫"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript", "eslint", "frontend"]
publication_name: "cybozu_frontend"
published: false
---

## typescript-eslit v6 リリース！

2023/07/10 に typescript-eslint の v6 がリリースされました。

https://typescript-eslint.io/blog/announcing-typescript-eslint-v6/

メジャーバージョンアップということで多くの BreakingChange があるのですが、その中でも`Reworked Configuration Names`と呼ばれる変更は利用者に大きな影響を与えそうです。

`Reworked Configuration Names`はざっくり言うと「config に書く`recommended`のようなルールセットの名前や枠組みが変わるよ」という変更です。

ルールセットの名前だけでなく、含まれるルールや分類自体に変更があるので、設定ファイルを v5 のままアップデートしてしまうと意図したルールセットと異なる設定になってしまいます。

ここでは上記の記事にある変更点を紹介しつつ、なるべく既存の設定のままアップデートする方法を紹介します。

## `Reworked Configuration Names` についての解説

### v5 までのルールセットについておさらい

元々 v5 で用意されているルールセットは下記の３つでした。

- `recommended`: 推奨ルールのうち、TypeScript の型情報が必要がないルール群
- `recommended-requiring-type-checking`: 推奨されるが TypeScript の型情報が必要なルール群
- `strict`: `recommended`より厳しく、場合によっては意見の分かれるルール群

これらのルールセットは互いに重複がなかったので、「`recommended`を利用しつつ、必要に応じて`recommended-requiring-type-checking`を追加する」といったような運用を行うことが多かったと思います。例えば`recommended`と`recommended-requiring-type-checking`を有効にする場合は以下のように設定ファイルに記載していました。

```js
module.exports = {
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:@typescript-eslint/recommended-requiring-type-checking",
  ],
  plugins: ["@typescript-eslint"],
  // ...略
};
```

### v6 からのルールセット

一方 v6 ではこれらのルールセットに対して大きく 3 つの変更点が入っています。

1. ルールセット自体を`Functional rule`と`Stylistic rule`という 2 区分に分けた
2. それぞれのルールセットに「 TypeScript の型情報がいらないルール群」と 「TypeScript の型情報が必要なルール群」の２バージョンを用意した
3. 2 における「TypeScript の型情報が必要なルール群」は「 TypeScript の型情報がいらないルール群」を含むようになった

1 つ目は複数のルールセットを「コードの正しさの保証やバグを防ぐ目的の`Functional rule`」と「機能よりも構文の一貫性などを保証する`Stylistic rule`」に分けるというものです。`Functional rule`には`recommended`,`recommended-type-checked`,`strict`,`strict-type-checked`の 4 つのルールセットが、`Stylistic rule`には`stylistic`,`stylistic-type-checked`の 2 つのルールセットが含まれます。

2 つ目は全てのルールセットに「 TypeScript の型情報がいらないルール群」と 「TypeScript の型情報が必要なルール群」の２バージョンをセットで用意したというものです。「TypeScript の型情報が必要なルール群」のは必ずルールセット名の後ろに`-type-checked`が付くようになります。(例 : `recommended`と`recommended-type-checked`)

:::message
元記事にもある通り、v5 以前は`strict`ルールセットに「 TypeScript の型情報がいらないルール」と 「TypeScript の型情報が必要なルール」が混じっており、これを不満に感じていたユーザーが一定数いたようです。
:::

３つ目は v5 と違い「TypeScript の型情報が必要なルール群」が同じ名前の「 TypeScript の型情報がいらないルール群」を含むようになったというものです。例えば、`recommended-type-checked`を指定する場合、`recommended`は指定しなくてもよくなります。次の図のように考えるとわかりやすいですね。

![](/images/tsEslintV6Guide/ruleSetChange.png)

これらアップデートを踏まえると、元記事にある通り以下のような６つのルールセットの構成になります。

- Functional rule configurations
  - `recommended` / `recommended-type-checked` : 推奨ルール
  - `strict` / `strict-type-checked` : より厳しく、場合によっては意見の分かれるルール
- Stylistic rule configurations
  - `stylistic` / `stylistic-type-checked` : 統一された構文使用のためのルール

また、`strict`は全ての`recommended`ルールを、`strict-type-checked`は全ての`recommended-type-checked`ルールを含むようになりました。

よって v6 におけるルールセットの関係を図にすると以下のようになります。

![](/images/tsEslintV6Guide/allRules.png)

このようなルールセット区分の見直しに加え、今までルールセットに入っていなかったルールが各ルールセットに追加されたり、削除されたりしています。そのため、v5 のルールセット利用時と同じルールを適用したい場合は独自に設定する必要があります。

### (閑話)なぜ「TypeScript の型情報が必要なルール」は分けられているのか

ここまで読んでいる方の中には「TypeScript の型情報が必要なルール」をわざわざ切り出すのはなぜか疑問に思う方もいると思います。

一番大きな理由は、「TypeScript の型情報が必要なルールはパフォーマンス面で若干不利になるため、非常に大きなプロジェクトなどを運用するユーザーが選択できるようにしたい」というものでしょう。

https://typescript-eslint.io/linting/typed-linting/#how-is-performance

そもそも、typescript-eslint の全てのルールで「TypeScript の型情報」が必要になるわけではありません。もう少し厳密にいうと必ずしも「TypeScript Compiler API の Type Checker」を利用する必要はないということです。実際、typescript-eslint のドキュメントでも Type Checker を利用するルールは`Typed Rules`という形で分けて解説されてます。

https://typescript-eslint.io/developers/custom-rules#typed-rules

ただこれだけどイメージが湧きにくいので具体例を挙げてみましょう。

例えば、`Hoge[]`と`Array<Hoge>`を使い分けるようにする[`array-type`](https://typescript-eslint.io/rules/array-type)のようなルールはわざわざ TypeScript Compiler API の Type Checker を利用せずとも、AST の情報だけで判断・修正することができます。実際[`array-type`ルールの実装](https://github.com/typescript-eslint/typescript-eslint/blob/main/packages/eslint-plugin/src/rules/array-type.ts)では Type Checker は利用されていません。

一方、Union(`|`)や Intersection(`&`)のメンバー重複を禁止する[`no-duplicate-type-constituents`](https://typescript-eslint.io/rules/no-duplicate-type-constituents)のようなルールは Type Checker を利用しています。この例では「型レベルの重複(= 同じ型を意味するか)」を判断する部分で [Type Checker を利用](https://github.com/typescript-eslint/typescript-eslint/blob/main/packages/eslint-plugin/src/rules/no-duplicate-type-constituents.ts#L127-L129)しています。

このように TypeScript Compiler API の Type Checker を利用することで TypeScript の型情報をフルに使ったルールが作成できる一方、TypeScript Compiler API へのアクセスは AST の解析と比べてパフォーマンス面で不利になります。実際自分が`no-duplicate-type-constituents`ルールを追加した際はレビューで[その点を指摘してもらい](https://github.com/typescript-eslint/typescript-eslint/pull/5728#discussion_r1083520606)、TypeScript Compiler API へのアクセスをなるべく遅延し、取得結果をキャッシュするように改善しました。

もちろん大規模なプロジェクトでなければそこまで問題になることは少ないでしょうし、typescript-eslint 側は「型情報が必要なルール」を有効にすることを強く推奨しています。型情報を必要とするルールのパフォーマンスについては、どうしても lint が遅いと感じた時に考慮するくらいで良いのかも知れません。

> We strongly recommend you do use type-aware linting, but the above information is included so that you can make your own, informed decision.
> (https://typescript-eslint.io/linting/typed-linting#how-is-performance)

## 実際にありそうなケースでのアップデート

ここでは v5 で標準的なルールセットを利用してる場合に、**とりあえず v5 と同じルールのまま** v6 にアップデートする方法を紹介します。
追加でルールを設定・無効化している場合、紹介するルール設定と重複やバッティングする可能性があるのでうまくマージしてください。

また個別のルールの変更状況は以下に乗っています。細かく設定を見ながら変えたい人はこちらを参照すると良いでしょう。
https://github.com/typescript-eslint/typescript-eslint/discussions/6014

### v5 で `recommended` のみ設定していた場合

`recommended`ルールの一部は`stylistic`に移動したため v5 の`recommended`ルールと同じ設定にする場合、基本的には v6 の`recommended`ルールにルールを追加する必要があります。具体的には以下のようにルールを変更すると良いででしょう。

```js
module.exports = {
  extends: ["plugin:@typescript-eslint/recommended"],
  // ... (略)
  rules: {
    // v6 で recommended から削除されたものを有効化
    "no-extra-semi": "off",
    "@typescript-eslint/no-extra-semi": "error",
    // v6 で strict に移動したルールを有効化
    "@typescript-eslint/no-non-null-assertion": "warn",
    // v6 で stylistic に移動したルールを有効化
    "no-empty-function": "off",
    "@typescript-eslint/no-empty-function": "error",
    "@typescript-eslint/adjacent-overload-signatures": "error",
    "@typescript-eslint/no-empty-interface": "error",
    "@typescript-eslint/no-inferrable-types": "error",
    "@typescript-eslint/prefer-namespace-keyword": "error",
    // v6 で recommended に追加されたルールを無効化
    "@typescript-eslint/no-duplicate-enum-values": "off",
    "@typescript-eslint/no-unsafe-declaration-merging": "off",
  },
};
```

また v6 で`recommended`と`stylistic`を有効にし、 v5 の`recommended`になかったルール群を削るという方法もあります。こちらの場合は以下のような設定にすれば v5 の`recommended`ルールと同じになります。

```js
module.exports = {
  extends: [
    "plugin:@typescript-eslint/recommended",
    "plugin:@typescript-eslint/stylistic",
  ],
  // ... (略)
  rules: {
    // v6 で recommended から削除されたものを有効化
    "no-extra-semi": "off",
    "@typescript-eslint/no-extra-semi": "error",
    // v6 で strict に移動したルールを有効化
    "@typescript-eslint/no-non-null-assertion": "warn",
    // v6 で recommended に追加されたルールを無効化
    "@typescript-eslint/no-duplicate-enum-values": "off",
    "@typescript-eslint/no-unsafe-declaration-merging": "off",
    // stylistic を有効にしたため v5 の recommended にないルールを無効化
    "@typescript-eslint/array-type": "off",
    "@typescript-eslint/ban-tslint-comment": "off",
    "@typescript-eslint/class-literal-property-style": "off",
    "@typescript-eslint/consistent-generic-constructors": "off",
    "@typescript-eslint/consistent-indexed-object-style": "off",
    "@typescript-eslint/consistent-type-assertions": "off",
    "@typescript-eslint/consistent-type-definitions": "off",
    "@typescript-eslint/no-confusing-non-null-assertion": "off",
    "@typescript-eslint/prefer-for-of": "off",
    "@typescript-eslint/prefer-function-type": "off",
  },
};
```

typescript-eslint 側としては`recommended`と`stylistic`を両方有効にすることを想定していそう(元記事では v5 の`recommended`と v6 の`recommended`・`stylistic`の差分が取り上げられていた)なので後者の設定にする方が良いかもしれません。

### v5 で `recommended`と`recommended-requiring-type-checking`を指定していた場合

v5 で `recommended`と　`recommended-requiring-type-checking`を指定していた場合、v6 の`recommended-type-checked`増えたルールを一旦`"off"`にするのが良いでしょう。前述した通り、`recommended`は`recommended-type-checked`に含まれるので`recommended-type-checked`だけ指定します。

```js
module.exports = {
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended-type-checked",
  ],
  // ...略
  rules: {
    // v6 で recommended から削除されたものを有効化
    "no-extra-semi": "off",
    "@typescript-eslint/no-extra-semi": "error",
    // v6 で strict に移動したルールを有効化
    "@typescript-eslint/no-non-null-assertion": "warn",
    // v6 で stylistic に移動したルールを有効化
    "no-empty-function": "off",
    "@typescript-eslint/no-empty-function": "error",
    "@typescript-eslint/adjacent-overload-signatures": "error",
    "@typescript-eslint/no-empty-interface": "error",
    "@typescript-eslint/no-inferrable-types": "error",
    "@typescript-eslint/prefer-namespace-keyword": "error",
    // v6 で recommended に追加されたルールを無効化
    "@typescript-eslint/no-duplicate-enum-values": "off",
    "@typescript-eslint/no-unsafe-declaration-merging": "off",
    // v6 で recommended-type-checked に追加されたルールを無効化
    "@typescript-eslint/no-base-to-string": "off",
    "@typescript-eslint/no-duplicate-type-constituents": "off",
    "@typescript-eslint/no-redundant-type-constituents": "off",
    "@typescript-eslint/no-unsafe-enum-comparison": "off",
  },
};
```

ちなみに v5 の`recommended-requiring-type-checking`にあったルールで v6 の`stylistic-type-checked`に移動したルールは存在しないので、「v5 と同じルールを再現する」という目的だけであれば`stylistic-type-checked`を有効にしなくても問題ありません。

## まとめ

この記事では typescript-eslint の v6 で大きな変更となる`Reworked Configuration Names`についてそのアップデート内容と実際に想定されるケースでのアップデート例を紹介しました。

今回は影響を最小限にできるよう「v5 と同じルールのままのルールでとりあえず v6 にあげる」という点を重視してアップデート例を紹介しました。そのため、この設定のままだと新しく入ったルールなどは利用できず、現在の typescript-eslint のルールを十分に使いこなせていない可能性があります。 v6 へのアップデートを機に適用しているルール自体の見直もしてみてはいかがでしょうか。
