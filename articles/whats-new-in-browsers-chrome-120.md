---
title: "<details>要素のname属性サポート・:dir() CSS 擬似クラスなど - What's new in Browsers!"
emoji: "🪗" # お好きな絵文字を
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CybozuFrontendWeekly", "frontend"]
published: false
publication_name: "cybozu_frontend"
---

※ この記事は [Cybozu Frontend Advent Calendar 2023](https://adventar.org/calendars/9255) の 2 日目の記事です !

What's new in Browsers!は、サイボウズのフロントエンドエンジニアがブラウザの最新情報から気になるトピックを紹介するシリーズです。
今回は Chrome 120 の更新内容から CSS/HTML に関する気になるトピックとして以下の 4 つをピックアップして紹介します。

- Accordion pattern using `name` attribute on `<details>`
- `:dir()` pseudo-class selector
- CSS `<image>`, `<transform-function>`, `<transform-list>` syntax for registered custom properties
- Media Queries: scripting feature

## Accordion pattern using name attribute on `<details>`

`<details>` 要素は開いた時のみ情報が表示されるような折りたたみウィジェットを作成できる要素です。detail 要素が閉じている時に表示される概要やタイトルは `<summary>` 要素で指定します。

```html
<details>
  <summary>詳細</summary>
  この文章はウィジットを開いた時だけ表示される詳細です。
</details>
```

この `<details>` 要素を利用すると JS と CSS を駆使せずとも「アコーディオンメニュー」と呼ばれるような開閉可能なウィジットが並べられたメニューを簡単に作成できます。

ただ、このようなアコーディオンメニューの中には「複数あるウィジットのうち必ず 1 つしか開けない」ような制約を持ったものがあります(以降この記事では「排他的アコーディオンメニュー」と呼びます)。このような排他的アコーディオンメニューの挙動を実現するには `<details>` 要素の `open` 属性を JS で排他制御するのが通例でした。

Chrome120 からサポートされる `<details>` 要素の `name` 属性を利用すればこの「排他的アコーディオンメニュー」を JS での操作なしに実現できます。具体的な挙動として、同じ `name` 属性の値が指定された `<details>` 要素は必ず最後に展開された要素以外展開されなくなります。同じ `name` 属性の値が指定されたラジオボタン(`<input type="radio">`)要素は必ず 1 つしか選択されないのと同じようにイメージすると分かりやすいですね。

実際に利用すると次のようになります。

```html
<details name="menu">
  <summary>オフィス</summary>
  <ul>
    <li>東京</li>
    <li>大阪</li>
    <li>松山</li>
  </ul>
</details>
<details name="menu">
  <summary>部署</summary>
  <ul>
    <li>開発本部</li>
    <li>運用本部</li>
  </ul>
</details>
<details name="menu">
  <summary>製品</summary>
  <ul>
    <li>kintone</li>
    <li>Garoon</li>
    <li>サイボウズOffice</li>
    <li>メールワイズ</li>
  </ul>
</details>
```

この例では複数の `<details>` 要素に同じ `"menu"` という値の `name` 属性を指定しています。このようにすることで JS での操作なしに「排他的アコーディオンメニュー」が Chrome 120 からは実現できるようになります。

## `:dir()` pseudo-class selector

`:dir()` CSS 擬似クラスがサポートされました。`:dir()` CSS 擬似クラスは中に含まれる文字列の方向で要素を指定できる擬似クラスです。`:dir(ltr)` は左から右へのテキストの方向性にマッチし、`:dir(rtl)` は右から左へのテキストの方向性を持つ要素にマッチします。

```css
/* 右から左への文字列がある要素を選択 */
:dir(rtl) {
  color: red;
}
```

ただし `:dir()` CSS 擬似クラスが考慮するのは `dir` 属性などで指定された意味的な文字列の方向であり、`direction` のような CSS プロパティによって指定された文字列の方向には対応しません。

また `[dir=ltr]` のように `dir` 属性のつく要素を属性セレクターで探す場合とも挙動が異なります。 `dir` 属性の値は指定しない場合親から継承しますが、 `dir` 属性を指定していない要素は属性セレクターで探すことができません。一方で `:dir()` CSS 擬似クラスは継承した `dir` 属性の値も含め、ユーザーエージェントが計算した値を元に全てマッチします。

特定の文字列の方向の文書全体にスタイルを適用したい場合などは `:dir()` CSS 擬似クラスが活躍しそうです。

## CSS `<image>`, `<transform-function>` and `<transform-list>` syntax for registered custom properties

現在 CSS Houdini と呼ばれる CSS の低レベル API のセットの仕様が策定されています。 この API 群の中には CSS Properties and Values API と呼ばれる、独自の CSS プロパティを作成できる API があります。CSS 上でこの独自の CSS プロパティを作成するために利用される構文が `@property` ルールです。

`@property` ルールでは次のようにプロパティに許容される構文や初期値、カスタムプロパティの登録を既定で継承するかなどを設定します。

```css
@property --cybozu-blue {
  syntax: "<color>"; /* 許容される構文 */
  inherits: false; /* カスタムプロパティの登録を既定で継承するか */
  initial-value: #64bdd4; /* 初期値 */
}
```

今回のアップデートでは「許容される構文の種類」を指定する `syntax` の取りうる値として以下の 3 つが追加されました。

- `<image>` : イメージとして指定できる値を表すデータ型。 例) background-image の値として指定できる
- `<transform-function>` : 要素の外見に影響する座標変換を表すデータ型。 例:) transform プロパティの中で指定できる。
- `<transform-list>` : 上記の　`<transform-function>` のリストを表すデータ型。 例:) transform プロパティの値として指定できる。

これにより　`@property` ルールで独自のトランジションやアニメーションを定義しやすくなります。

## Media Queries: scripting feature

CSS のメディアクエエリのメディア特性に `scripting` が追加されました。このメディア特性を利用することで、JavaScript のようなスクリプト言語が現在のドキュメントでサポートされているかを問い合わせることができます。取りうる値は `none`,`initial-only`,`enabled` の３つで、それぞれ次のような状態を意味します。

- `none` : スクリプト言語が使えない環境または無効になっている
- `initial-only` : スクリプト言語が最初のページロード時のみ有効
- `enabled` : スクリプト言語がサポートされて有効になっている

```css
@media (scripting: enabled) {
  /* スクリプト言語が有効な時 */
}
```

## 他ブラウザの実装状況・標準化の状況

最後にここまで紹介してきた４つの機能について 2023 年 11 月末時点での他ブラウザのサポート状況をまとめます。

|                                                                          | Firefox | Safari    |
| ------------------------------------------------------------------------ | ------- | --------- |
| `name` attribute on `<details>`                                          | ❌      | 🔬(17.2)  |
| `:dir()` pseudo-class selector                                           | ✅(49)  | ✅(16.4)  |
| `<image>` syntax for custom properties                                   | ❌      | ✅ (16.4) |
| `<transform-function>` & `<transform-list>` syntax for custom properties | ❌      | ✅ (16.4) |
| Media Queries: scripting feature                                         | ✅(113) | ✅(17)    |

今回紹介した機能に関しては Safari はほとんど対応しており、Firefox は一部未サポートという状況です。逆に Chrome120 がサポートしたことにより主要 3 ブラウザでサポートが揃ったのは `:dir()` pseudo-class selector と Media Queries: scripting feature になります。

## 終わりに

Cybozu Frontend Advent Calendar 2023 はこちらになります。

https://adventar.org/calendars/9255
