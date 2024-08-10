---
title: "「FlatConfig対応できた!」と自信を持ちたいあなたへ"
emoji: "✅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["JavaScript", "eslint", "frontend"]
publication_name: "cybozu_frontend"
published: false
---

:::message
この記事は、[CYBOZU SUMMER BLOG FES '24](https://cybozu.github.io/summer-blog-fes-2024/) (Frontend Stage) DAY 10 の記事です。
:::

## 自信を持って「FlatConfig 対応完了した！」って言いたい

以前、自分が書いた記事では FlatConfig 自体についてや FlatConfig 移行という言葉の多層性、実際に移行するときに役立つ Tips などを紹介しました。

https://zenn.dev/cybozu_frontend/articles/about-eslint-flat-config

ただ、この記事では FlatConfig 移行における大事な最後のピースである「移行ができたことをチェックする」という点について言及しませんでした。Config を FlatConfig の形に変更できたからといって、動かないのはもっての他、動いているように見えて以前と同じチェックができていないようでは「移行できた」とは言えないでしょう。

この記事では、既存の旧 Config から FlatConfig に移行している人に向けて、何を持って移行できたと判断するか、そのためのツールを作った話をします。

## 何を満たしたら安全に移行できたとするか

### 前提

ここでは旧 Config から FlatConfig へ移行する際に「ひとまず Config 以外の差分なく移行できるのが理想である」という前提に立ちます。

もちろん、FlatConfig 移行は適用しているルールや利用している Plugin を見直す絶好の機会となり得るでしょう。一方で、FlatConfig への移行と同時にこのような見直しを行う場合、意図しないルールの増減や設定変更がないかより慎重に判断する必要が出てきます。

また、一度 FlatConfig に移行してしまえば、ESLint Config Inspector のようなツールも利用しやすくなるといったメリットもあります。

https://eslint.org/blog/2024/04/eslint-config-inspector/

したがって、より安全に移行するのであれば、まず「旧 Config と差分なく移行できた」ことを確認した上で、改めてルールや設定の見直しを行うことをお勧めします。

### どんな差分が生じるのか

今回のような Config の移行に限らず、eslint のアップデートや設定変更で生じうる差分には以下のようなものがあります。

1. lint を適用するファイルの範囲の差分
2. 特定のファイルに対して適用されている rule・および設定、option の差分
3. rule や parser などの内部的な挙動差分

今回のように同じ Plugin や Sharable Config を利用しつつ、旧 Config から FlatConfig へ移行する場合、生じうる差分は 1. と 2. のケースだと考えられます。逆に言えば、「lint を適用しているファイル範囲が一緒(1)」で「すべてのファイルで適用されている rule・および設定、option が変わっていないか無視できる(2)」ことを確認できれば、「Config の移行による差分はほぼない」と言えそうです。

## 差分をチェックする方法

では、実際に上記の 2 つの基準から差分がないことを確認する方法を考えましょう。

### 「lint を適用するファイルの範囲が同じ」ことをチェック

これは、旧 Config 環境と FlatConfig 環境の両方で利用しているプロジェクト内の lint 対象ファイル一覧を取得できれば良さそうです。

しかし現状、lint の対象ファイル一覧を取得する API などは公開されていません。そのため、実際に eslint を `--debug` オプション付きで実行し、そのログから対象ファイルの一覧を取得するのが現実的な方法と言えそうです。ちなみに、この方法自体は以下の記事を参考にしました。（ありがとうございます 🙏）

https://blog.nnn.dev/entry/2023/10/19/110000

やや泥臭い方法になりますが、これを旧 Config 環境と FlatConfig 環境の両方で行い、一覧を比較できれば「lint を適用するファイルの範囲が同じ」かどうかの確認ができそうです。

### 「すべてのファイルで適用されている rule・および設定、option が変わっていない」ことをチェック

特定のファイルに対して適用されている eslint rule や設定を取得したい場合、eslint の `calculateConfigForFile` という API が利用できます。

https://eslint.org/docs/latest/integrate/nodejs-api#-eslintcalculateconfigforfilefilepath

具体的には、以下のように利用する config ファイルを指定した上でファイルパスから適用されている eslint rule や設定を取得できます。

```js
const eslint = new ESLint({ overrideConfig: config }); // 利用するconfigを指定
const calculated = await eslint.calculateConfigForFile(filePath); // 適用されいてるruleやoption,設定が計算される
```

したがって、前節で取得した lint 対象ファイル全てに対して旧 Config / FlatConfig での計算値を比較すれば、「すべてのファイルで適用されている rule・および設定、option が変わっていない」ことが確認できそうです。

ただし、旧 Config を指定するか FlatConfig を指定するかによって `calculateConfigForFile()` の計算結果の構造が微妙に変わる点は注意が必要です。具体的には以下のような違いがあります。

- 旧 Config 環境で計算すると `parserOption` に入る情報は FlatConfig 環境下では基本的に `languageOption.parserOption` に配置される
  - ただし `"ecmaVersion"` と `"sourceType"` は `languageOption` 直下に配置される。
- 旧 Config 環境で計算すると `env` プロパティが指定されていることがあるが、FlatConfig 環境下では `env` は廃止されている
  - 旧 Config 環境では `globals` を直接指定するだけでなく `env` で環境情報を指定できたが、FlatConfig では一律 `globals` オブジェクトで指定することになったので `env` と `globals` の対応関係を考える必要がある

そのため、純粋に旧 Config / FlatConfig で計算されたオブジェクトを比較するだけでは「適用されている rule・および設定、option が変わっていない」ことは検証できず、どちらかに合わせた変換が必要となります。

また、`rules` 配下の各 rule 設定も `"off" | "warn" | "error"` で指定するか `0 | 1 | 2` で指定するかが plugin や config ごとに異なったり、変わったりすることがあるので、単純な値比較はできません。これらの値も文字列か数値かどちらかに正規化する処理が必要になります。

## 人力チェックなんて無理？ じゃあツールを作ろう！

ここまで「FlatConfig へ移行した際に差分がないかのチェック方法」を具体的に見てきましたが、正直これらを人力で行うのはかなり厳しいでしょう。

「lint を適用するファイルの範囲が同じ」ことをチェックするくらいであればまだ何とかなりそうな気もしますが、「適用されている rule・および設定、option が変わっていない」ことのチェックに関しては、微妙な計算値の構造の違いを吸収しつつ正規化も行う必要があります。おまけにこれを lint 対象の全ファイルで行うとなると気が遠くなるでしょう。

そこで、これらをチェックできる `check-eslint-config-compat` という CLI ツールを作ってみました。

https://github.com/sajikix/check-eslint-config-compat

### `check-eslint-config-compat` の仕組み

上で説明した通り、`check-eslint-config-compat` のコアとなるアイデアは以下の 2 点について旧 Config 環境と FlatConfig 環境での計測結果を比較することで差分を検出するというものです。

- eslint の適用範囲
- 適用範囲内すべてのファイルの rule・および設定、option

ただ、eslint のバージョンや環境によっては旧 Config と FlatConfig を同じ環境で動作させられない可能性があります。そのため、`check-eslint-config-compat` は動作を次の 2 ステップに分けて差分を検出します。

1. 旧 Config 環境で適用されるファイル一覧とすべてのファイルでの設定情報を計算し、JSON データとして保存
2. FlatConfig 環境でも適用されるファイル一覧とすべてのファイルでの設定情報を計算し、1. で保存した JSON データと比較することで差分を検出

このように 2 ステップに分けることで「1. を実行 → eslint のアップデート & FlatConfig に書き換え → 2. を実行してチェック」といった利用も可能になります。

#### 1.のステップについて

1 ステップ目では適用ファイル一覧とすべてのファイルでの設定情報を計算して JSON データに保存するわけですが、その方法をもう少し具体的に説明すると以下のようになります。

1. CLI オプションから旧 config のオブジェクトを取得
2. eslint を `--debug` オプション付きで実行し、そのログから対象ファイルの一覧を取得する
3. 2.で取得したパス全てに対して `calculateConfigForFile` を実行
4. 3. で計算したデータを FlatConfig 向けの構造に変換 & rule の設定を正規化
5. 4. で全く同じ出力になったファイルはグルーピングする
6. 2. と 5.の情報を JSON に書き出す

基本的な情報は 1~3 までで取得できていますが、4. を行うことで後に FlatConfig 環境下の計算値と比較しやすくしています。また、5.をしないと eslint の適用されているファイルの数分データが長くなってしまうため、なるべく書き出すデータを少なくする工夫をしています。

実際に保存されるデータは以下のような形式になっています。

```json
{
  "targets": [
    "./src/hoge.ts",
    // .... 対象ファイルの一覧
  ],
  "filesConfig": [
    {
      "key": "./src/hoge.ts", // 最初に計算したfileをkeyにしてる
      "targetFilePaths": [
      "./src/hoge.ts"
      //...同じ設定値のファイルたち
      ],
      "languageOptions": {
        "ecmaVersion": "latest",
        "sourceType": "module",
        "globals": {...},
        "parseOptions": {...}
      },
      "rules": {
        "indent" : [...],
        //...
      }
    }
  ]
```

#### 2.のステップについて

2 ステップ目では、1 ステップ目で作成したデータ（以下、Compat データと呼ぶことにします）と現在の環境下での計算値を比較します。具体的には以下のような手順で確認します。

1. CLI オプションから旧 config のオブジェクトを取得
2. eslint を `--debug` オプション付きで実行し、そのログから対象ファイルの一覧を取得する
3. 2. と Compat データの `"targets"` を比較する（差分があれば報告する）
4. 2. で取得したパス全てに対して `calculateConfigForFile` を実行
5. 4. で取得し、正規化したデータと Compat データから取得した設定情報が同じかチェックする
6. 5. で差分があればどのファイルのどこが違うのか報告する

これで晴れて eslint の「適用範囲」と「適用範囲内すべてのファイルの rule・および設定、option」の差分を検出できたことになります。

### `check-eslint-config-compat` の簡単な使い方紹介

`check-eslint-config-compat` の仕組みを理解したところで、簡単な使い方も紹介します。

この package は GitHub Packages で公開しているため、準備として直下の `.npmrc` で以下のように設定する必要があります。

```.npmrc
@sajikix:registry=https://npm.pkg.github.com
```

次に旧 Config 環境下で `extract` コマンドを実行し、Compat データを作成します。

```
@sajikix/check-eslint-config-compat extract -c .eslintrc.js -e js,jsx -t ./
```

`-c` は config ファイルのパス、`-e` は対象の拡張子の指定、`-t` は lint のターゲットを指定するオプションです。また、`-o` で Compat データの出力先も指定できます。（デフォルトでは `./.compat.json` で作成されます）

Compat データが得られたら、eslint をアップグレードしたり、環境変数をセットするなどして FlatConfig が利用できる環境にし、新しく FlatConfig 形式の config ファイルを作成します。FlatConfig 形式の config ファイルを一通り書けたと思ったら、今度は `compare` コマンドで現在の設定が Compat データと合致するか確かめます。

```
@sajikix/check-eslint-config-compat compare -c eslint.config.js -i .compat.json -t ./
```

`-c` は config ファイルのパス、`-i` は Compat データのファイルパス、`-t` は lint のターゲットを指定するオプションです。

このように `compare` コマンドを実行すると、Compat データ（=旧 Config 環境）との差分を報告してくれます。例えば、eslint の適用範囲が異なっている場合、以下のように報告します。

```
🚨 There is a difference in lint targets
following files are increased as lint targets... [
  './hoge.ts',
  './fuga.ts'
]
```

また、rule の option に差分があったときなどは以下のように報告してくれます。

```
"object-property-newline" rule have different options.
  - Deleted from oldConfig
  + Added in newConfig.

  @@ -2,3 +2,2 @@
  "allowAllPropertiesOnSameLine": true,
  -   "allowMultiplePropertiesPerLine": false,
  }
```

### まだできていないこと・注意点

このような `check-eslint-config-compat` ですが、まだまだ動作が不安定な部分や改善できていない点がいくつかあります。

- デフォルト値を解釈した差分検知ができない
  - 現在の挙動では rule の option でデフォルト通りに設定した時と設定しなかった時を違う option として認識してしまいます。
- グローバルの差分がどうしても出やすい
  - `env` → `globals` の変換を [`globals` パッケージ](https://www.npmjs.com/package/globals)を利用して行なっているが、差分として検知されてしまうことが多い
- `extract` コマンドで指定できるのは旧 Config 限定、`compare` コマンドで指定できるのは FlatConfig 限定になってしまっている
  - 本当は FlatConfig 同士の比較とかもしたいニーズはありそう
- 上記のような理由から大きいプロジェクトだとかなり差分の報告が長くなる傾向にある。
  　- ターミナルで見切れてしまう場合は、適宜 log ファイルとして書き出すなどの方法を取ると良いです。

基本的には少しでも差分があれば報告する側に倒しているので、「報告された差分を見て無視できるものかどうか判断してもらう」という使い方をしてもらえると嬉しいです。(今後 `--ignore` のようにな option を加えるかもしれません。)

## まとめ

今回は FlatConfig 移行における旧 Config からの差分検知の方法と、それを行うツールの紹介をしました。帰るまでは遠足なように、ちゃんと移行できたことを確認するまでが FlatConfig 移行です。この知見が少しでも安全な FlatConfig 移行の助けになれば嬉しいです。
