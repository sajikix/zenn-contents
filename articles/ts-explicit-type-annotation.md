---
title: "明示的な型注釈によって推論コストを下げるというアプローチ"
emoji: "⏳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript", "frontend"]
publication_name: "cybozu_frontend"
published: true
---

近年、TypeScript を取り巻くエコシステムでは、**ユーザーに明示的な型注釈を求めることで、推論や型生成のコストを下げる**というアプローチが注目されています。TypeScript 5.5 beta で　発表された `--isolatedDeclarations` オプションはその代表的な機能ですし、Deno の提供する新しいパッケージレジストリ JSR が提唱している [`slow types` という考え方](https://jsr.io/docs/about-slow-types)も同様のアプローチを求めるものです。

この記事では、上記のようなアプローチが提案された経緯や解決したい課題について、TypeScript を利用するエコシステムの状況も踏まえて整理します。

## TypeScript を取り巻くツールチェインと型情報を利用する上でのパフォーマンス

皆さんがご存知の通り、TypeScript の型推論は非常に賢く、その機能は日々アップデートされています。特に以下のような機能は他の言語でも見られることが少なく、TypeScript の推論をより強力なものとしています。

- Generics や豊富な Utility Types
- Conditional Type や Recursive Type などの複雑な型ロジック機能
- 高度な Control flow analysis

一方で、型推論機能を発達させてきたが故に以下のような問題も起き始めています。

- 大規模なプロジェクトでタイプチェックに時間がかかる（パフォーマンス問題）
- tsc 本家以外で高度な型推論をする手段がない（ロックイン問題）
- 明確な挙動や型生成結果の仕様がない

これらの問題は、TypeScript 自体の機能複雑化や Rust 製のツールチェインの発達により顕在化しています。

### Rust 製ツールチェインの発達と Rust で型情報を扱う問題

近年のフロントエンドエコシステムでは、Rust 製のビルドツールや Linter、Formatter が増えてきています。Rust 製のツール群はパフォーマンス面で大きく有利ですが、JavaScript で実装された既存の資産を再利用できないという欠点があります。JavaScript の Parser などは各 Rust ツールで再実装が行われており、一部 Rust 実装間で再利用されつつ、概ね言語仕様の変更には追従できている状況です。

一方で、TypeScript の型情報を利用する Lint 機能やビルド機能を Rust 製ツールで実装する場合、「TypeScript 同等の推論・チェック機能を持った実装が tsc 以外存在しない」という問題が発生します。型情報を利用した Lint 機能やビルド機能とは具体的に次のようなものです。

- `typescript-eslint` にある `no-floating-promises` のように型情報を使った Lint ルールを作成したい場合
- 主にライブラリ提供者側として型宣言ファイルを始めとする型情報の外部出力をしたい場合
- 型情報から自動的にドキュメントを作成したり最適化を行うような場合

### TypeScript Compiler API のパフォーマンス問題

特に Rust 製のツールにおける構文解析は非常に高速です。そのため、TypeScript Compiler API を利用する場合、この部分の速度が Linter やビルドツールの大きなボトルネックとなります。

また、TypeScript Compiler API を利用した型情報の取得は、プロジェクトが大きくなるほどそのコストが上がります。TypeScript は明示的に型を書かなくても、利用している・import している変数や関数から型を推論します。逆に言えば、型情報を確定するにはその型の依存を全て解決する必要があります。

![1つの型が深く依存しているイメージ](/images/tsExplicitTypeAnnotation/ts-type-dep.png)
_1 つの型が深く依存しているイメージ_

この図のように「調べたい型が特定の型の推論結果に依存していて、その型もさらに他の型の推論結果に依存して...」となると、1 つの型・1 つの構文をチェックするだけでもチェックする範囲が膨大になります。型情報を必要とするルールでは毎回このような型依存の解決が必要になるため、ファイル全体の型チェックよりも Lint における型情報の取得のコストが高くなることもしばしばあります。
詰まるところ、TypeScript Compiler API のパフォーマンス問題は以下の点が原因になっているように思えます。

- そもそも複雑な型ロジックや解析においてコストが高い
- 推論のための依存範囲が広くなりやすく、特に大規模プロジェクトではこの依存解決が膨大になる
- 加えて、JavaScript で書かれているため言語レベルで実行速度にビハインドがある

## 型情報を利用しつつ実行速度を早くするための改善案

TypeScript Compiler API による型情報の取得が遅い問題に対して、これまでにいくつかの改善策が考えられてきました。

### 改善案 1 : tsserver を別プロセスで立ち上げる

1 つ目の方法は、「毎回 TypeScript Compiler API を呼び出すのではなく、TypeScript の Language Server([tsserver)](<https://github.com/microsoft/TypeScript/wiki/Standalone-Server-(tsserver)>))を利用することでパフォーマンスを改善しよう」という試みです。Language Server を別プロセスで起動しておくことでエディタの挙動に近づけることも可能になり、事前に解析した部分のキャッシュなども活用でき、速度の向上が見込めます。

![tsserverを利用した型情報の取得](/images/tsExplicitTypeAnnotation/linter-with-tsserver.png)
_tsserver を利用した型情報の取得_

typescript-eslint ではこの機能が　`parserOptions`　の　`EXPERIMENTAL_useProjectService` として試験的に導入されています。

https://typescript-eslint.io/packages/parser/#experimental_useprojectservice

[この機能を導入した PR](https://github.com/typescript-eslint/typescript-eslint/pull/6754) によれば、一部のケースで実行速度を 60~70%減少させる効果があったようです（一部悪化するケースもあるため、絶対に速度が向上するとは言えません）。

### 改善案 2 : tsc 代替プロジェクト

2 つ目の方法は、tsc 自体を他のより高速な言語で作り直し、高速化を図るというものです。実際に Rust や C++で TypeScript 互換のコンパイラ・チェッカーを作成するプロジェクトが複数あります。具体的には次のようなものが有名です。

- [stc](https://github.com/dudykr/stc) : TypeScript の型チェック部分のみを Rust で再実装しようとするプロジェクト
- [ezno](https://github.com/kaleidawave/ezno) : Rust で書かれたコンパイラ・型チェッカ
- [TypeRunner](https://github.com/marcj/TypeRunner) : C++ によって tsc を代替しようとかなり古い試み

しかしながら、stc と TypeRunner は現在どちらも開発が中止されており、ezno はそもそも TypeScript との完全互換を目指していません。したがって、2024 年 5 月現在、TypeScript 互換の別実装は存在していないことになります。

特に Rust の代替実装として期待されていた stc の開発が今年に入って終了したことは、tsc 代替の難しさを物語っているように思います。

> TypeScript was not something that I could follow up on in an alternative language.
> https://github.com/swc-project/swc/issues/571#issuecomment-1915966297

これらの状況を踏まえるとそもそも TypeScript 互換の別実装を開発し続けるには以下のような難しさがありそうです。

- 型推論や挙動に対して明確な仕様書がない
- リリースごとに推論機能がアップデートされ、挙動や出力結果が変わる
- Microsoft の強力なリソースによって開発が実現している

## ユーザに明示的な型注釈を求めることで推論コストを下げる

ここまで 2 つの改善案を見てきましたが、近年これらの改善案に加えて「モジュールからエクスポートされる関数や変数に明示的な型注釈を求めることで推論コストを下げよう」というアプローチが見られるようになってきました。

### 明示的な型注釈があると何が楽になるのか

関数や変数をエクスポートする場合に明示的な型注釈をつけることをユーザーに強制すれば、モジュールを跨いだ型依存の解決をなくすことができます。

![深い依存解決を避けられる](/images/tsExplicitTypeAnnotation/ts-type-dep-iso.png)
_深い依存解決を避けられる_

モジュールを跨いだ型依存の解決をなくすことで、以下のような具体的なメリットがあります。

- モジュールを跨いだ依存解決がなくなるので、型情報の取得が純粋に高速化する
- モジュール間の依存がなくなるので、型検査と型宣言ファイル出力を並列化できる

また、副次的なメリットとして「型注釈が明示的になることで静的な型情報の取得が楽になる」というものもあります。これは後述する tsc 以外での型宣言出力や型推論のサブセット実装におけるメリットと言えます。

### `--isolatedDeclarations` option

このようにモジュールからエクスポートされる関数や変数に明示的な型注釈をつけることを強制するためのオプションとして提案されたのが、`--isolatedDeclarations` オプションです。`--isolatedDeclarations` オプションは Bloomberg のエンジニアによって 2023 年に提案されたもので、詳しくは TypeScript Congress 2023 の発表でも話されています。

https://portal.gitnation.org/contents/faster-typescript-builds-with-isolateddeclarations

その後、2024 年 4 月に発表された TypeScript 5.5 beta で正式にオプションとして導入されることが発表されました。

https://devblogs.microsoft.com/typescript/announcing-typescript-5-5-beta/#isolated-declarations

`--isolatedDeclarations` オプションが有効になった環境下では、明示的な型注釈のない変数や関数をエクスポートしようとするとエラーになります。

```ts
// Function must have an explicit return type annotation with --isolatedDeclarations.(9007)
export const getRandomNum = () => Math.random() * 10; // ❌
```

この例では `getRandomNum` に明示的な型注釈がないにもかかわらず export しているためエラーが出ます。`--isolatedDeclarations` option が有効下では以下のように修正する必要があります。

```ts
export const getRandomNum = (): number => Math.random() * 10; // ✅
```

明らかに型の推論が容易な以下のような場合は、型の注釈を明示的に書かなくてもエラーになりませんが、基本的にプリミティブな定数を宣言する場合など以外は該当しないでしょう。

```ts
// 数値などのプリミティブな値を直接代入しているもの
export const MAX_USER = 100;
// 数値などのプリミティブな値を直接returnしている関数
export const getMaxUser = () => {
  return 100;
};
// asなどで値や返り値をキャストしているもの(回避策として推奨されるとは限らない)
export const MAX_DATA_SIZE = (MAX_USER * 256) as number;
```

既存のコードベースで `--isolatedDeclarations` オプションを有効にした場合、ほとんどのエクスポートする関数・変数に明示的な型注釈を追加する必要が出てくるはずです。

### JSR の slow types

`--isolatedDeclarations` と似た概念のものとして JSR の `slow types` が挙げられます。deno が運用する新しいパッケージレジストリである JSR では「明示的に記述されていない型」や、「理解するために広範な推論を必要とするほど複雑な型」を `slow types` としてマークし、Package から export している関数や変数でこのようなような型を利用していないかをチェックしています。

https://jsr.io/docs/about-slow-types

JSR ではドキュメントの自動生成や npm 互換の型宣言生成のために多くの型解析を行います。そのため、複雑な型や依存の深い型が含まれると、単純な実行時間の増加以外にも以下のような問題が発生します。

- 一部 npm 互換の型宣言生成を行えない場合がある
- 型からドキュメントを生成できない場合がある
- 推論などの挙動変更により生成物の冪等性が損なわれる

このように、パフォーマンス観点以外でも型情報を利用した生成物の正確性や冪等性を向上させるために、明示的な型注釈は有効です。(ちなみに、パフォーマンスにおいても `slow types` を含むパッケージの解析は 1.5 倍から 2 倍ほど遅くなるようです。) このため、JSR では外部に export される変数や関数に関しては `slow types` を使用せず明示的に型をつけることを推奨しており、これをパッケージのスコアリングにも反映させています。

deno-lint では検出された `slow types` にエラーを出す `no-slow-types` という lint ルールがあり、deno で JSR に公開するようなパッケージを作成する場合は自動的に有効化されます。

### tsc 型宣言ファイル出力と型推論のサブセット実装

型注釈が明示的になることは、型宣言のビルドや型情報取得の速度を上げるだけでなく、静的な型情報の取得を楽にするというメリットもあります。

上記の通り、TypeScript 完全互換のコンパイラ・チェッカーを別言語で実装する壁は高いですが、「明示的な型注釈があること・lint に必要な型情報の取得だけで良い」などの制約を設ければ、型宣言の出力や型情報の取得をある程度低コストで実装し直せる可能性があります。

#### 型宣言ファイル出力

tsc 以外のツールによる型宣言の出力に関しては、TypeScript による `--isolatedDeclarations` のリリースブログ内でも「想定されるユースケース」として記載されています。

> Imagine if you wanted to create a faster tool to generate declaration files, perhaps as part of a publishing service or a new bundler.
> https://devblogs.microsoft.com/typescript/announcing-typescript-5-5-beta/#isolated-declarations

実際に `--isolatedDeclarations` が有効化されていれば、エクスポートされる型情報は全て明示的に記述されているため、複雑な型推論なしに型定義ファイルを生成できます。

#### Biome と Oxc の型推論サブセット実装

また、lint 機能を提供する Biome と Oxc は、型情報の取得方法として Rust で限定的な型推論のサブセットを実装する方針で進めています。これは、TypeScript の完全互換を目指すのではなく、Lint ルールを実装する上で必要な型情報の取得だけに用途を絞って推論機能を実装するという方針です。

`--isolatedDeclarations` はこの方針の強い後押しとなる可能性があります。`--isolatedDeclarations` が有効な環境下ではモジュールを跨ぐ型注釈が明示的になるため、ファイル内の構文解析だけで型情報を確定できるようになります。`--isolatedDeclarations` が今後どれだけ有効化されるのかは未知数ですが、「`--isolatedDeclarations` を有効にした時のみ型情報を利用した Lint をサポートする」といった具合に、型推論を必要とする機能が特定の制約上で有効になる可能性が考えられます。

型推論のサブセット実装についてや各ツールの方針について詳しく知りたい方以下のブログも参考にすると良いでしょう。

https://zenn.dev/cybozu_frontend/articles/biome-roadmap-2024#%E5%9E%8B%E3%82%B7%E3%82%B9%E3%83%86%E3%83%A0

https://zenn.dev/kirohi/articles/3c644b614977fe

### 制約と使い所(想像を含む)

このように、TypeScript を取り巻くツール群では、明示的に型注釈を強制するアプローチを前提とした実装が検討され始めていますが、「明示的に型注釈を書く必要がある」という制約はユーザーの負担となりえます。特に、比較的小規模なプロジェクトや型宣言ファイルの出力が必要ないプロジェクトでは、メリットを享受しづらいかもしれません。

#### 明示的な型注釈のメリットが得られやすいケース

ここからは個人的な考えを含みますが、以下のようなケースでは `--isolatedDeclarations` オプションや `no-slow-types` を有効にして明示的な型注釈を強制してもメリットが得られやすそうです。

- 型宣言ファイルを出力する必要があるパッケージ・ライブラリ
  - アプリケーションに比べて独立性が高く、型宣言生成の速度向上の恩恵を受けやすい
  - 特に、JSR に公開する場合は外部に公開されているモジュールに対して明示的な型注釈がついていることが強く推奨される
- モノレポ環境下の他のパッケージが依存しているロジック・コンポーネントのパッケージ
  - これも型宣言ファイルを出力するパッケージ・ライブラリと得られる恩恵は近い
  - 加えて、モノレポ構成のように開発しているパッケージ間で依存のある構成では、型検査と型宣言ファイル出力の並列化のメリットが大きい
- 今後 oxc や biome のような型推論サブセットの実装を検討しているツールを全面的に採用する方針の場合
  - 前述のように、型推論のサブセットの実装では今回のような制約のある環境下でしか推論をサポートしない可能性がある（まだ詳しく方針は決まっていないが）

#### 部分的な --isolatedDeclarations の有効化

また、今後 `--isolatedDeclarations` を部分的に有効化できるようになれば、「毎回型注釈を書く手間」と「性能面でのビハインド」のバランスをとることも可能になるかもしれません。

例えば、「アプリケーションの各コンポーネントの index ファイルだけは明示的に型注釈を書く」、「エクスポートされた hooks や repository には型注釈をちゃんと書く」といったように、アプリケーションの構造上依存の境界となる部分だけ制約を強くすることは、パフォーマンス改善につながるだけでなく、設計上の境界や依存を明確化する上でも役立つと思います。

## おわりに

この記事では、「ユーザーにより明示的な型の注釈を求めることで推論や型生成のコストを下げる」というアプローチについて、TypeScript を取り巻くエコシステムの現状を踏まえながら、その役割や制約、期待されることなどを整理してきました。

`--isolatedDeclarations` や `slow types` 自体もまだ発表されたばかりの機能で、ツール群の対応も決まりきっていない状況ではありますが、今後の動向を追う上でこの記事が役に立つと嬉しいです。

## 参考

今回の記事を作成するにあたって以下の記事・リリースノート・issue 等を参考にさせていただきました。この記事で端折った部分も多いので合わせて読むとより理解が深まると思います。

- https://portal.gitnation.org/contents/faster-typescript-builds-with-isolateddeclarations
  - TypeScript Congress 2023 で行われた `--isolatedDeclarations` option 提案者によるトーク
- https://devblogs.microsoft.com/typescript/announcing-typescript-5-5-beta/#isolated-declarations
  - TypeScript 5.5 beta のリリースノート
- https://jsr.io/docs/about-slow-types
  - JSR slow types についての解説ページ
- https://github.com/jsr-io/jsr/issues/444#issuecomment-2079772908
  - `slow types` についてその必要性などが詳しく書かれた issue コメント
- https://zenn.dev/kirohi/articles/3c644b614977fe
  - Biome の CoreMember である unvalley さんが TS Kaigi で発表した「Exploring Type-Informed Lint Rules in Rust based TypeScript Linters」というトークをブログに書き起こし加筆したもの
  - Rust ツールチェインにおいて型情報をどう取得するのかという問題の現状と各ツールでの方針がまとまっている
- https://www.joshuakgoldberg.com/blog/rust-based-javascript-linters-fast-but-no-typed-linting-right-now/
  - typescript-eslint の maintainer である Josh Goldberg 氏による Rust 製 Linter と型情報の利用についての現状(2024 年 1 月時点)をまとめた記事
- https://www.mizdra.net/entry/2024/05/25/173706
  - mizdra さんによる tsc の大体は作れるのか？というテーマでその可能性や背景を解説したブログ
