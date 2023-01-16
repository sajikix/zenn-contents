---
title: "Frontend Weekly #1 (2023-01-10号)"
emoji: "🎍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
publication_name: "cybozu_frontend"
published: false
---

こんにちは! サイボウズ株式会社 フロントエンドエキスパートチームの[Saji(@sajikix)](https://twitter.com/sajikix)です。

# はじめに

フロントエンドエキスパートチームでは毎週火曜日に Frontend Weekly という 「1 週間の間にあったフロントエンドニュースを共有する会」 を社内でに開催しています。

今年からはこの Frontend Weekly で取り上げたニュースを Zenn でも簡単に共有できたらと思っています！

この記事では 2023/01/10 の Frontend Weekly で取り上げた記事や話題を共有していきます！

# 今週取り上げた記事・話題

## Next.js 13.1 がリリース

https://nextjs.org/blog/next-13-1

Next.js 13.1 がリリースされました。主要なアップデートは以下の通りです。

- app ディレクトリを使った際の初回の JS の読み込みサイズ減少
- next-transpile-modules で行ってたモジュールのトランスパイルの機能が本体に組み込まれた
- Edge Runtime が stable に
- etc...

## Future of Storybook in 2023

https://storybook.js.org/blog/future-of-storybook-in-2023/

Storybook が 2022 年の振り返りと、2023 にやることについてブログを出しています。

2022 年の振り返りとしては以下のようなことが挙げられています。

また 2023 年には以下のようなことに取り組むようです。

## Interop 2022: end of year update

https://web.dev/interop-2022-wrapup/

Interop2022 として取り組んだブラウザ相互互換性に関する各ブラウザのアップデートについてのまとめです。

:::message
Interop は Web プラットフォームの相互運用性を改善するために各ブラウザベンダーなどが協力する取り組みです。発足の経緯など詳しくは以下の記事がわかりやすいです。
https://web.dev/interop-2022/
:::

Interop 自体の取組開始当初(2022 年 3 月ごろ)と比べると互換性のベンチマークスコアがかなり改善されています。特に、[ダッシュボード](https://wpt.fyi/interop-2022) を見ると Viewport Units は全ブラウザて 100% になっています。

また web.dev にブラウザ互換性の最新情報に関する[記事集](https://web.dev/tags/newly-interoperable/)があるので合わせて参照すると良さそうです。

引き続きブラウザの相互運用性向上に向けて Interop2023 の計画段階に入っており、近く採用された機能が発表になるそうなので注目していきたいところです。

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

React Native が TypeScript を first-class サポートしました。

共有会参加者からは 「React は今だに Flow で書かれているけどこの先どうするんだろう」 といった声も出ていました。

## ウェブアクセシビリティ導入ガイドブック

https://www.digital.go.jp/resources/introduction-to-web-accessibility-guidebook/

デジタル庁がウェブアクセシビリティ導入ガイドブックを公開しています。

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

具体的には...。

Chrome と Firefox が v100 になる時も同じような問題がありましたが、UA 文字列のパース処理には気をつけたいところです。

## WebKit Features in Safari 16.2

https://webkit.org/blog/13591/webkit-features-in-safari-16-2/

Safari 16.2 の新機能等についてです。
上で書いた　 interop 2022 を頑張りましたみたいな話もしてます。

## 技術的負債は開発者体験を悪化させる / Technical Debt and Developer Experience

https://speakerdeck.com/mtx2s/technical-debt-and-developer-experience

技術的負債を作らないように関係者へ説明するときにとても使える資料皆さんも一度目を通しておくと良さそうです。

# あとがき

今後も可能な限り Frontend Weekly であがったニュースを Zenn でも共有していくつもりですのでよろしくお願いします！

フロントエンドエキスパートチームでは毎月最終火曜の 17 時から Frontend Monthly というイベントを Youtube Live で開催しています。その月のフロントエンド注目ニュースやゲストを呼んでの対談などフロントエンドに関する発信をしていますので是非見てくださいね！

https://www.youtube.com/playlist?list=PLPTndynQK4dxLZFEZgOZjt_zKG-0JWoWy

またフロントエンドエキスパートチームでは、一緒にサイボウズのフロントエンドを最高にする仲間を募集しています！ 気になる方は下のリンクをクリック！

https://cybozu.github.io/frontend-expert/
