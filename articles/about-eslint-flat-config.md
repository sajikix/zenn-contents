---
title: "仕組みと嬉しさから理解するeslint FlatConfig対応"
emoji: "👷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["JavaScript", "eslint", "frontend"]
publication_name: "cybozu_frontend"
published: false
---

:::message
この記事は、[CYBOZU SUMMER BLOG FES '24](https://cybozu.github.io/summer-blog-fes-2024/) (Frontend Stage) DAY 3 の記事です。
:::

## 重い腰を上げて FlatConfig 対応をした

ESLint が新しい設定形式として FlatConfig を導入してから随分と経ち、最新バージョンの v9 では FlatConfig がデフォルトになりました。一方で利用者の多い plugin でもなかなか対応が進まず、周りでは思ったよりも FlatConfig への移行が進んでいない印象を受けます。

とはいえ次のバージョンである v10 では FlatConfig しかサポートしないことが予定されており、今まで移行を見送ってきた方も「さすがにそろそろ移行するか...」と思っているのではないでしょうか。自身の所属チームで管理している ESLint の rule セット : [@cybozu/eslint-config](https://github.com/cybozu/eslint-config) でも遅ればせながら FlatConfig 対応を進めています。（現在はアルファ版で提供中です）

そんな `@cybozu/eslint-config` を FlatConfig に移行した際の知見を中心に記事を書いてもいいのですが、細かい移行方法については [公式の移行ガイド](https://eslint.org/docs/latest/use/configure/migration-guide) が詳しいですし、詳細な移行事例については今まで多くの記事が出ています。

そこでこの記事では少し趣向を変えて「FlatConfig に変わったことで何が嬉しいのか」や「ESLint エコシステム全体での FlatConfig 対応とはどんな作業なのか」といったもう少しコアな部分に焦点を当てて FlatConfig を見ていきます。これにより単なる利用者だけでなく、Sharable Configs を公開している人や自作の plugin を作った人の参考にもなればと思います。もちろん最後には `@cybozu/eslint-config` を FlatConfig に移行する中で得られた細かな知見も共有するので、「FlatConfig については理解してるぜ！」という方はその部分だけでも読んで参考にしてもらえると嬉しいです。

## FlatConfig とは何を改めて理解する

まずは FlatConfig について、大きくどのような変更があったのか、その変更がどのようなメリットをもたらすのかに注目して見ていきましょう。

### 旧 config からの大きな変更点

旧 Config（eslintrc）から FlatConfig への大きな変更点は、「override や extends という概念がなくなる」点と「rule や config を独自で解決せず、JavaScript のモジュール解決の仕組みに乗った」点の 2 つと言えるでしょう。

#### `override` や `extends` と言う概念がなくなる

FlatConfig は `override` や `extends` という概念をなくし、代わりに `configuration object` と呼ばれる各設定情報を要素とする配列で表現するようになりました。

```js
export default [
  // 配列の中身一つ一つがconfiguration object
  {
    files: ["src/**/*.js"],
    ignores: ["**/*.config.js", "!**/eslint.config.js"],
    rules: {
      semi: "error",
    },
  },
];
```

この `configuration object` は基本的に以下のように動作します。

- `configuration object` には対象ファイル、plugin、rule、ignore、その他の設定などが書ける
- 適応する rule が重複した場合、配列の後ろのものが優先される

そのため、`configuration object` として前段に配置すれば `extends` と同じような挙動を実現でき、対象ファイルを絞った上で配列の後段に配置すれば `override` と同じような挙動を再現できます。

```js
export default [
    pluginA.config.recommended, // extends と同じ
    {
        rules: {
            semi: "warn"
        }
    }
    {
    // override と同じ
        files: ["src/**/*.jsx"],
        rules: {
            semi: "error"
        }
    },
];
```

#### rule や config を独自で解決せず JS のモジュール解決の仕組みに乗った

今までは以下のように config 名や plugin 名を文字列で書くと、ESLint が独自に rule や config の実態を解決して適用していました。

```
{
  extends: [
    '@cybozu/eslint-config/presets/react-typescript-prettier',
    'plugin:import/recommended',
  ],
  plugins:['react']
}
```

しかし今後は明示的に **JavaScript のモジュール**として plugin や config を import する必要があります。具体的には以下のように config は import してそのまま配置し、plugin も import して `configuration object` の `plugins` に配置する必要があります。

```js
import examplePlugin from "eslint-plugin-example";
import exampleConfigRecommended from "eslint-plugin-x";
export default [
  exampleConfigRecommended, // configをextendsするのに相当
  {
    plugins: {
      example: examplePlugin, // pluginsに指定するのに相当
    },
    rules: {
      "example/rule1": "warn",
    },
  },
];
```

### ESLint 側の気持ちになってみる

ここまで FlatConfig で「どのように変わったか」を簡単に整理しました。実際、設定ファイルを FlatConfig 対応する**だけ**ならここまでの知識で十分そうです。しかし一歩踏み込んで「なぜ ESLint はこの形に変更したのか」を考えると、FlatConfig にすることの嬉しさやどうしてこのような設計になったのかが理解しやすくなります。（ここで「嬉しい」のは必ずしもエンドユーザーとは限らないことに注意しましょう）

上記の 2 つの大きな変更点に共通するのはどちらも「**今まで独自に構築・実行してきた仕組みを JavaScript の標準的な仕組みに乗せ直す取り組み**」であったことです。つまり、「独自で築いてきた仕組み」が複雑化するにつれて、その管理や維持が大変になってきた背景が伺えます。では FlatConfig が解決したかった「独自で築いてきた仕組みの辛さ」はどのようなものなのでしょうか。

#### rule や設定を独自で解決する辛さ

旧 Config では次の例のように文字列で `extends` する config や plugin を指定していました。

```js
module.exports = {
  // ...
  root: true,
  extends: [
    "@cybozu/eslint-config/presets/react-typescript-prettier",
    "plugin:import/recommended",
  ],
  plugins: ["react"],
};
```

つまり、`'@cybozu/eslint-config'` や `'plugin:import/recommended'`、`'react'` といった「文字列から実際の config/plugin パスへの解決」動作は全て ESLint 側に任されていたことになります。特に `extends` では相対パス指定、plugin 内の config の指定、Sharable Configs パッケージの指定など複数の記法をサポートする必要がありました。

- plugins: `"react"` → `./node_modules/eslint-plugin-react/index.js`
- extends: `'plugin:import/recommended'` → `./node_modules/eslint-plugin-import/config/recommended.js`
- extends: `'@cybozu/eslint-config/presets/react-typescript-prettier'` → `./node_modules/@cybozu/eslint-config/presets/react-typescript-prettier.js`
- extends: `'../../.eslintrc'` → `'../../.eslintrc'`

また、ユーザーからしても lint を実行するまで `extends` や `plugins` で指定している文字列が有効なものなのか確認できません。（ESLint 独自の rule で解釈しているからそれはそう）

そのため、設定ファイルの機能を充実させるほど JavaScript のモジュール解決に似たリッチな仕組みを ESLint 側で開発・メンテする必要が出てきます。これは ESLint 側としてかなり大変だったと思われます。

一方、FlatConfig では config や plugin の解決が全て JavaScript のモジュール解決の仕組みに乗りました。ESLint は設定ファイルを**評価するだけ**で依存が全て解決された設定情報を得られるわけです。

#### extends と override が重なる辛さ

ESLint ではファイルごとに「このファイルに適用する rule はどれか」を設定ファイルから計算しマージする必要があります。しかし `extends` と `override` が多重に行われているような環境下では、この解決が非常に複雑になります。plugin や Sharable Configs を利用している場合、`extends` した config の中でも `extends` や `override` が行われていることは珍しくなく、場合によってはこれが 2 重 3 重になることもあります。また利用ケースこそ少ないですが、`override` の中でも他の config を `extends` できるため、場合によっては解決しなくてはいけない依存が広範囲に広がる可能性があります。

![extendsとoverrideによる依存](/images/aboutEslintFlatConfig/extendsDep.png)

このような状態でファイルごとの rule を計算する場合、それだけで多くの依存を解決する必要があり、加えてそれぞれの `config` で `ignore` や glob パターンが設定されていたりするとその計算はさらに複雑になります。またユーザー自体もどの config のどの部分が優先されてマージされるのか認識しづらくなってしまいます。

一方、FlatConfig では設定ファイルを評価した時点で config オブジェクトは**必ず 1 次元の配列**になります。この状態であれば単純に各 `configuration object` の `files` プロパティを見ながらフィルタし、後置された rule が勝つようマージするだけで適用すべき rule が計算できます。

![FlatConfigにおけるruleの計算イメージ](/images/aboutEslintFlatConfig/flatConfigCalc.png)

これは `extends` や `override` の依存解決を行なっていた旧 config と比べると遥かに効率化されており、JavaScript の処理として実装しやすい無理がない設計になっています。

#### 副次的に得られるメリット

rule や config を独自で解決しなくなったことで、副次的に得られるメリットがいくつかあります。その中でも大きいのがカスタム rule やカスタム plugin をパッケージ化することなく利用可能になったという点です。

今までは rule や config を独自で解決する都合上、カスタム rule を含む plugin には形態や命名規則に様々な制限がありました。その中でも「plugin は npm のパッケージとして読み込まれないといけない」という点は、自作 rule を作って簡単に試したい人にとってかなり不便な制約でした。

一方、FlatConfig 環境下ではモジュールとして import できる plugin オブジェクトを用意できれば、パッケージ化された plugin と相違なく読み込むことができます。パッケージ化しなくても簡単に自作 rule を試すことができるのは副次的なメリットの 1 つと言えます。

## 「FlatConfig 対応」の多層性

ここまで FlatConfig 自体の復習と FlatConfig になったことでどう嬉しいのかという話をしてきました。ここからは「FlatConfig 対応」の話をしていくのですが、そもそもどういう状態になれば「FlatConfig に対応した」と言えるのでしょうか？

ご存知の通り、ユーザーの設定ファイルを FlatConfig にするだけでは「FlatConfig に対応した」ことにはなりません。利用している plugin や Sharable Configs も「FlatConfig に対応」していないと完全に対応したとはいえなさそうです。では「FlatConfig に対応した Sharable Configs/plugin」かどうかはどうやって見分けるのでしょうか？[^1]

### ESLint ユーザーの行う「FlatConfig 対応」

利用している plugin や Sharable Configs が FlatConfig に対応している場合、ユーザーは既存の設定を FlatConfig の形式に書き換えるだけで対応が済みます。より具体的には次のような作業を進めるだけで良いでしょう。

- 利用している plugin や Sharable Configs を import する
- 既存の個別に設定した rule やその他項目を `configuration object` の方式にする
- extends している config は前段に、override している config は後段に配置する

また、利用している plugin や Sharable Configs が FlatConfig に対応していなくても [FlatCompat](https://github.com/eslint/eslintrc) を利用すれば暫定的な対応ではありますが移行できます。FlatCompat は config 全体の変換だけでなく `extends` や `env`、`plugin` だけといった部分的な変換にも対応しています。

### Sharable Configs パッケージに対する「FlatConfig 対応」

ではどうなれば Sharable Configs パッケージが FlatConfig に対応していると言えるのでしょうか。これは ESLint ユーザーの行う「FlatConfig 対応」とそこまで変わりません。結局のところ Sharable Configs パッケージが export しているのも config（ファイル/オブジェクト）なので、FlatConfig 形式のオブジェクト（`configuration object` の配列）が公開されているパッケージであれば FlatConfig に対応していると言えます。

ただ、FlatConfig 環境下では「FlatConfig 形式の config オブジェクトがモジュールとして import できさえすれば良い」ので、旧 config 時に[存在した以下のような制約](https://eslint.org/docs/v8.x/extend/shareable-configs#creating-a-shareable-config)に従わなくてよくなります。

- `eslint-config-` や `@scope/eslint-config` で始まるパッケージ名でないといけない
- package.json の `main` エントリポイントで config オブジェクトが export されていること

逆にこれらの制約は前述した「rule や config を独自で解決する辛さ」を低減するための規則だったとも言えます。

### plugin に対する「FlatConfig 対応」

最後に plugin の FlatConfig 対応とはどういったものなのか見ていきます。

そもそも ESLint の plugin は大きく「config を持つ plugin」と「config を持たない plugin」の 2 つに大別されます。plugin の実体は以下のように独自の `rule` や `configs`、`processors`、メタ情報などを持ったオブジェクトです。

```js
const plugin = {
    meta: {...},
    configs: {...},
    rules: {...},
    processors: {...}
};
```

基本的に plugin は 1 つ以上の rule を持ちますが、config は必須ではありません。rule の多い plugin などでは一括で rule を指定するために `"recommended"` や `"all"` といった config を持つことが一般的ですが、rule が少ない plugin であれば config を持たないものも少なくありません。FlatConfig はあくまで「`config` の新しい形式」ですから、その plugin が `config` を持っているかどうかで FlatConfig 対応の判断は変わります。また plugin の rule だけを利用していて config を利用しない場合、config が FlatConfig に対応しているかどうかは問題になりません。

#### plugin が config を持っていない場合 or plugin の config を利用していない場合

config を持っていない場合、基本的には FlatConfig で利用できることが多いです。ただし以下のような設定を行なっている場合、FlatConfig ではサポートしていないので注意が必要です。

- `plugin.processor` プロパティがあり、そのキーに拡張子を使っている
  - 拡張子をキーにする記法はサポートされなくなった
- `plugin.environments` を利用しており、plugin.config 内で plugin.environments と同じ設定を行なっていない
  - `plugin.environments` は読まれなくなるので Configs に移動し config の 1 つとして読み込むなどの方法が必要

詳しくは ESLint 公式の [Plugin Migration to Flat Config](https://eslint.org/docs/latest/extend/plugin-migration-flat-config) を参照してください。

#### plugin が config を持っていてその config を利用している場合

plugin が config を持っていてかつ利用している場合、その config はもちろん FlatConfig の形式であることが求められます。一方、Sharable Configs パッケージの時と同様、FlatConfig 形式の config は JavaScript のモジュールとして import できるようになっていればよく、必ずしも plugin オブジェクトの `configs` プロパティに配置しないといけないわけではありません。（逆に言えば旧 config 環境下では plugin オブジェクトの `configs` プロパティに config を配置しないと `plugin:hoge/recommended` のような名前で `extends` できませんでした。）

そのため FlatConfig に対応した plugin の config 提供方法は主に 2 種類あります。

1. plugin オブジェクトの `configs` プロパティに旧 config と合わせて FlatConfig 対応のものが列挙されている
2. plugin オブジェクトとは別に何らかの形で config オブジェクトを export している

1 の場合、公式では「旧 config のキーには `-legacy` を付けて、将来サポートされないことを明確にすること」が[推奨されています](https://eslint.org/docs/latest/extend/plugin-migration-flat-config#backwards-compatibility)。

```js
const plugin = {
  //...
  configs: {
    recommended : {...} // FlatConfig
    "recommended-legacy" : {...} // 旧 config
  },
};
```

しかし実際には `"flat/recommended"` のように FlatConfig であることを明示的に表す形で提供している plugin もあります。そのため、plugin に応じて README や実際の plugin オブジェクトを確認することが大切です。

2 の場合、plugin オブジェクト自体と違うパスで export している場合もありますし、plugin によっては config や config を生成するための関数を plugin 本体とは違うパッケージとして公開している場合もあります。

後者の代表的な例としては `typescript-eslint` が挙げられます。`typescript-eslint` の plugin の本体は `@typescript-eslint/eslint-plugin` という名前のパッケージですが、これとは別に `typescript-eslint` という名前のパッケージを公開しており、このパッケージから FlatConfig 形式の config や config を簡単に生成できる関数などを export しています。

```js
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.recommended
);
```

このように FlatConfig 環境下では config を生成・公開する方法がより柔軟になったと言えます。

## FlatConfig 対応で覚えておくと役に立ちそうなポイント

最後に、実際に FlatConfig 対応をする中でハマったポイントや困ったときに使えるかもしれない Tips を紹介します。

### files を指定しないときの挙動

FlatConfig の `configuration object` では `files` プロパティで明確にその config が対象とする範囲を指定する形に変更されました。ユーザー側として対象を明確に指定するのは可読性のためにも良さそうですが、Sharable Configs や plugin の config を作る側にとっては、ユーザーが実際どのようなファイル（js/ts/jsx/tsx...）やパスで実行するかわからず少し不便です。むしろこのような config 提供者としてはざっくりと「ユーザーの lint の対象になっている JS/TS 系のファイル全て」といった指定がしたくなります。

このような指定ができるよう、FlatConfig の `configuration object` で `files` プロパティを指定しない場合「他の config オブジェクトによってマッチされたすべてのファイルに適用される」という rule があります。

https://eslint.org/docs/latest/use/configure/configuration-files#configuration-objects

そのため基本的に Sharable Configs や plugin の config は `files` プロパティを指定しないで提供されることが多いです。これは「ユーザーは別の `configuration object` で対象にしたいパスや拡張子をちゃんと指定しているだろう」という前提に立つものですので、Sharable Configs や plugin の config だけを使っていたり、「特定のパスはこれらの config にしか任せていない」といった設定にする場合は注意しましょう。

### グローバルな ignore

旧 config では `ignorePatterns` や `.eslintignore` ファイルで lint 対象からファイルを除外できました。FlatConfig でも `configuration object` に `ignores` プロパティがあり、「`ignores` プロパティだけの `configuration object`」はグローバルな ignore 設定とみなされます。

```js
export default [
  {
    ignores: [".config/*"], // global な ignore
  },
  {
    ignores: [".mock/*"], // この configuration object だけ有効な ignore
    files: ["./src/"],
    rules: {....}
  },
];
```

またグローバルな ignore の設定として `["**/node_modules/", ".git/"]` はデフォルトでセットされています。

https://eslint.org/docs/latest/use/configure/ignore

### 共通の `settings`

旧 config では plugin などもアクセス可能な共通の設定情報を指定する `settings` プロパティがありました。FlatConfig においてはこれに変わるものとして `configuration object` に `settings` プロパティがあります。`configuration object` 内にあるのでそのスコープに閉じているように見えますが、`settings` プロパティで設定された情報は全ての `configuration object` で共有されるので注意しましょう。

```js
export default [
  {
    settings: {
      sharedData: "Hello", // このconfiguration object以外でも読める
    },
    plugins: {...},
    rules: {...},
  },
];
```

また `settings` プロパティ配下は全体で共有されてしまうため、plugin ごとに名前空間を作ることが慣例となっています。

https://eslint.org/docs/latest/use/configure/configuration-files#configuring-shared-settings

### 微妙に違う languageOption

旧 `parserOptions` は基本そのまま `languageOptions` 内に移動しましたが、`ecmaVersion` と `sourceType` だけは `parserOptions` から一段上の `languageOptions` 配下に移動しました。

```js
export default [
  {
    files: ["**/*.js", "**/*.mjs"],
    languageOptions: {
      ecmaVersion: 5, // 今はここに移動
      sourceType: "script", // 今はここに移動
      parserOptions: {
        requireConfigFile: false,
        //以前はこちらに書いていた
      },
    },
  },
];
```

「`parserOptions` はそのまま `languageOptions` 内に移動すればいいんだな」と思っている人は注意しましょう。

## まとめと次回予告

ここまで FlatConfig についてと FlatConfig 移行という言葉の多層性、実際に移行するときに役立つ Tips などを紹介してきました。この記事を読んで、FlatConfig と FlatConfig 移行についてより詳しくなり、移行するモチベーションや移行した config をより深く理解するきっかけなれば嬉しいです。

さて、この記事では FlatConfig 移行最後のチェックポイントである「何を持って移行できたとするのか」という点について触れていません。本来 lint が以前と同じように効いているのを確認するまでが FlatConfig 移行なはずです。というわけで「FlatConfig に移行できた」ことをどうやって確認するかやそのためのチェックツールを作った（ている）話を次回は書こうと思います。お楽しみに！

## 参考

- [ESLint's new config system, Part 1: Background](https://eslint.org/blog/2022/08/new-config-system-part-1/)
- [ESLint / Documentation](https://eslint.org/docs/latest/)

[^1]: 主要な Plugin であれば eslint のリポジトリにある [Tracking: Flat Config support](https://github.com/eslint/eslint/issues/18093) という issue からサポート状況を見ることができます。
