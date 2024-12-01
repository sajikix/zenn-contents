---
title: "Intl.LocaleオブジェクトとIntl Locale Info proposal(#3)"
emoji: ""
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: false
publication_name: "cybozu_frontend"
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 3 日目の記事です。

今回は Intl.Locale オブジェクトについて初期化のオプション、プロパティ・メソッドなどを一通り解説します。また関連する Intl Locale Info proposal についても紹介します。

## Intl.Locale オブジェクトとは

1 つの Intl.Locale オブジェクトは Intl が用意している、「ロケール情報をより簡単に操作するため」のオブジェクトです。

ここで言うロケール情報とは以下のような情報のことです。

- 言語
- 地域
- 文字体系
- カレンダー
- etc..

同じくロケール情報を表せるものとして、[2 日目で紹介した]()ロケール識別子(言語タグ)がありますが、これらと比べ Intl.Locale オブジェクトは以下の点でアドバンテージがあります。

- ローケルに関する各項目を簡単に取得できる
  - ロケール識別子の場合パースが必要ですが Intl.Locale オブジェクトでは言語や地域といった情報をプロパティにアクセスするだけで取得できます。
- ローケルに関する各項目を簡単に変更できる
  - ロケール識別子の場合パース → 該当部分の書き換えという処理が必要になりますが、Intl.Locale オブジェクトではプロパティに値をセットするだけです。

一方 Intl.Locale オブジェクトは JS のオブジェクトなので当然ロケール識別子と比べるとシリアライズが面倒という問題はあります。従ってサーバーなど違うランタイムとのやり取りなどではロケール識別子の方が優れる場合もあります。

### 使う場面

[1 日目の記事]()で解説した通り、Intl のコンストラクタプロパティは必ず第１引数にロケール識別子か Intl.Locale オブジェクト(またはそれらの配列)を受け取ります。

```ts
const jpLocale = new Intl.Locale("ja-JP", { hourCycle: "h12" });
// Intl.LocaleをIntl のコンストラクタプロパティで第１引数として利用する例
const formatter = new Intl.DateTimeFormat(jpLocale, { calendar: "japanese" });
```

したがって一度アプリケーションの用途にあった Intl.Locale オブジェクトを作成すれば、そのオブジェクトを利用した Intl のコンストラクタプロパティで共通のロケールを利用できるようになります。

## Intl.Locale オブジェクトの作成とオプション

ここからは Intl.Locale オブジェクトを利用する方法について具体的に見ていきます。Intl.Locale オブジェクトは `Intl.Locale()` コンストラクタを初期化することで得られます。

```ts
const jpLocale = new Intl.Locale("ja-JP", { hourCycle: "h12" });
```

第１引数はロケール識別子(言語タグ)を受け取ります。一見他の Intl のコンストラクタプロパティと同様に思えますが、以下の２点で異なるので注意が必要です。

- 文字列のロケール識別子しか受け取らない
- ロケール識別子配列や `undefined` を受け取らない

第２引数はロケール情報を構成または補う情報を指定できるオプションです。具体的には以下の９つのオプションが存在しています。

- `language`
  - 言語を指定するオプション
  - 例 : `ja`,`en` のような言語の種別
  - 基本的にはロケール識別子で指定することが多いがオプションでも指定できる
- `script`
  - 文字体系を指定するオプション
  - 例 : `Hant` : 繁体字と `Hans` : 簡体字など
  - 基本的にはロケール識別子で指定することが多いがオプションでも指定できる
- `region`
  - 地域を指定するオプション
  - 例: `US`/`UK`(`en-US`/`en-UK`)など
  - 基本的にはロケール識別子で指定することが多いがオプションでも指定できる
- `calendar`
  - 使用する暦のオプション
  - 例: `japanese`(和暦)や `hebrew`(ヘブライ暦)など
  - ここで指定すると `Intl.DateTimeFormat` で考慮してくれる
- `collation`
  - 文字列の並べ替え順序について指定するオプション
  - 例: `phonebk` を指定するとドイツ語で使われるようなウムラウトが付く文字(Ä、Ö、Ü)などを文字列並び替えで考慮するようになる
  - ここで指定すると `Intl.Collator` で考慮してくれる
- `numberingSystem`
  - 命数法と呼ばれる、「数をどう表すか」のオプション
  - 例: `jpan` の場合一十百千... と言った 4 桁区切りの漢数字で表現する
  - ここで指定すると `Intl.NumberFormat` で考慮してくれる
- `caseFirst`
  - 文字列の並べ替え時に大文字小文字を考慮するかのオプション
  - `"upper"`, `"lower"`, `"false"` のどれかを取る。
  - `"upper"` の場合は大文字が先に `"lower"` の場合小文字が先に `"false"` の場合何もぜずに元の順序を保持する
  - ここで指定すると `Intl.Collator` で考慮してくれる
- `hourCycle`
  - 時刻を 24 時間表記にするか 12 時間表記にするのか、またそれぞれ０始まりかを指定するオプション
  - `"h23"`, `"h12"`, `"h11"`, `"h24"` の４パータンが指定できるが、`"h24"` 方式は基本どの地域でも採用されていない
  - ここで指定すると `Intl.DateTimeFormat` で考慮してくれる
- `numeric`
  - 文字列の並び替えにおいて、数値部分を比べるときに「文字列」として比較するか「数値の大小」として比較するかのオプション
  - 審議値をとり、`false` の場合は「文字列」として比較し、`true` だと「数値の大小」として比較する(default は `false`)
  - 例 : `true` だと、 `"A12"` vs `"A2"` では `"A2"` が先になる(数値の大小で言えば 2 < 12 なので)
  - ここで指定すると `Intl.Collator` で考慮してくれる

このように最初３つのオプション `language`、`script`、`region` は言語や地域などを指定するオプションで、多くの Intl の機能に影響を与えるます。一方残りのオプションは特定の機能(`Intl.Collator` や `Intl.DateTimeFormat` など) の挙動に影響を与えるオプションになっています。

### 初期化時の優先度問題

これらのオプションを見ていると、同じ内容をロケール識別子とオプションでそれぞれ指定できることに気づくと思います。ではそれぞれで違う値を指定した場合優先されるのはどちらなのでしょうか？

例えばロケール識別子では `ja-JP` としておいて `language` や `region` で `en-US` を指定した場合について考えます。

```ts
const locale = new Intl.Locale("ja-JP", { language: "en", region: "US" }); //
```

この場合ロケール識別子よりもオプションで指定した値、つまり `en-US` の方が優先されます。ミス以外でこのような矛盾した値をセットすることは少ないでしょうがミスを発見する際などの役立つでしょう。

また特定の機能の挙動を変えるオプションは Intl.Locale オブジェクトを初期化する場合以外でも指定できます。例えば `hourCycle` などは Intl.DateTimeFormat でも初期化のオプションとして指定できます。では Intl.Locale の初期化時と、その Intl.Locale オブジェクトを利用した各 API のオプションで違う値を指定した場合どちらが優先されるのでしょうか。

```ts
const jpLocale = new Intl.Locale("ja-JP", { hourCycle: "h12" }); // Intl.Locale オブジェクト初期化時は h12 を指定
const formatter = new Intl.DateTimeFormat(jpLocale, { hourCycle: "h23" }); // 利用時は h23 を指定
formatter.format(new Date());
```

この場合、 Intl.Locale オブジェクトを利用する各 API のオプションの方が優先されます。したがって上の例では `"h23"` の方が優先されて「０時始まりの 24 時表記」になります。

よってまとめると

- ロケール識別子と初期化オプションでは初期化オプションの方が優先される
- Intl.Locale オブジェクトを初期化オプションと各 API のオプションでは各 API のオプションの方が優先される

のようになります。

### ロケール識別子では細かいオプションを表現できないのか？

このように Intl.Locale オブジェクトの初期化では一般的なロケール識別子(`"en-US"` みたいなもの)よりも多くのオプションを指定できているように見えます。これらの情報をロケール識別子に含めることはできないのでしょうか？(できない場合ロケール識別子は含められる情報量の面でも下位互換になってしまうのでしょうか？)

実は Intl.Locale オブジェクトで指定できるすべてのオプションは u(Unicode)拡張を利用してロケール識別子に含めることができます(Unicode 拡張については[2 日目の記事]()で解説しました)。

具体的には各オプションがそれぞれ以下の Unicode 拡張に対応します。

- `calendar` → `u-ca-[value]`
- `collation` → `u-co-[value]`
- `numberingSystem` → `u-nu-[value]`
- `caseFirst` → `u-kf-[value]`
- `hourCycle` → `u-hc-[value]`
- `numeric` → `u-kn-[value]`

例えば `hourCycle` オプションで `h12` を指定するのは `u-hc-h12` のような Unicode 拡張をロケール識別子に含むと同じ意味になります。

従って Intl.Locale オブジェクトの初期化においては「ロケール識別子」も「初期化オプション」も指定できる情報は同じです。(現実問題としてたくさんの Unicode 拡張をロケール識別子に含めることで可読性が悪くならないかは留意すべきだと思いますが)。

## Intl.Locale のプロパティ・メソッド

上記の通り Intl.Locale オブジェクトを初期化する際には様々なオプションが指定できました。

これらの初期化時に指定した値すべては、オプションと同じ key 名で生成した Intl.Locale オブジェクトから参照できます。

```ts
const jpLocale = new Intl.Locale("ja-JP", { hourCycle: "h12" });
console.log(jpLocale.language); // "ja"
console.log(jpLocale.region); // "JP"
console.log(jpLocale.hourCycle); // "h12"
```

初期化時のオプションで指定しなかった物については `numeric` を覗き `undefined` として保存されています。(`numeric` は default 値が `false`)

これら初期化時にオプションとして存在した 9 つのプロパティに加え Intl.Locale オブジェクトは　`baseName`　というプロパティを持っています。これは初期化オプションにおける `language`、`script`、`region` を `"-"` で繋いだ文字列を保持するプロパティです。

<!-- 本当は+変異型まで？ -->

また Intl.Locale オブジェクトは上記の 10 個のプロパティ以外に、３つのメソッドも持ちます。

- `toString()`
- `maximize()`
- `minimize()`

`toString()` メソッドは Intl.Locale オブジェクト内に含まれた情報を全て含むようなロケール識別子を返すメソッドです。`language`, `script`,`region` などの情報はもちろんそれ以外に指定したオプションも Unicode 拡張を使ってロケール識別子内に盛り込まれます。

```ts
const jpLocale = new Intl.Locale("ja", { region: "JP", hourCycle: "h12" });
console.log(jpLOcale.toString()); // ja-JP-u-hc-h12
```

`maximize()` メソッドは。

`minimize()`メソッドは。

## 利用例

<!-- 長すぎたら削る -->

### Intl.Locale オブジェクトを共有する

アプリケーション内で利用する Intl.Locale オブジェクトを共有し、Intl の各コンストラクタプロパティで利用することは次のような利点があります。

- `language`、`script`、`region` と言った基本的なロケール情報を統一できる
- 各 Intl 機能のうちロケールに強く依存するオプションのデフォルト値を保持しておける

Intl.Locale オブジェクトを共有すると、毎度ロケール識別子を使ってロケールを指定することがなくなり、ロケール指定ミスを防ぐことができます。また元の Intl.Locale オブジェクトを変更するだけで簡単にアプリケーション全体のロケールを切り替えられます。

さらに `calendar`, `hourCycle` などの特定の機能の挙動を変えるオプションは、あらかじめ指定しておくことで各 Intl 機能のデフォルトオプションのように振舞います。これは Intl 全体のデフォルト値とは違う挙動をアプリケーション全体で統一したい時に有用だと言えます。もちろん上で説明した通り、これらのオプションは個別に上書きが可能です。

### Locale 情報からロジックを分岐する

## Intl Locale Info proposal

このような Intl.Locale オブジェクトに対して、「現状のプロパティ以上の情報を取得するメソッドを追加しよう」という提案が、Intl Locale Info proposal です。2024/12 現在は Stage3 の提案で、すでにでは利用可能です。(APIが変更になる可能性があるので利用には注意が必要)。

### 提案されている Locale Info とユースケース

具体的に提案されているメソッドは以下のnつです。

- getA()

## まとめと次回予告

今回は Intl.Locale オブジェクトと Intl Locale Info proposal について解説しました。次回 4 日目ではロケールネゴシエーションと Intl.LocaleMatcher Proposal について解説します。

## 参考文献

- [Unicode Locale Data Markup Language (LDML)](https://unicode.org/reports/tr35/#unicode-locale-data-markup-language-ldml)
- [Unicode Locale Data Markup Language (LDML)
  Part 5: Collation](https://www.unicode.org/reports/tr35/tr35-collation.html#unicode-technical-standard-35)
