---
title: "仕組みと嬉しさから理解するeslint FlatConfig対応"
emoji: "👮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["JavaScript", "eslint", "frontend"]
publication_name: "cybozu_frontend"
published: true
---

:::message
この記事は、[CYBOZU SUMMER BLOG FES '24](https://cybozu.github.io/summer-blog-fes-2024/) (Frontend Stage) DAY 3 の記事です。
:::

## 重い腰を上げて FlatConfig 対応をした

eslint が新しい config 形式として FlatConfig を打ち出してから随分と経ち、最新バージョンの v9 では FlatConfig がデフォルトになりました。一方で利用者の多いプラグインでもなかなか対応が進まず、周りでは思ったよりも FlatConfig 移行が進んでいない印象を受けます。

とはいえ次のバージョンである v10 では FlatConfig しかサポートしないことが予定されており、今まで移行を見送ってきた方も「流石にそろそろ移行するか...」と思っているのではないでしょうか。自身の所属チームで管理している eslint の rule-set : [@cybozu/eslint-config](https://github.com/cybozu/eslint-config) でも遅ればせながら FlatConfig 対応を進めています。(現在は alpha で提供中です)

そんな `@cybozu/eslint-config` を FlatConfig に移行した際の知見を中心に記事を書いてもいいのですが、細かい移行方法については [公式の migration guide]() が詳しいですし、詳細な移行事例については今まで多くの記事が出ています。

そこでこの記事では少し趣向を変えて「FlatConfig に変わったことで何が嬉しいのか」や「eslint エコシステム全体での FlatConfig 対応とはどんな作業なのか」といったもう少しコアな部分に焦点を当てて FlatConfig を見ていきます。これにより単なる利用者だけでなく sharable config を公開している人や自作の Plugin を作った人の参考にもなればと思います。もちろん最後には `@cybozu/eslint-config` を FlatConfig に移行する中で得られた細かな知見も共有するので、「FlatConfig については理解してるぜ！」という方はその部分だけでも読んで参考にしてもらえると嬉しいです。

## FlatConfig とは何を改めて理解する

まずは FlatConfig について、大きくどのような変更があったのか、その変更がどのようなメリットをもたらすのかに注目して見ていきましょう。

### 古い形式の config からの大きな変更点

旧 config (eslintrc) から FlatConfig への大きな変更点は「override や extends と言う概念がなくなる」点と「rule や config を独自で解決せず JS のモジュール解決の仕組みに乗った」点の２つと言えるでしょう。

#### override や extends と言う概念がなくなる

FlatConfig は `override` や `extends` と言う概念をなくし、代わりに `configuration object` と呼ばれる各設定情報を要素とする配列で表現するようになりました。

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

この configuration object は基本的に以下のように動作します。

- configuration objects には対象ファイル,plugin,rule,ignore,その他設定などが書ける
- ルール設定が重複した場合配列の後ろのルールが勝つ

そのため config を `configuration objects` として前段に配置すれば `extends` と同じような挙動を実現できるし、対象 file を絞った上で `configuration object` を配列の後段に配置すれば `override` と同じような挙動を再現できます。

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

今までは以下のように config 名や plugin 名を文字列で書くと eslint が独自に rule や config の実態を解決して適応していました。

```
{
  extends: [
    '@cybozu/eslint-config/presets/react-typescript-prettier',
    'plugin:import/recommended',
  ],
  plugins:['react']
}
```

しかし今後は明示的に**JS のモジュールとして** plugin や config を import する必要があります。具体的には以下のように config は import してそのまま配置、plugin も import して configuration objects の `plugins` に配置する必要があります。

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

### eslint 側の気持ちになってみる

ここまで FlatConfig で「どのように変わったか」を簡単に整理しました。実際、設定ファイルを FlatConfig 対応する**だけ**ならここまでの知識で十分そうです。しかし一歩踏み込んで「何故 eslint はこの形に変更したのか」を考えると FlatConfig にすることの嬉しさやどうしてこのような設計になったのかが理解しやすくなります。(ここで「嬉しい」のは必ずしもエンドユーザーとは限らないことに注意しましょう)

上記の 2 つの大きな変更点に共通するのはどちらも「**今まで独自に構築・実行してきた仕組みを JS の標準的な仕組みに乗せ直す取り組み**」であったことです。つまり「独自で築いてきた仕組み」が複雑化するにつれてその管理や維持が大変になってきた背景が伺えます。では FlatConfig が解決したかった「独自で築いてきた仕組みの辛さ」はどのようなものなのでしょうか。

<!-- FlatConfig に移行する背景は eslint 公式が FlatConfig を最初に紹介した post:「[ESLint's new config system, Part 1: Background](https://eslint.org/blog/2022/08/new-config-system-part-1/)」に書かれています。ここでは 2 つの大きな変更点からこの post を振り返ることで改めて eslint の解決したかったことを紐解きます。-->

#### rule や config を独自で解決する辛さ

旧 config では次の例のように文字列で extends する config や plugin を指定していました。

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

つまり、`'@cybozu/eslint-config'` や `'plugin:import/recommended'`,`'react'` といった「文字列から実際の Config/Plugin path への解決」動作は全て **eslint 側に任されていた**ことになります。特に `extends` では相対パス指定、plugin 内の config の指定、shareable configuration package の指定など複数の記法をサポートする必要がありました。

- plugins:` "react"` → `./node_modules/eslint-plugin-react/index.js`
- extends : `'plugin:import/recommended'` → `./node_modules/eslint-plugin-import/config/recommended.js`
- extends : `'@cybozu/eslint-config/presets/react-typescript-prettier'` → `./node_modules/@cybozu/eslint-config/presets/react-typescript-prettier.js`
- extends : `'../../.eslintrc'` → `'../../.eslintrc'`

またユーザーからしても lint を実行するまで `extents` や `plugins` で指定してる文字列が有効なものなのか確認できません。(eslint 独自のルールで解釈してるからそれはそう)

そのため設定ファイルの機能を充実させるほど JS のモジュール解決に似たリッチな仕組みを eslint 側で開発・メンテする必要が出てきます。これは eslint 側としてかなり大変だったと思われます。

一方 FlatConfig では Config や Plugin の解決が全て JS のモジュール解決の仕組みに乗りました。eslint は config ファイルを**評価するだけ**で依存が全て解決された設定情報得られるわけです。

#### extends と override が重なる辛さ

eslint では file ごとに「この file に適応するルールはどれか」を設定ファイルから計算しマージする必要があります。しかし `extends` と `override` が多重に行われているような環境下ではこの解決が非常に複雑になります。Plugin や sharable config を利用している場合、`extends` した config の中でも `extends` や `override` が行われていることは珍しくなく、場合によってはこれが 2 重 3 重になることもあります。また利用ケースこそ少ないですが、 `override` の中でも他の config を `extends` できるため場合によっては解決しなくてはいけない依存が広範囲に広がる可能性があります。

![extendsとoverrideによる依存](/images/aboutEslintFlatConfig/extendsDep.png)

このような状態の config でファイルごとの rule 計算する場合、それだけで多くの依存を解決する必要があり、加えてそれぞれの `config` で `ignore` や glob pattern が設定されていたりするとその計算はさらに複雑になります。またユーザー自体もどの config のどの部分が優先的されてマージされるのか認識しづらくなってしまいます。

一方 FlatConfig では Config を評価した時点で Config オブジェクトは**必ず１次元の配列**になります。この状態であれば単純に各 `configuration object` の `files` プロパティを見ながら filter し、後置されたルールが勝つようマージするだけで適応すべきルールが計算できます。

![FlatConfigにおけるruleの計算イメージ](/images/aboutEslintFlatConfig/flatConfigCalc.png)

これは extends や override の依存解決を行なっていた古い形式の config と比べると遥かに効率化されており、JS の処理として実装しやすい無理がない設計になっています。

#### 副次的に得られるメリット

rule や config を独自で解決しなくなったことで、副次的に得られるメリットがいくつかあります。その中でも大きいのがカスタムルールやカスタム Plugin を Package 化することなく利用可能になったという点です。

今までは rule や config を独自で解決する都合上、カスタムルールを含む Plugin には形態や命名規則に様々な制限がありました。その中でも「Plugin は npm の package として読み込まれないといけない」という点は自作ルールを作って簡単に試したい人にとってかなり不便な制約でした。

一方 FlatConfig 環境下ではモジュールとして import できる Plugin オブジェクトを用意できれば Package 化された Plugin と相違なく読み込むことができます。

## 「FlatConfig 対応」の多層性

ここまで FlatConfig 自体の復習と FlatConfig になったことでどう嬉しいのかという話をしてきました。ここからは「FlatConfig 対応」の話をしていくのですが、そもそもどういう状態になれば「FlatConfig に対応した」と言えるのでしょうか？

ご存知の通りユーザーの configFile を FlatConfig にするだけでは「FlatConfig に対応した」ことにはなりません。利用している plugin や sharable config も「FlatConfig に対応」していないと完全に対応したとはいえなさそうです。では「FlatConfig に対応した sharable config/Plugin」かどうかはどうやって見分けるのでしょうか？[^1]

### eslint ユーザーの行う「FlatConfig 対応」

利用している Plugin や sharable config が FlatConfig に対応している場合、ユーザーは既存の Config を FlatConfig の形式に書き換えるだけで対応が済みます。より具体的には次のような作業を進めるだけで良いでしょう。

- 利用している Plugin や sharable config を import する
- 既存の個別に設定した rule やその他項目を configuration object の方式にする
- extends している config は前段に override している設定は後段に配置する

また利用している Plugin や sharable config が FlatConfig に対応していなくても [FlatCompat](https://github.com/eslint/eslintrc) を利用すれば暫定的な対応ではありますが移行できます。FlatCompat は Config 全体の変換だけでなく `extends` や `env`,`plugin` だけといった部分的な変換にも対応しています。

### sharable config Package に対する「FlatConfig 対応」

ではどうなれば sharable config Package が FlatConfig に対応していると言えるのでしょうか。これは eslint ユーザーの行う「FlatConfig 対応」とそこまで変わりません。結局のところ sharable config Package が export しているのも config(file/ object) なので、 FlatConfig 形式のオブジェクト(configuration object の配列)が公開されているパッケージであれば FlatConfig に対応していると言えます。

ただ、FlatConfig 環境下では「FlatConfig 形式の config オブジェクトがモジュールとして import できさえすれば良い」ので、古い形式の config 時に[存在した以下のようなルール](https://eslint.org/docs/v8.x/extend/shareable-configs#creating-a-shareable-config) に従わなくてよくなります。

- `eslint-config-` や `@scope/eslint-config` で始まるパッケージ名でないといけない
- package.json の `main` エントリポイントで config オブジェクトが export されていること

逆にこれらのルールは前述した「rule や config を独自で解決する辛さ」を低減するためのルールだったとも言えます。

### プラグインに対する「FlatConfig 対応」

最後に FlatConfig 対応されている Plugin とはどういったものなのか見ていきます。

そもそも eslint の Plugin は大きく「config を持つ Plugin」と「config を持たない Plugin」の 2 つに大別されます。Plugin の実体は以下のように独自の `rule` や `config`,`processors`, メタ情報などを持ったオブジェクトです。

```js
const plugin = {
    meta: {...},
    configs: {...},
    rules: {...},
    processors: {...}
};
```

基本的に Plugin は 1 つ以上の rule を持ちますが、config は必須ではありません。rule の多い plugin などでは一括で rule を指定するために `"recommended"` や `"all"` といった `config` を持つことが一般的ですが、rule が少ない Plugin であれば config を持たないものも少なくありません。FlatConfig はあくまで「`config` の新しい形式」ですから、 その Plugin が `config` を持っているかどうかで FlatConfig 対応の判断は変わります。またそもそも Plugin の rule だけを利用していて config を利用しない場合、config が FlatConfig に対応しているかどうかは問題になりません。

#### Plugin が config を持っていない場合 or Plugin の config を利用していない場合

config を持っていない場合、基本的には FlatConfig で利用できることが多いです。ただし以下のような設定を行なっている場合、FlatConfig では読み取れないので注意が必要です。

- `plugin.processor` プロパティがあり、その key に拡張子を使っている
  - 拡張子を key にする記法はサポートされなくなった
- `plugin.environments` を利用しており、plugin.config 内で plugin.environments と同じ設定を行なっていない
  - `plugin.environments` は読まれなくなるので Configs に移動し config の 1 つとして読み込むなどの方法が必要

詳しくは eslint 公式の [Plugin Migration to Flat Config](https://eslint.org/docs/latest/extend/plugin-migration-flat-config) を参照してください。

#### Plugin が config を持っていてその config を利用している場合

Plugin が config を持っていてかつ利用している場合、その config はもちろん FlatConfig の形式であることが求められます。一方、sharable config Package の時と同様、FlatConfig 形式の Config は JS のモジュールとして import できるようになっていればよく、必ずしも Plugin オブジェクトの `config` プロパティに配置しないといけないわけではなくなりました。(逆に言えば古い形式の config 環境下では Plugin オブジェクトの `configs` プロパティに config を配置しないと `plugin:hoge/recommended` のような名前で extends できませんでした。)

そのため FlatConfig に対応した config を持つ Plugin の config 提供方法は主に２種類あります。

1 : plugin オブジェクトの config プロパティに古い形式の config 用のものと合わせて flatConfig 対応のものが列挙されている
2 : plugin オブジェクトとは別に何らかの形で config オブジェクトを export している

1 の場合公式では「古い形式の config の key には `-legacy` を付けて、将来サポートされないことを明確にすること」が[推奨されて](https://eslint.org/docs/latest/extend/plugin-migration-flat-config#backwards-compatibility)います。

```js
const plugin = {
  //...
  configs: {
    recommended : {...} // FlatConfig
    "recommended-legacy" : {...} // 古い形式の config
  },
};
```

しかし実際には `"flat/recommended"` のように FlatConfig であることを明示的に表す形で提供している。Plugin もあります。そのため Plugin に応じて README や実際の Plugin オブジェクトを確認することが大切です。

2 の場合、Plugin オブジェクト自体と違う path で export している場合もありますし、Plugin によっては config や config を生成するための関数を Plugin 本体とは違う Package として公開している場合もあります。

後者の代表的な例としては `typescript-eslint` が挙げられます。`typescript-eslint` の Plugin の本体は `@typescript-eslint/eslint-plugin` という名前の Package ですが、これとは別に `typescript-eslint` という名前の Package を公開しており、この package から FlatConfig 形式の config や config を簡単に生成できる関数などを export しています。

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

FlatConfig の `configuration object` では `files` プロパティで明確にその config が対象とする範囲を指定する形に変更されました。ユーザー側として対象を明確に指定するのは可読性のためにも良さそうですが、sharable config や plugin の config を作る側にとっては、ユーザーが実際どのようなファイル(js/ts/jsx/tsx...)やパスで実行するかわからず少し不便です。むしろこのような config 提供者としてはでざっくりと「ユーザーの lint の対象になってる JS/TS 系のファイル全て」といった指定がしたくなります。

このような指定ができるよう、FlatConfig の `configuration object` で `files` プロパティを指定しない場合「他の設定オブジェクトによってマッチされたすべてのファイルに適用される」というルールがあります。

https://eslint.org/docs/latest/use/configure/configuration-files#configuration-objects

そのため基本的に sharable config や plugin の config は `files` プロパティを指定しないで提供されることが多いです。これは「ユーザーは別の `configuration object` で対象にしたいパスや拡張子をちゃんと指定しているだろう」という前提に立つものですので、sharable config や plugin の config だけを使っていたり、「特定の Path はこれらの config にしか任せていない」といった設定にする場合は注意しましょう。

### グローバルな ignore

古い形式の config では `ignorePatterns` や `.eslintignore` ファイルで lint 対象からファイルを除外できました。FlatConfig でも `configuration object` に `ignores` プロパティがあり、「`ignores` プロパティだけの `configuration object`」はグローバルな ignore 設定とみなされます。

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

### 共通の settings

古い形式の config では Plugin などもアクセス可能な共通の設定情報を指定する `settings` プロパティがありました。FlatConfig においてはこれに変わるものとして `configuration object` に `settings` プロパティがあります。`configuration object` 内にあるのでそのスコープに閉じているように見えますが、`settings` プロパティで設定された情報は全ての `configuration object` で共有されるので注意しましょう。

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

また `settings` プロパティ配下は全体で共有されてしまうため Plugin ごとに名前空間を作ることが慣例となっています。

https://eslint.org/docs/latest/use/configure/configuration-files#configuring-shared-settings

### 微妙に違う languageOption

古い形式の parserOptions は基本そのまま languageOptions 内に移動しましたが、ecmaVersion と sourceType だけは parserOptions から一段上の languageOptions 配下に移動しました。

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

「parserOptions はそのまま languageOptions 内に移動すればいいんだな」と思っている人注意しましょう。

## まとめと次回予告

ここまで FlatConfig についてと FlatConfig 移行という言葉の多層性、実際に移行するときに役立つ Tips などを紹介してきました。この記事を読んで、FlatConfig と FlatConfig 移行についてより詳しくなり、移行するモチベーションや移行した config をより深く理解するモチベーションになれば嬉しいです。

さて、この記事では FlatConfig 移行最後のチェックポイントである「何を持って移行できたとするのか」という点について触れていません。lint が以前と同じように効いているのを確認するまでが FlatConfig 移行なはずです。というわけで「FlatConfig に移行できた」ことをどうやって確認するかやそのためのチェックツールを作った(ている)話を次に書こうと思います。お楽しみに！

## 参考

- [ESLint's new config system, Part 1: Background](https://eslint.org/blog/2022/08/new-config-system-part-1/)

[^1]: 主要な Plugin であれば eslint のリポジトリにある [Tracking: Flat Config support](https://github.com/eslint/eslint/issues/18093) という issue からサポート状況を見ることができます。
