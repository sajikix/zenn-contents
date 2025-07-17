---
title: "自力で時差計算をしろと言われたら。あるいはDSTによる変換の曖昧性について。"
emoji: "🕰️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["JavaScript", "frontend", "Temporal", "Date"]
publication_name: "cybozu_frontend"
published: true
---

:::message
この記事は、[CYBOZU SUMMER BLOG FES '25](https://cybozu.github.io/summer-blog-fes-2025/)の記事です。
:::

## 自力で計算なんてしたくはないけども

皆さんは自力で UTC と特定のタイムゾーンの間の時差の計算をしたことはありますか？

そもそも、システムのタイムゾーンと UTC の時差を計算するだけなら Date オブジェクトを利用すれば簡単に計算できますから、自力で計算する必要はありませんね。

```javascript
const date = new Date();
const utcOffset = date.getTimezoneOffset(); // 分単位で取得
console.log(`UTCとの時差: ${utcOffset}分`);
```

ではもし計算したい時差が「システムのタイムゾーンと異なるタイムゾーン」だったらどうでしょうか?

JS の`Date`オブジェクトは「UTC ↔︎ システムのローカルタイムゾーン」での相互変換しかサポートしていないので、他のタイムゾーンの時差を計算する場合はライブラリを利用することが一般的です。例えば、[date-fns](https://date-fns.org/)であれば、[date-fns-tz](https://github.com/marnusw/date-fns-tz)という拡張パッケージを利用することで、特定のタイムゾーンの時差を考慮した日時を計算できます。

```javascript
import { format, toZonedTime } from "date-fns-tz";

const date = new Date("2025-07-17T03:00:00Z");

const jpDate = toZonedTime(date, "Asia/Tokyo");
const parisDate = toZonedTime(date, "Europe/Paris");
```

大抵の場合はこのようにライブラリを利用することでカバーできますが、ごくたまに自力で計算しないといけない場合があります。(もちろんそうなっている時点で良い設計でない可能性は高いですが)。考えられる可能性としては、「バックエンドが利用しているタイムゾーンデータ」と「一般的なライブラリや Date/Intl が参照しているタイムゾーンデータ」が異なるにもかかわらずフロント側で時差の計算をしなければならない場合などです。

こうなると、バックエンドのタイムゾーンのデータを参照するなどして自力で時差の計算する必要が出てきます。またそこまで悲惨な状況になってなかったとしても、日時というドメインを理解する上で時差の計算方法を知っておくことは損にはならないでしょう。

この記事では、そんな時差の計算を自力で行う方法を解説します。

## 自力で計算するための基礎知識

まずは自力で時差の計算をするための基礎知識をおさらいしましょう。時差計算の実装を行うには大きく以下の 2 つの知識が必要です。

- そもそも時差のデータはどこでどのように定義されているのか
- 時差の変更(≒ サマータイム)はどのようにされるのか

### ブラウザのタイムゾーンのデータはどこから？

ブラウザやライブラリの利用するのタイムゾーンのデータは、[IANA Time Zone Database](https://www.iana.org/time-zones)というデータベースが元になっています。このデータベースは、世界中のタイムゾーンに関する情報を提供しており、各タイムゾーンの標準時とサマータイムの開始・終了日時、タイムゾーンの変更記録などが含まれています。例として日本標準時のデータベースの一部は以下のようになっています。

```
# Rule	NAME	FROM	TO	-	IN	ON	AT	SAVE	LETTER/S
Rule	Japan	1948	only	-	May	Sat>=1	24:00	1:00	D
Rule	Japan	1948	1951	-	Sep	Sat>=8	25:00	0	S
Rule	Japan	1949	only	-	Apr	Sat>=1	24:00	1:00	D
Rule	Japan	1950	1951	-	May	Sat>=1	24:00	1:00	D

# Zone	NAME		STDOFF	RULES	FORMAT	[UNTIL]
Zone	Asia/Tokyo	9:18:59	-	LMT	1887 Dec 31 15:00u
			9:00	Japan	J%sT
```

ブラウザの場合、この IANA Time Zone Database のデータを含んだ[CLDR](https://cldr.unicode.org/)や ICU の機能を経由してタイムゾーンの情報を利用しています。

「CLDR や ICU という名前を聞いたことないよ」という方は、以前の別の記事で解説しているので、そちらもぜひ読んでみてください。

https://zenn.dev/sajikix/articles/intl-advent-calendar-24-06

### サマータイム(DST)について復習

また、サマータイムについても復習しておきましょう。サマータイム(DST:daylight saving time)は 1 年のうち日中の時間が長くなる夏を中心とする時期に、日中の明るい時間を有効利用するため、時計を通常よりも進めることで、日が暮れる時刻を遅らせる時間制度のことです。基本的には春頃に標準時から時間を１時間進めて、秋に 1 時間戻す操作を行います。

より具体的には以下のような調整をそれぞれ春と秋に行います。

#### サマータイム開始時(春頃)

サマータイム開始時には時間を１時間進めるので、1 日が１時間短くなります。したがって実施日の夜中のどこかの１時間が以下のように無くなります。

| 現地時刻 | 時差   |
| -------- | ------ |
| 01:58    | -08:00 |
| 01:59    | -08:00 |
| 03:00    | -07:00 |
| 03:01    | -07:00 |

#### サマータイム終了時(秋頃)

サマータイム終了時には時間を１時間戻すので、1 日が１時間長くなります。したがって実施日の夜中のどこかの時刻が以下のように２回来ることになります。

| 現地時刻      | 時差   |
| ------------- | ------ |
| 00:58         | -07:00 |
| 00:59         | -07:00 |
| 01:00(１回目) | -07:00 |
| 01:59(１回目) | -07:00 |
| 01:00(２回目) | -08:00 |
| 01:01(２回目) | -08:00 |

## 時差計算をしてみよう

サマータイムの仕組みやタイムゾーンのデータを理解したところで、実際に時差を計算してみましょう。

### タイムゾーンのデータ形式

今回は UTC における任意のタイミング・任意のタイムゾーンの時差を返すことができるデータがあると仮定します。もちろんサマータイムや時差自体の変更も記録しているとします。

| 開始(UTC)            | 終了(UTC)            | 時差(秒単位) |
| -------------------- | -------------------- | ------------ |
| 1970-01-01T00:00:00Z | 1970-03-29T06:59:59Z | -28800       |
| 1970-03-29T07:00:00Z | 1970-10-25T05:59:59Z | -25200       |
| 1970-10-25T06:00:00Z | 1971-04-25T06:59:59Z | -28800       |
| 1971-04-25T07:00:00Z | 1971-10-31T05:59:59Z | -25200       |
| 1971-10-31T06:00:00Z | 1972-04-30T06:59:59Z | -28800       |
| 1972-04-30T07:00:00Z | 1972-10-29T05:59:59Z | -25200       |

似たような形式として、タイムゾーンごとに通常時差の期間とサマータイム期間の長さを交互に記録したデータとして保持している場合もあります。

```javascript
// 1970/01/01 00:00:00 UTC からの時差を記録している例
const transitions = [
  259200, // サマータイム(分)
  266400, // 通常時(分)
  259200, // サマータイム(分)
  266400, // 通常時(分)
  259200, // サマータイム(分)
  266400, // 通常時(分)
  // ...
];
```

また、先述した[date-fns-tz](https://github.com/marnusw/date-fns-tz)では、Intl の DateTimeFormat を利用してタイムゾーンのデータを取得するという方法をとっています。Intl.DateTimeFormat を経由してブラウザのタイムゾーンのデータを利用することで、タイムゾーンデータによるバンドルサイズ増加や追従のコストを抑えているわけです。

https://github.com/marnusw/date-fns-tz/blob/4f3383b26a5907a73b14512a2701f3dfd8cf1579/src/_lib/tzTokenizeDate/index.ts#L19-L37

このようにタイムゾーンデータをどのように保持・取得するかは様々ですが、「UTC 時刻とタイムゾーンから時差を得る」という挙動自体は本質的にはどの方式も変わりません。ここで大事なのは、以下のように「UTC 時刻とタイムゾーンから適切な時差を返すことができる関数」が実装できるという前提です。

```javascript
function getOffset(utcDate, timeZone) {
  // utcDate: UTCの日時
  // timeZone: タイムゾーン名（例: "Asia/Tokyo"）
  // 戻り値: utcDateで指定したタイミングでtimeZoneで指定タイムゾーンの時差（秒単位）
}
```

### そこまで計算が難しくなさそうなケース

#### UTC → ローカルタイム書式化は Intl.DateTimeFormat で簡単にできる

UTC からローカルタイムゾーンへの変換は、ライブラリを利用せずとも`Intl.DateTimeFormat`を利用することで行えます。具体的には、インスタンスを生成するときの`timeZone`オプションを指定することで、任意のタイムゾーンのローカル時間に書式化することができます。formatToParts メソッドを利用すれば、書式化された日時の各部分を取得することもできます。

```javascript
const utcDate = new Date("2025-07-17T03:00:00Z");
const formatter = new Intl.DateTimeFormat("ja-JP", {
  timeZone: "Asia/Tokyo",
  dateStyle: "short",
  timeStyle: "short",
});
const jpDateString = formatter.format(utcDate); // '2025/7/17 12:00:00'
const jsDateParts = formatter.formatToParts(utcDate);
// [
//   { type: "year", value: "2025", },
//   { type: "literal", value: "/", },
//   { type: "month", value: "07", },
//   { type: "literal", value: "/", },
//   { type: "day", value: "17", },
//   { type: "literal", value: " ", },
//   { type: "hour", value: "12", },
//   { type: "literal", value: ":", },
//   { type: "minute", value: "00", },
// ];
```

詳しい Intl.DateTimeFormat のオプションについては以前記事を書いているのでぜひ読んでみてください。

https://zenn.dev/sajikix/articles/intl-advent-calendar-24-08

#### ローカルタイム → UTC はサマータイムのないタイムゾーンなら簡単

サマータイムのないタイムゾーンでは時差の変更がない限り UTC との時差は一定ですから計算は比較的単純です。例として日本のローカル時間を UTC に変換する場合を考えてみましょう。

:::details 日本 ↔︎ UTC も過去まで遡ると一定じゃない
厳密な話をすると日本でも 1948 年から 51 年まで 3 年間サマータイムを導入していたので、「過去のすべてのタイミングで時差が９時間で一定」とは言えません。
また、さらに遡ると 1887 年以前の時差は`09:18:59`として計算されます。（JS の Date でもこう計算されるので試してみてください）。これは明石が日本の標準時として決まる以前の日時に対して「東京の標準時（地方時）」を日本の標準時として扱っているためです。
:::

日本と UTC の時差は基本一定ですから、以下のように計算すれば日本時間から UTC が得られます

1. UTC と日本の時差を取得（９時間）
2. 日本のローカル時間に「Z」をつけた UTC 時間を生成
3. 2 は時差分ずれている UTC なので、９時間マイナスする

実際にコードで見てみましょう。

```javascript
const localDate = {
  year: "2025",
  month: "07",
  day: "17",
  hour: "12",
  minute: "00",
};
const localeDateAsUtc = new Date(
  `${localDate.year}-${localDate.month}-${localDate.day}T${localDate.hour}:${localDate.minute}:00Z`
); // ローカル時間をそのままUTCにした日時（= 時差分進んでる）
const offsetInSecond = getOffset(new Date(localeDateAsUtc), "Asia/Tokyo"); // - 9 * 60 * 60 秒

const utcDate = new Date(localeDateAsUtc.getTime() - offsetInSecond * 1000); // ローカル時間そのままUTCから９時間引く
utcDate.toISOString(); // '2025-07-17T03:00:00.000Z'
```

#### サマータイムのあるタイムゾーンでも各期間中は簡単

またサマータイムのあるタイムゾーンであっても、それぞれの期間中であれば getOffset が適切な時差を返してくれるので、ローカルタイム → UTC の変換は上記の日本のケースと同様に簡単に計算できます。

```javascript
// 略

// 8 * 60 * 60 秒（サマータイム期間中なので）
// - もしサマータイム期間外なら 7 * 60 * 60 秒が返される
const offsetInSecond = getOffset(
  new Date(localeDateAsUtc),
  "America/Los_Angeles"
);

// 略
```

### これで OK? (ではない)

これで「サマータイムがない場合」も「サマータイムがある場合」も、UTC↔︎ ローカルタイムゾーンの計算が実装できたように見えますが、果たして完璧に実装できたと言えそうでしょうか？

残念ながらこれらの実装では「サマータイムの終了・開始時におけるローカル →UTC 変換の曖昧性」という問題を考慮できていません。そしてこの「曖昧性問題」が時差計算を難しくしている一番の原因とも言えます。

## サマータイムによる変換の曖昧性

前述の通り、サマータイム開始日は特定の時間が 1 時間無くなり、逆に終了日には 1 時間増えることになります。そのため、サマータイムの開始・終了時においてはどうしても「どちらと解釈するべきか曖昧」な時間が発生します。

- サマータイム開始日の「なくなる 1 時間」を指定してしまった場合
  - 例：開始日に 2 時が無くなるとする → 開始日の 02:30 を指定した場合の時差はサマータイム適用範囲？
- サマータイム終了日の２回ある時刻を指定してしまった場合
  - 例：終了日の 1 時が２回くるとする → 終了日の 01:30 を指定した場合の時差はサマータイム適用範囲？

曖昧になってしまう以上、サマータイムとして扱うか、通常時として扱うかを選ぶ必要がありますが、Date や各ライブラリなどでは概ね以下のような挙動をとります。

- サマータイム開始時 : 無くなった時刻を指定した場合サマータイムとして扱う
- サマータイム終了時 : ２回訪れる時刻を指定した場合、最初の方の時差(=サマータイム時差)として扱う

つまり、「曖昧な時刻はサマータイム側の時差になるとして解釈する」というのが Date やメジャーなライブラリの挙動です。また、この挙動は RFC 5545 (iCalendar)の挙動にも一致します。

### どうやって求めるか

ここからは上記の挙動を元に、実際にサマータイムの開始時・終了時の補正を加味した時差計算を考えていきましょう。

そもそもサマータイムのあるタイムゾーンではローカル時間から正確な時差を得ることが出来ません(サマータイムの開始・終了時の時差が曖昧なため)。そのため一度「仮の時差」を設定し、その時差を使った「計算の整合性チェック」を行うことでサマータイムの開始・終了時かどうかを見極めながら計算するという手法を取ります。

「仮の時差」を決める方法はいろいろ考えられますが、ここでは「変換したいローカル日時と同じ日付・時刻の値を持った UTC 日時」から得るものとします。

- 例　ローカル時間が　 2025/03/09 02:30 の場合
  - 2025-03-09T02:30:00Z という UTC から「仮の時差」を計算
  - ↑ ローカル日時と同じ日付・時刻の値を持っている

それぞれサマータイム開始時と終了時での計算方法を見ていきましょう。

#### サマータイム開始時

サマータイムの開始時の場合、上記の方法で取得した「仮の時差」は、**通常時の時差**になります。一方 Date オブジェクト やメジャーなライブラリの挙動に従うのであれば、「サマータイム開始時になくなる時刻」を指定した場合は、「**サマータイム期間中**」として扱うことになります。よって「サマータイム開始時になくなる時刻」を指定した場合、「仮の時差」と「仮の時差から計算した仮の UTC(以降仮の UTC)から計算した時差」でずれが生じることになります。

![仮の時差と仮の UTC から計算した時差がずれる](/images/calcTimeDiffOnYourOwn/summertime-start-1.png)

この場合、正しい時差は「仮の UTC から計算した時差」の方であることが図からもわかると思います。(この例では、-７時間が正しい時差ですね。)

しかしこれだけでは、「サマータイム開始時になくなる時刻（上図赤部分）」と「サマータイムが始まった以降の時刻（上図青部分）」の区別がつきません。下図のように、サマータイム開始以降も数時間の間（図の例では 7 時間）は「仮の時差」の算出値が**サマータイム開始前の時差**になってしまうからです。

![サマータイム開始後でも仮の時差はずれる](/images/calcTimeDiffOnYourOwn/summertime-start-2.png)

そこでさらに、「正しい時差 - 仮の時差」の分だけ仮の UTC を戻した UTC で時差を計算し、それが「正しい時差」と一致するかを確認します。もし一致しない場合、元のローカル時間はちょうど「サマータイム開始時になくなる時刻」を指定していたことがわかります。

![サマータイム開始時になくなる時刻](/images/calcTimeDiffOnYourOwn/summertime-start-3.png)

逆に一致した場合は、元のローカル時間は「サマータイム開始以降の時刻」を指定していたことがわかります。

![サマータイム開始以降の時刻](/images/calcTimeDiffOnYourOwn/summertime-start-4.png)

サマータイム開始時の計算方法についてまとめると、以下のパターンに分けられます。

- サマータイム開始前
  - 「仮の時差」と「仮の UTC から計算した時差」は同値
  - 正しい時差：「仮の時差」（=「仮の UTC から計算した時差」）
  - 変換されるべき UTC 日時：ローカル時間 + 「仮の時差」
- サマータイム開始時になくなる時刻
  - 「仮の時差」と「仮の UTC から計算した時差」が異なる
  - 正しい時差：「仮の UTC から計算した時差」
  - 変換されるべき UTC 日時：ローカル時間 + 「仮の時差」
- サマータイム開始以降の時刻
  - 数時間の間は「仮の時差」と「仮の UTC から計算した時差」が異なる
  - 正しい時差：「仮の UTC から計算した時差」
  - 変換されるべき UTC 日時：ローカル時間 + 「仮の UTC から計算した時差」

#### サマータイム終了時

同様にサマータイム終了時の場合も考えてみましょう。サマータイム終了時は、上記の方法で取得した「仮の時差」は、**サマータイム中の時差**になります。また Date オブジェクト やメジャーなライブラリの挙動に従うのであれば、「サマータイム終了時に２回現れる時刻」を指定した場合は、サマータイム期間として扱うことになります。よってサマータイムの終了時に２回現れる時刻を指定した場合、「仮の時差」と「仮の時差から計算した仮の UTC(以降仮の UTC)から計算した時差」の間でずれは生じず、「仮の時差」がそのまま正しい時差になります。

![サマータイム終了時の仮の時差](/images/calcTimeDiffOnYourOwn/summertime-end-1.png)

しかし、今度はサマータイム終了後で「仮の時差」と「仮の UTC から計算した時差」の間にずれが生じてしまいます。この場合正しいのは「仮の UTC から計算した時差」の方です。

![サマータイム終了後の仮の時差と仮の UTC から計算した時差のずれ](/images/calcTimeDiffOnYourOwn/summertime-end-2.png)

そのため、サマータイム終了時の計算方法は以下の２パターンに分けられます。

- サマータイム終了前 & サマータイム終了時に２回現れる時刻
  - 「仮の時差」と「仮の UTC から計算した時差」は同値
  - 正しい時差：「仮の時差」
  - 変換されるべき UTC 日時：ローカル時間 + 「仮の時差」
- サマータイム終了後
  - 数時間の間は「仮の時差」と「仮の UTC から計算した時差」が異なる
  - 正しい時差：「仮の UTC から計算した時差」
  - 変換されるべき UTC 日時：ローカル時間 + 「仮の UTC から計算した時差」

### 実装してみよう！

最後に、上記の計算方法を踏まえて実装に落とし込んでみましょう。前提として、以下のようなデータと関数があるとします。

- 変換したいローカルの日時データ（年月日、時分で分かれている）：`localDate`
- UTC の時刻とタイムゾーンから時差（秒）を取得する関数：`getOffset(utcDate, timeZone)`

まず全体像ですが、今回は`localDate`と`timeZone`を引数に取り、変換後の UTC 日時文字列と正しい時差を返す関数、`getCorrectOffsetAndUtcString(localDate, timeZone)`として時差計算の処理を実装します。

```javascript
// ローカルの日時データ
const localDate = {
  year: "2025",
  month: "03",
  day: "09",
  hour: "02",
  minute: "30",
};

// タイムゾーン名
const timeZone = "America/Los_Angeles";

// UTCの時刻とタイムゾーンから時差を取得する関数
const getOffset = (utcDate, timeZone) => {
  // すでに実装されているものとする
  // - 単位は秒
  // - 東半球の時差の場合は負の値が返る
  // - 例: "Asia/Tokyo" なら -9 * 60 * 60 秒
};

const getCorrectOffsetAndUtcString = (localDate, timeZone) => {
  // ...ここに実装していく
};

getCorrectOffsetAndUtcString(localDate, timeZone);
```

実際に`getCorrectOffsetAndUtcString`関数の中身を実装してみると以下のようになります。(コードとドキュメントを分けると読みづらいので以降はコードの中にコメントとして解説を入れていきます。)

```javascript
const getCorrectOffset = (localDate, timeZone) => {
  // 「仮の時差」を求めるための「変換したいローカル日時と同じ日付・時刻の値を持ったUTC日時」
  const localeDateAsUtc = new Date(
    `${localDate.year}-${localDate.month}-${localDate.day}T${localDate.hour}:${localDate.minute}:00Z`
  );

  // 「仮の時差」を取得
  const offsetGuess = getOffset(localeDateAsUtc, timeZone);

  // 「仮の UTC」を計算
  const utcGuess = new Date(localeDateAsUtc.getTime() + offsetGuess * 1000); // 2025-03-09T09:30:00.000Z

  // 「仮の UTC」から再度時差を取得
  const offsetFromUtcGuess = getOffset(utcGuess, timeZone);

  // 「仮の UTC」から計算した時差と「仮の時差」を比較
  if (offsetFromUtcGuess === offsetGuess) {
    // 「仮の UTC」から計算した時差と「仮の時差」が一致する場合
    // これは以下の可能性がある
    // - サマータイム開始前
    // - サマータイム終了時に２回現れる日時
    // - サマータイム終了の数時間後以降

    // どの場合においても
    // - 正しい時差: 「仮の時差」
    // - 変換されるべきUTC日時: ローカル時間 + 「仮の時差」
    // なので
    return {
      utcString: utcGuess.toISOString(),
      correctOffset: offsetGuess,
    };
  }

  // 「仮の UTCから計算した時差」と「仮の時差」がずれている場合
  // これは以下の可能性がある
  // - サマータイム開始時になくなる時刻
  // - サマータイム開始後の数時間
  // - サマータイム終了後の数時間

  // この場合、「仮の時差」-「仮のUTCから計算した時差」の分だけ仮のUTCから戻してさらに時差を計算する
  // - サマータイム終了時は仮のUTCから「進める」必要があるが、
  // - この場合「仮の時差」-「仮のUTCから計算した時差」が負になるので同じ式で計算できる。
  const offsetDiff = offsetGuess - offsetFromUtcGuess; // 仮の時差とずれた計算した時差の差分
  const utcGuessFromOffsetDiff = new Date(
    utcGuess.getTime() - offsetDiff * 1000
  ); // 仮のUTCから戻した（進めた）UTC
  const offset2 = getOffset(utcGuessFromOffsetDiff, timeZone); // さらに時差を計算

  if (offset2 === offsetFromUtcGuess) {
    // 「仮の UTC」から戻した（進めた）UTCから計算した時差と「仮のUTCから計算した時差」が一致する場合
    // これは以下の２通りに絞られる
    // - サマータイム開始後の数時間
    // - サマータイム終了後の数時間
    // どちらも「仮のUTCから計算した時差」が正しい時差になるので以下のように返す。
    return {
      utcString: new Date(
        localeDateAsUtc.getTime() + offsetFromUtcGuess * 1000
      ).toISOString(),
      correctOffset: offsetFromUtcGuess,
    };
  }

  // ここで最後に残るのは「サマータイム開始時になくなる時刻」の場合のみ
  // この場合は
  // - 正しい時差：「仮の UTC から計算した時差」
  // - 変換されるべきUTC日時：ローカル時間 + 「仮の時差」
  // なので以下のように返す
  return {
    utcString: utcGuess.toISOString(),
    correctOffset: offsetFromUtcGuess,
  };
};
```

これで、サマータイムの開始・終了時における変換の曖昧性を解決しつつ、UTC とローカルタイムゾーンの間の時差を計算することができました。

## 【おまけ】Temporal の `disambiguation` オプションについて

ここまで、自力で時差計算を行う方法を見てきましたが、`Temporal` が使えるようになれば、今までライブラリを使っていた方も標準機能で時差計算ができるようになります。

:::message
Temporal は JavaScript の仕様として提案されている日付・時刻を扱うための新しい API です。

詳細は [MDN の Temporal のドキュメント](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal) を参照してください。

また拙作ではありますが、[Temporal で変わる JavaScript の日時操作](https://gihyo.jp/article/2025/02/ride-modern-frontend-07)という記事も書いているので合わせて読んでもらえると嬉しいです。
:::

しかも、Temporal ではサマータイムの開始・終了時における変換の曖昧性の解決方法を`disambiguation`オプションで指定することができます。

- `compatible`
  - デフォルト値。Date やメジャーなライブラリと同様に「サマータイム側の時差で解釈する」。今回の記事での実装と同じ挙動。
- `earlier`
  - 開始終了どちらも変更前の時差で解釈する。`compatible`のサマータイムの終了時の挙動は`earlier`と同じ。
- `later`
  - 開始終了どちらも変更後の時差で解釈する。`compatible`のサマータイムの開始時の挙動は`later`と同じ。
- `reject`
  - 曖昧な時刻を指定した場合は例外を投げる。

初めて Temporal を使う方は、`disambiguation`オプションの説明を見てもピンとこないかもしれませんが、ここまでの記事を読んでいただけた皆様であればどのような挙動をするかイメージできるはずです。

これらのオプションについては以下の記事も詳しいので、ぜひ読んでみてください。

- [Ambiguity and gaps from local time to UTC time | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/ZonedDateTime#ambiguity_and_gaps_from_local_time_to_utc_time)
  - `disambiguation`オプションの挙動について詳しく書かれている MDN のドキュメントです。
- [Time Zones and Resolving Ambiguity](https://tc39.es/proposal-temporal/docs/timezone.html)
  - 仕様策定の段階のドキュメントですが、`disambiguation`オプションの挙動について実例を踏まえて詳しく書かれています。(今後の仕様変更で古い情報になる可能性があるので注意)

## まとめ

この記事では、「時差の計算」を自力で行うことによって、サマータイムの仕組みやサマータイムによる変換の曖昧性について理解を深めることを目指しました。ここにある実装とそのイメージが理解できていれば、今後面倒な時差の計算に直面してもきっと乗り越えることができると信じています。（そして「サマータイム」という仕組みが IT システムで再現する上でいかに面倒か、より解像度高く想像できるようになったのではないでしょうか。）

## 参考リンク

- [MDN - Temporal](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal)
- [MDN - Intl.DateTimeFormat](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat)
- [date-fns-tz](https://github.com/marnusw/date-fns-tz)
- [Time Zones and Resolving Ambiguity](https://tc39.es/proposal-temporal/docs/timezone.html)
