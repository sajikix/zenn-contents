---
title: "Interop2022の取組まとめなど : Cybozu Frontend Weekly (2023-01-10号)"
emoji: "🎍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["FrontendWeekly", "frontend"]
publication_name: "cybozu_frontend"
published: false
---

こんにちは! サイボウズ株式会社 フロントエンドエキスパートチームの[Saji(@sajikix)](https://twitter.com/sajikix)です。

# はじめに

フロントエンドエキスパートチームでは毎週火曜日に Frontend Weekly という 「1 週間の間にあったフロントエンドニュースを共有する会」 を社内でに開催しています。

今回は、2023/01/10 の Frontend Weeklyで取り上げた記事や話題をまとめたものになります。

# 取り上げた記事・話題

## Next.js 13.1 がリリース

https://nextjs.org/blog/next-13-1

Next.js 13.1 がリリースされました。主要なアップデートは以下の通りです。

- app ディレクトリを使った際の初回の JS の読み込みサイズ減少
- next-transpile-modules で行ってたモジュールのトランスパイルの機能が本体に組み込まれた
- Edge Runtime が stable に
- Turbopack がアップデートされ、Tailwind CSS , `next/image` , `@next/font` などがサポートされる

## Future of Storybook in 2023

https://storybook.js.org/blog/future-of-storybook-in-2023/

Storybook が 2022 年の振り返りと、2023 にやることについてブログを出しています。

2022 年は 7.0 の beta リリースが象徴的でしたが、具体的には以下のようなアップデート・改善が行われました。

- パフォーマンス
  - 遅延コンパイルと依存関係のプリバンドルによるオンデマンドアーキテクチャの採用で２倍近いパフォーマンス向上が見られた
- 安定性
  - early warning system を導入し安定性を向上
- 互換性
  - Frameworks API と呼ばれる新しいアーキテクチャを採用し各フレームワークやビルドツールとの互換性が向上した。ファーストクラスの Vite と NextJS のサポートだけでなく、進行中の SvelteKit のサポートや Remix, Qwik, SolidJS などのサポートの準備を進めている。
- テスト
  - [Storybook Test Runner](https://storybook.js.org/blog/interaction-testing-with-storybook/) をリリースし、 Storybook 上で行えるインタラクションテストが大きく普及。

2023 年は初のユーザーカンファレンスである「Storybook Day」を開催し、同時に v7.0 の安定版をリリース予定です。そのほかにも以下のような改善を予定しています。

- Turbopack と Vite によるビルドの高速化
- アクセシビリティテスト / フルページテスト
- React Native サポート
- etc...

2023 年も引き続き Storybook のアップデートに期待できそうです。

## Interop 2022: end of year update

https://web.dev/interop-2022-wrapup/

Interop2022 として取り組んだブラウザ相互互換性に関する各ブラウザのアップデートについてのまとめです。

:::message
Interop は Web プラットフォームの相互運用性を改善するために各ブラウザベンダーなどが協力する取り組みです。発足の経緯など詳しくは以下の記事がわかりやすいです。
https://web.dev/interop-2022/
:::

Interop 自体の取組開始当初(2022 年 3 月ごろ)と比べると互換性のベンチマークスコアがかなり改善されています。特に、[ダッシュボード](https://wpt.fyi/interop-2022) を見ると Viewport Units は全ブラウザて 100% になっています。

また web.dev にブラウザ互換性の最新情報に関する[記事集](https://web.dev/tags/newly-interoperable/)があるので合わせて参照すると良さそうです。

引き続きブラウザの相互運用性向上に向けて Interop2023 が計画段階に入っており、近く採用された機能が発表になるそうなので注目していきたいところです。

## 2022 JavaScript Rising Stars

https://risingstars.js.org/2022/en

毎年 JavaScript エコシステムにおけるトレンドをまとめてくれている JavaScript Rising Stars で 2022 年版が発表されました。

Most Popular Projects Overall 第一位は Bun でした。やはり今年発表された新しい JS Runtime である Bun の注目度は大きかったようで、特に 7 月の伸びがすごいことになっています。

他にも tRPC や Turborepo が各部門で上位に入っているのが 2022 年らしいですね。

## 新規サービスに Remix を採用した話

https://techblog.a-tm.co.jp/entry/2023/01/05/165133

エイチームさんが新規プロダクトの技術選定で、 Remix を採用したよというお話です。

Next.js と Remix が候補に上がり以下のような理由から Remix を採用したそうです。

- 開発チームがフロントエンド・バックエンドで分けられておらず、両方を扱うチームで、この境界を意識せずに開発したかったから
- 認証認可処理がサーバーサイドで完結できる
- 1 年以上前から Remix に注目し、社内での PoC 活動を通じて有用性や実サービスの開発に耐えうるかどうかを検証してた
- OSS 活動を通して開発コミュニティへの参画や、周辺ライブラリのメンテナンスを行ってた

Remix を新規プロダクトで採用した事例ままだまだ少ない印象があるので Next.js との違い、 Remix の利点などで採用するか悩んでいる方は参考になりそうです。

## First-class Support for TypeScript

https://reactnative.dev/blog/2023/01/03/typescript-first

React Native が TypeScript を first-class サポートしました。開発者としては以下の２点が大きなアップデートと言えそうです。

- アプリを作成するテンプレートがデフォルトで TypeScript に
- DefinitelyTyped を利用しなくても組み込みの型宣言が利用できるようになる

Frontend Weekly参加者からは 「React は今だに Flow で書かれているけどこの先どうするんだろう」 といった声も出ていました。

## ウェブアクセシビリティ導入ガイドブック

https://www.digital.go.jp/resources/introduction-to-web-accessibility-guidebook/

デジタル庁がウェブアクセシビリティ導入ガイドブックを公開しています。基本的な部分から丁寧に解説されているので１度目を通すと良さそうです。

## Advent Calendars For Web Designers And Developers (2022 Edition) — Smashing Magazine

https://www.smashingmagazine.com/2022/12/tech-advent-calendars-web-developers-web-designers-2022/

2022 年の関心別アドベントカレンダーまとめです。

共有会参加者イチ推しアドベントカレンダー/記事は以下になります。

- [PerfPlanet](https://calendar.perfplanet.com/2022/)
- [LCP attribution: a fetchpriority breakdown](https://calendar.perfplanet.com/2022/lcp-attribution-a-fetchpriority-breakdown/)
- [Faster data visualizations](https://calendar.perfplanet.com/2022/faster-data-visualizations/)

忙しくてまだ読めていないみなさんも是非、自分の関心ごとのあるアドベントカレンダーを探してみてください。

## Remove the frozen `rv:109.0` IE11 UA workaround after Firefox reaches version 120 (desktop and Android)

https://bugzilla.mozilla.org/show_bug.cgi?id=1806690

一部サイトで UA 文字列から各ブラウザのバージョンを判別する際、firefox のバージョン表記を IE11 と誤認する不具合が発生しました。このため一部の UA 文字列を 109 で当分固定することになりました。

例えば Firefox の v110 でアクセスした場合の UA 文字列は以下のようになります。

```
Mozilla/5.0 (Windows NT 10.0; rv:110.0) Gecko/20100101 Firefox/110.0
```

この文字列の中の `rv:11` の部分だけとマッチ (後ろの `0` を無視して先頭マッチ) して IE11 と判断しているサイトが一定数あったために今回のような不具合が発生しました。そのため v120 になるまでは `rv:` の部分を `rv:109` で固定し、`Firefox/` の部分はバージョン通りに進める形で対応するようです。

Chrome と Firefox が３桁バージョン ( v100 ) になる時も同じような問題がありましたが、UA 文字列のパース処理には気をつけたいところです。

## WebKit Features in Safari 16.2

https://webkit.org/blog/13591/webkit-features-in-safari-16-2/

Safari 16.2 の新機能についてです。

大きく取り上げられていたのは以下の２点です。

- CSS の `font-variant-alternates` プロパティで多くの値に対応した
- CSS の Alignment における `last baseline` に対応

また上で書いた　interop 2022 についての取り組みも紹介されています。

## 技術的負債は開発者体験を悪化させる / Technical Debt and Developer Experience

https://speakerdeck.com/mtx2s/technical-debt-and-developer-experience

技術的負債について開発者体験という視点からわかりやすく分析・言語化している資料です。

技術的負債を作らないように関係者へ説明するときにとても参考になる資料なので、皆さんも一度目を通しておくと良さそうです。

# あとがき

今後も可能な限り Frontend Weekly であがったニュースを Zenn でも共有していくつもりですのでよろしくお願いします！

フロントエンドエキスパートチームでは毎月最終火曜の 17 時から Frontend Monthly というイベントを Youtube Live で開催しています。その月のフロントエンド注目ニュースやゲストを呼んでの対談などフロントエンドに関する発信をしていますので是非見てくださいね！

https://www.youtube.com/playlist?list=PLPTndynQK4dxLZFEZgOZjt_zKG-0JWoWy

またフロントエンドエキスパートチームでは、一緒にサイボウズのフロントエンドを最高にする仲間を募集しています！ 気になる方は下のリンクをクリック！

https://cybozu.github.io/frontend-expert/
