---
title: "Intl.LocaleオブジェクトとIntl Locale Info proposal(#3)"
emoji: "📍"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: false
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 3 日目の記事です。

今回は Intl.Locale オブジェクトについて、初期化のオプション、プロパティ・メソッドなどを一通り解説します。また、関連する Intl Locale Info プロポーザルについても紹介します。

## Intl.Locale オブジェクトとは

1 つの Intl.Locale オブジェクトは、Intl が用意している「ロケール情報をより簡単に操作するため」のオブジェクトです。

ここで言うロケール情報とは、以下のような情報のことです。

- 言語
- 地域
- 文字体系
- カレンダー
- など

同じくロケール情報を表せるものとして、[2 日目で紹介した](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-02)ロケール識別子（言語タグ）がありますが、これらと比べて Intl.Locale オブジェクトはロケール文字列を自前でパースする必要がなく、用途に合わせてロケールに関する各項目を簡単に取得できるというメリットがあります。

一方、Intl.Locale オブジェクトは JavaScript のオブジェクトなので、ロケール識別子と比べるとシリアライズが面倒という問題があります。したがって、サーバーなど異なるランタイムとのやり取りではロケール識別子の方が優れる場合もあります。

### よく使う場面

[1 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-01)で解説した通り、Intl のコンストラクタプロパティは必ず第 1 引数にロケール識別子か Intl.Locale オブジェクト（またはそれらの配列）を受け取ります。

したがって、一度アプリケーションの用途に合った Intl.Locale オブジェクトを作成すれば、Intl のコンストラクタプロパティを初期化する際に共通のロケールを利用できるようになります。

```ts
const jpLocale = new Intl.Locale("ja-JP", { hourCycle: "h12" });
// Intl.LocaleをIntl のコンストラクタプロパティで第１引数として利用する例
const formatter = new Intl.DateTimeFormat(jpLocale, { calendar: "japanese" });
```

## Intl.Locale オブジェクトの作成とオプション

ここからは、Intl.Locale オブジェクトを利用する方法について具体的に見ていきます。Intl.Locale オブジェクトは `Intl.Locale()` コンストラクタを初期化することで得られます。

```ts
const jpLocale = new Intl.Locale("ja-JP", { hourCycle: "h12" });
```

第 1 引数はロケール識別子（言語タグ）を受け取ります。一見、他の Intl のコンストラクタプロパティと同様に思えますが、以下の 2 点で異なるので注意が必要です。

- 文字列のロケール識別子しか受け取らない
- ロケール識別子配列や `undefined` を受け取らない

第 2 引数はロケール情報を構成または補う情報を指定できるオプションです。具体的には以下の 9 つのオプションがあります。

- `language`
  - 言語を指定するオプション
  - 例: `ja`, `en` のような言語の種別
  - 基本的にはロケール識別子で指定することが多いが、オプションでも指定できる
- `script`
  - 文字体系を指定するオプション
  - 例: `Hant`（繁体字）や `Hans`（簡体字）など
  - 基本的にはロケール識別子で指定することが多いが、オプションでも指定できる
- `region`
  - 地域を指定するオプション
  - 例: `US` / `UK`（`en-US` / `en-UK`）など
  - 基本的にはロケール識別子で指定することが多いが、オプションでも指定できる
- `calendar`
  - 使用する暦のオプション
  - 例: `japanese`（和暦）や `hebrew`（ヘブライ暦）など
  - ここで指定すると `Intl.DateTimeFormat` で考慮される
- `collation`
  - 文字列の並べ替え順序について指定するオプション
  - 例: `phonebk` を指定すると、ドイツ語で使われるようなウムラウトが付く文字（Ä、Ö、Ü）などを文字列並び替えで考慮するようになる
  - ここで指定すると `Intl.Collator` で考慮される
- `numberingSystem`
  - 命数法と呼ばれる、「数をどう表すか」のオプション
  - 例: `jpan` の場合、一十百千... といった 4 桁区切りの漢数字で表現する
  - ここで指定すると `Intl.NumberFormat` で考慮される
- `caseFirst`
  - 文字列の並べ替え時に大文字小文字を考慮するかのオプション
  - `"upper"`, `"lower"`, `"false"` のどれかを取る
  - `"upper"` の場合は大文字が先に、`"lower"` の場合は小文字が先に、`"false"` の場合は元の順序を保持する
  - ここで指定すると `Intl.Collator` で考慮される
- `hourCycle`
  - 時刻を 24 時間表記にするか 12 時間表記にするのか、またそれぞれ 0 始まりかを指定するオプション
  - `"h23"`, `"h12"`, `"h11"`, `"h24"` の 4 パターンが指定できるが、`"h24"` 方式は基本どの地域でも採用されていない
  - ここで指定すると `Intl.DateTimeFormat` で考慮される
- `numeric`
  - 文字列の並び替えにおいて、数値部分を比べるときに「文字列」として比較するか「数値の大小」として比較するかのオプション
  - 真偽値を取り、`false` の場合は「文字列」として比較し、`true` だと「数値の大小」として比較する（デフォルトは `false`）
  - 例: `true` だと、`"A12"` vs `"A2"` では `"A2"` が先になる（数値の大小で言えば 2 < 12 なので）
  - ここで指定すると `Intl.Collator` で考慮される

このように、最初の 3 つのオプション `language`、`script`、`region` は言語や地域などを指定するオプションで、多くの Intl の機能に影響を与えます。一方、残りのオプションは特定の機能（`Intl.Collator` や `Intl.DateTimeFormat` など）の挙動に影響を与えるオプションになっています。

### 初期化時の優先度問題

これらのオプションを見ていると、同じ内容をロケール識別子とオプションでそれぞれ指定できることに気づくと思います。それでは、異なる値を指定した場合、どちらが優先されるのでしょうか？

例えば、ロケール識別子では `ja-JP` としておいて、`language` や `region` で `en-US` を指定した場合を考えます。

```ts
const locale = new Intl.Locale("ja-JP", { language: "en", region: "US" }); //
```

この場合、ロケール識別子よりもオプションで指定した値、つまり `en-US` の方が優先されます。このような矛盾した値をセットすることは少ないでしょうが、ミスを発見する際などに役立つでしょう。

また、特定の機能の挙動を変えるオプションは Intl.Locale オブジェクトを初期化する場合以外でも指定できます。例えば、`hourCycle` などは `Intl.DateTimeFormat` でも初期化のオプションとして指定できます。それでは、Intl.Locale の初期化時と、その Intl.Locale オブジェクトを利用した各 API のオプションで異なる値を指定した場合、どちらが優先されるのでしょうか。

```ts
const jpLocale = new Intl.Locale("ja-JP", { hourCycle: "h12" }); // Intl.Locale オブジェクト初期化時は h12 を指定
const formatter = new Intl.DateTimeFormat(jpLocale, { hourCycle: "h23" }); // 利用時は h23 を指定
formatter.format(new Date());
```

この場合、Intl.Locale オブジェクトを利用する各 API のオプションの方が優先されます。したがって、上の例では `"h23"` の方が優先されて「0 時始まりの 24 時表記」になります。

まとめると、

- ロケール識別子と初期化オプションでは初期化オプションの方が優先される
- Intl.Locale オブジェクトの初期化オプションと各 API のオプションでは各 API のオプションの方が優先される

ということになります。

### ロケール識別子では細かいオプションを表現できないのか？

このように、Intl.Locale オブジェクトの初期化では一般的なロケール識別子よりも多くのオプションを指定できるように見えます。これらの情報をロケール識別子に含めることはできないのでしょうか？（できない場合、ロケール識別子は含められる情報量の面でも下位互換になってしまうのでしょうか？）

実は、Intl.Locale オブジェクトで指定できるすべてのオプションは u (Unicode) 拡張を利用してロケール識別子に含めることができます。（Unicode 拡張については[2 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-02)で解説しました）。

具体的には、各オプションがそれぞれ以下の Unicode 拡張に対応します。

- `calendar` → `u-ca-[value]`
- `collation` → `u-co-[value]`
- `numberingSystem` → `u-nu-[value]`
- `caseFirst` → `u-kf-[value]`
- `hourCycle` → `u-hc-[value]`
- `numeric` → `u-kn-[value]`

例えば、`hourCycle` オプションで `h12` を指定するのは、`u-hc-h12` のような Unicode 拡張をロケール識別子に含むのと同じ意味になります。

したがって、Intl.Locale オブジェクトの初期化においては「ロケール識別子」も「初期化オプション」も指定できる情報は同じです。（現実問題として、多くの Unicode 拡張をロケール識別子に含めることで可読性が悪くならないかは留意すべきですが）。

## Intl.Locale のプロパティ・メソッド

上記の通り、Intl.Locale オブジェクトを初期化する際には様々なオプションが指定できました。

これらの初期化時に指定した値は、オプションと同じキー名で生成した Intl.Locale オブジェクトから参照できます。

```ts
const jpLocale = new Intl.Locale("ja-JP", { hourCycle: "h12" });
console.log(jpLocale.language); // "ja"
console.log(jpLocale.region); // "JP"
console.log(jpLocale.hourCycle); // "h12"
```

初期化時のオプションで指定しなかったものについては、`numeric` を除き `undefined` として保存されています。（`numeric` はデフォルト値が `false`）

これら初期化時にオプションとして存在した 9 つのプロパティに加え、Intl.Locale オブジェクトは `baseName` というプロパティを持っています。これは初期化オプションにおける `language`、`script`、`region` またはロケール識別子の場合、変化形サブタグまでを `"-"` で繋いだ文字列を保持するプロパティです。

また、Intl.Locale オブジェクトは 10 個のプロパティ以外に、3 つのメソッドも持ちます。

- `toString()`
- `maximize()`
- `minimize()`

`toString()` メソッドは、Intl.Locale オブジェクト内に含まれた情報をすべて含むようなロケール識別子を返すメソッドです。`language`, `script`, `region` などの情報はもちろん、それ以外に指定したオプションも Unicode 拡張を使ってすべてロケール識別子内に盛り込まれます。

```ts
const jpLocale = new Intl.Locale("ja", { region: "JP", hourCycle: "h12" });
console.log(jpLOcale.toString()); // ja-JP-u-hc-h12
```

`maximize()` メソッドは、元々の Intl.Locale オブジェクトから、可能な限り `language`, `script`, `region` を補完した Intl.Locale オブジェクトを返します。ロケール識別子で例えると、`"en"` を補完したら `"en-Latn-US"` になります。

```ts
const english = new Intl.Locale("en");
console.log(english.maximize().baseName); // "en-Latn-US"
```

`minimize()` メソッドは `maximize()` メソッドの逆で、`maximize()` メソッドで補完できるような `script`, `region` の情報を可能な限り削除した Intl.Locale オブジェクトを返します。ただし、規定のロケールから補完できないようなものは削除されません。

```ts
const english = new Intl.Locale("en-Latn-US");
console.log(english.minimize().baseName); // "en" (Latn-USは補完できる)
const hant = new Intl.Locale("zh-Hant-TW");
console.log(hant.minimize().baseName); // "zh-TW" (TWでHantは補完できるが、zhだけでは補完できない)
```

## Intl.Locale オブジェクトを共有するメリット

アプリケーション内で Intl.Locale オブジェクトを共有し、Intl の各コンストラクタプロパティで利用することには次のような利点があります。

- `language`、`script`、`region` といった基本的なロケール情報を統一できる
- 各 Intl 機能のうち、ロケールに強く依存するオプションのデフォルト値を保持できる

Intl.Locale オブジェクトを共有すると、毎回ロケール識別子を使ってロケールを指定する必要がなくなり、ロケール指定ミスを防ぐことができます。また、元の Intl.Locale オブジェクトを変更するだけで簡単にアプリケーション全体のロケールを切り替えられます。

さらに、`calendar` や `hourCycle` など特定の機能の挙動を変えるオプションは、あらかじめ指定しておくことで各 Intl 機能のデフォルトオプションのように振る舞います。これは、Intl 全体のデフォルト値とは異なる挙動をアプリケーション全体で統一したいときに有用です。もちろん、上で説明した通り、これらのオプションは個別に上書きが可能です。

また、これらの Intl.Locale オブジェクトは `toString()` メソッドや `minimize()`, `maximize()` メソッドで簡単にロケール識別子へとシリアライズ可能です。これにより、サーバーとの通信などでロケール情報を送る際にも便利です。

## Intl Locale Info プロポーザル

このような Intl.Locale オブジェクトに対して、「現状のプロパティ以上の情報を取得するメソッドを追加しよう」という提案が、Intl Locale Info プロポーザルです。より具体的には、Intl.Locale オブジェクトを作成する際に指定したような情報について、各ロケールで利用可能なものをリストアップしたりより詳細な情報を提供する API を提供します。

https://github.com/tc39/proposal-intl-locale-info

2024 年 12 月現在、これは Stage 3 の提案で、すでに Chrome と Safari では利用可能です。（API が変更になる可能性があるので、利用には注意が必要です）。

### 提案されている Locale Info の getter

具体的に提案されているメソッドは以下の 7 つです。

- `getCalendars()`
  - そのロケールで使えるカレンダーの一覧を返す
  - 例: ロケール識別子が `ja-JP` なら `['gregory', 'japanese']`
- `getCollations()`
  - そのロケールで使える文字列の並べ替えのパターンを返す
  - 例: ロケール識別子が `ja-JP` なら `['emoji', 'eor', 'unihan']` (`'unihan'` は漢字の部首の画数順で並ぶ)
- `getHourCycles()`
  - そのロケールで使える時刻表示の一覧を返す
  - 例: ロケール識別子が `ja-JP` なら `['h23']`
- `getTimeZones()`
  - そのロケールで使えるタイムゾーンの一覧を返す
  - 例: ロケール識別子が `ja-JP` なら `['Asia/Tokyo']`
- `getWeekInfo()`
  - そのロケールにおける曜日の法則
    - `firstDay`（週の最初の曜日）、`weekend`（週末の曜日）、`minimalDays`（月または年の第 1 週に最低限必要な日数）を返す
  - 例: ロケール識別子が `ja-JP` なら `{"firstDay": 7, "weekend": [6, 7], "minimalDays": 1}`
- `getNumberingSystems()`
  - そのロケールで使える命数法の一覧を返す
  - 例: ロケール識別子が `ja-JP` なら `['h23']`
- `getTextInfo()`
  - そのロケールにおける文字の方向を `ltr` (left-to-right) か `rtl` (right-to-left)で返す
  - 例: ロケール識別子が `ja-JP` なら `{"direction": "ltr"}`

これにより、より細かいレベルでのロケールに応じた処理が実現できるようになりますね。

## まとめと次回予告

今回は Intl.Locale オブジェクトについて、その生成方法とプロパティ・メソッドなどについて解説しました。また、このような Intl.Locale オブジェクトをより便利にするための提案、Intl Locale Info プロポーザルについても解説しました。次回 [4 日目]()では「ロケールネゴシエーション」と Intl.LocaleMatcher プロポーザルについて解説します。

## 参考文献

- Intl Locale Info proposal
  - https://github.com/tc39/proposal-intl-locale-info
