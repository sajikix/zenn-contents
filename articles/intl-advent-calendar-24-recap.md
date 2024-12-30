---
title: "1人Intl Advent Calendar を完走しました"
emoji: "↩️"
type: "idea"
topics: ["Intl", "i18n", "frontend"]
publication_name: "cybozu_frontend"
published: true
---

こんにちは！サイボウズ株式会社フロントエンドエキスパートチームの[@sajikix](https://x.com/sajikix)です。

## 1 人 Intl Advent Calendar を書きました

今年(2024 年)の 12 月 1 日から 25 日まで、「1 人 Intl Advent Calendar」と題して Intl に関わる技術ブログを 25 日連続で執筆しました。

https://adventar.org/calendars/10555

この記事ではその振り返りと個人的なおすすめの記事・読み方などを紹介します。

### なんで `Intl` で 1 人 Advent Calendar を書いたの？

そもそも、`Intl` について興味を持って調べ始めたのは去年(2023 年)に遡ります。元々は社内で多言語対応と文言リソース管理の改善をしようというプロジェクトにフロントエンド刷新メンバーとして参加していて、その関連で 1on1 してた先輩に勧められて調べ始めました。

その後、チームで担当させていただいた WEB+DB Press の連載で入門記事を書かせていただいたり、その年の JSConf JP で発表させてもらったりと、Intl に関する発信の機会について恵まれました。

https://speakerdeck.com/sajikix/intlnojin-madetokorekara

また今年の夏にはフロントエンドカンファレンス北海道で `Intl.MessageFormat` について紹介する発表もさせていただきました。

https://speakerdeck.com/sajikix/the-future-of-frontend-i18n-intl-dot-messageformat

このような中で、今年の年末近くになり仲間内で「1 人アドベントカレンダー書いてみたくない？」という話があがりました。個人的にもいつかは 1 人で書いてみたいという思いもあり、書くのであれば今まで調べていたことの総まとめとして `Intl` をテーマにしようと決めました。

### 今回の Advent Calendar コンセプト

上記の通り、今回の Advent Calendar は「個人的に Intl について今まで調べたことの整理とまとめ」という側面が強いです。そのため以下のような目標・コンセプトを持って構成したつもりです。

- 全部の Intl API と Proposal を網羅する
- 前提となる関連仕様などもなるべく紹介する
- 機能のまとまりごとに分けて記事を書いていく

そもそも、今ままで色々調べた Intl についての知識を自分の中で整理する意味もあるので、API や Proposal を網羅するようにしています。(もし漏れてる Proposal などあれば教えていただけると嬉しいです)。また `Intl` はどうしても他の標準仕様と関連が強く、その点が理解をする上でのハードルになりがちだと感じていたため、関連する仕様についてもなるべく紹介するような構成にしました。さらに 1 人 Advent Calendar は記事数も多く連日投稿するのでなるべく近い機能の API や Proposal をまとめたり、近い日付で置くようにしています。

また、記事の内容については「**前提知識が必要な場合はなるべく載せる**」、「**コード例をなるべく増やす**」、「**なるべく具体的なユースケースを載せる**」といったことを意識しています。日時や言語の文法に関わる機能などはそもそも「国際化をしていないと実感しづらい問題」を解決しようとするものが多く、前提となる課題とそれがどう解決されるのかを意識しやすいように努めました。

### 特におすすめの記事

ここでは想定する読者層に合わせて、特におすすめの記事をいくつか紹介したいと思います。

#### Intl を初めて知った方や JS 初学者の方

`Intl` について最近知ったよという方やそもそも最近 JavaScript の勉強をしているよという方は以下のような主要な機能を紹介した記事を薦めたいです。

- 1 日目 : [Intl の全体像と基本となる IF(#1)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-01)
  - Intl の全体像と基本的な使い方、共通する仕様について解説しているので、まずはこちらを読むと良いかと思います。
- 7 日目と 8 日目 : [Intl.DateTimeFormat の基本(#7)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-07) / [Intl.DateTimeFormat の主要オプションを抑える(#8)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-08)
  - 日時のフォーマットについての記事です。日時のフォーマットは Intl での国際化の中でも一番使われる機能の 1 つなので、おすすめです。
- 13 日目と 14 日目 : [Intl.NumberFormat の基本(#13)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-13) / [Intl.NumberFormat の主要オプションを抑える(#14)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-14)
  - 数値のフォーマットについての記事です。こちらも日時と同様に国際化の中でよく使われる機能の 1 つであり押さえておくと色々な場面で使えるのでおすすめです。

#### Intl の API を望来したい方

`Intl` の API についてはなんとなく知っているけど、各機能を網羅して知りたいという方は上記の記事に加えて以下の記事を見ると網羅できます。

- [Intl における 2 つの組み込みメソッド(#5)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-05)
- [Intl.RelativeTimeFormat(#11)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-11)
- [Intl.PluralRules(#17)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-17)
- [Intl.ListFormat(#18)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-18)
- [Intl.Collator(#19)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-19)
- [Intl.Segmenter(#20)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-20)
- [Intl.DisplayNames(#22)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-22)

#### 周辺仕様やライブラリに興味がある方

`Intl` の全体像については知ってるけど、細かい仕様や関連する仕様などについてはそこまで詳しくないという方には以下のような記事でより Intl について詳しくなれるかなと思っています。

- 2 日目 : [ロケール識別子(BCP47)と Unicode 拡張について(#2)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-02)
  - Intl で使われるロケール識別子についての記事です。何気なく使っている「言語タグ」への理解を深められますし、 Unicode 拡張などは Intl の文脈でよく出てきます。
- 3 日目 : [Intl.Locale オブジェクトと Intl Locale Info proposal(#3))](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-03)
  - Intl.Locale オブジェクトは他の Intl API でも使えるオブジェクトですが、意外と紹介する記事も少ないなと思っています。
- 4 日目 : [ロケールネゴシエーションと Intl.LocaleMatcher Proposal(#4)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-04)
  - Intl がロケールをどう解決しているのか、というかなり仕様や実装側によった話です。実際にコードを書く上で意識することは少ないかもしれませんが、知っておくと Intl の挙動が理解しやすくなるかと思います。
- 6 日目 : [Intl を支える Unicode の仕様・データ・ライブラリ(#6)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-06)
  - Intl は間違いなく Unicode の提供する仕様やデータ、ライブラリと密接に関わっているのですが、これらについて Intl と合わせて紹介する日本語の記事は少ないと思っています。
- 9 日目 : [Intl.DateTimeFormat と Temporal の関連(#9)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-09)
  - Intl と新しい日時操作 API Temporal についてその関連を紹介した記事です。少し脱線気味かもしれませんが今後の来そうな API の関連についてもぜひ注目したいと思っています。

#### Proposal に興味がある方

`Intl` に限らず新しい仕様の提案やその議論に興味がある方も一定数いるかと思います。そういった方には以下のような Proposal についての記事をおすすめします。

- 4 日目 : [ロケールネゴシエーションと Intl.LocaleMatcher Proposal(#4)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-04)
- 10 日目: [世界の暦に対応するための Proposal(#10)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-10)
- 12 日目 : [Intl.DurationFormat Proposal(#12)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-12)
- 16 日目 : [Smart Unit Preferences と Representing Measures Proposal(#16)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-16)
- 21 日目 : [Intl.Segmenter v2 Proposal(#21)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-21)
- 23 日目~ : [Intl.MessageFormat 基礎(#23)](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-23)

### 完走した感想

事前に調べていた部分が多かったとはいえ、25 日連続で記事を書き続けるのは初めてで非常に大変でした。ギリギリの投稿時間になってしまう記事もありましたが、なんとか 1 日も落とすことなく完走できたのは良かったです。また書いている間は常にスケジュールと次の記事のプレッシャーを感じながら過ごしていたので、その分完走した時の達成感もひとしおだったように思います。

もちろん辛いことだけではなく、記事を書くことで新しい発見や、取りこぼしていた機能の知見が得られることも多く、非常に有意義ない経験になりました。またより知識を深めたい領域や使用が増えたりといった収穫もあったので今後の探究活動にも繋げていきます。

ひとまずこの 1 人 Intl Advent Calendar は終わりましたが、25 記事に分かれていて読みづらいと自分でも少し思っているので、来年時間のあるタイミングでまとめて加筆修正しつつ Zenn Book にもしたいと思っています。(その時は是非また読んでいただけると嬉しいです)。
