---
title: "Temporalの近況(主にScopeを狭める話)"
emoji: "🕰️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["JavaScript", "ECMAScript", "frontend"]
publication_name: "cybozu_frontend"
published: false
---

## Temporal についておさらい

Temporal は新しく JavaScript の仕様として提案されている「日時を操作するための新しいグローバルオブジェクト」です。現在は Stage3 のプロポーザルとして細かい API などの形が協議されています。

https://github.com/tc39/proposal-temporal

Temporal は既存の Date オブジェクトにある次のような課題を解決するべく提案されました。

- ユーザーの現地時間と UTC 以外のタイムゾーンはサポートされない
- パーサーの動作の信頼性が低い
- 日付オブジェクトが何を指しているのかわかりづらい
- サマータイム動作が不安定
- 日時計算の API が扱いにくい
- 非グレゴリオ暦がサポートされていない

Temporal ではこれらの問題に対して以下のような機能を追加することで解決しようとしています。

- Wall-Clock Time と Exact Time の明確な分離
- タイムゾーンとカレンダーのサポート
- 豊富な計算 API
- 明確な日時文字列(タイムスタンプ)の定義とパース処理

### Wall-Clock Time と Exact Time の明確な分離

Wall-Clock Time とはカレンダーやタイムゾーンに依存した日時・時刻のことで、Exact Time は地球上のどこでも同じである正確な時間(=UTC 時間)のことを指します。

- Wall-Clock Time
  - ある場所の特定の時間を指すローカルな時間
  - サマータイム制度や国によるタイムゾーンの変更などにより変更されることがある
- Exact Time
  - 全世界共通の正確な時間
  - 1970 年 1 月 1 日 00:00 UTC の Unix エポックからの経過で管理する

Temporal ではこの２つのうちのどちらを表しているか(あるいは両方の情報を持っているか)を API レベルで明確に区別します。`Temporal.PlainDateTime` のように `Plain〇〇` とつくクラスは全て Wall-Clock Time を表し、タイムゾーンなどの情報を持っていません。逆に `Temporal.Instant` は Exact Time の情報しか持っていません。このように名前レベルで Wall-Clock Time なのか Exact Time なのか使用者が意識できる設計になっています。

Wall-Clock Time と Exact Time を相互変換するには、「Wall-Clock Time と Exact Time にどのくらいの時差があるのか」と「Wall-Clock Time が Exact Time と同じカレンダーシステムを利用しているか」を知る必要があります。つまりタイムゾーンとカレンダーの情報がないと Wall-Clock Time と Exact Time を相互変換は不可能なわけです。

そのため、Temporal には `Temporal.ZonedDateTime` という Exact Time の情報とタイムゾーン・カレンダーの情報を併せ持ったデータ型が存在します。この `Temporal.ZonedDateTime` を介してのみ Wall-Clock Time と Exact Time の相互変換が可能になります。

![](https://tc39.es/proposal-temporal/docs/object-model.svg)
_各クラスの関係図(Temporal の Doc より)_

:::details 変換の曖昧性について
ちなみに Exact Time → Wall-Clock Time の変換は一意に定まりますが、Wall-Clock Time → Exact Time の変換は「サマータイム解決の曖昧さ」と「タイムゾーン定義の変更可能性」から曖昧性が生じます。Temporal ではこのような曖昧さをなるべく吸収できるようにタイムゾーンの競合や欠如に対してどう曖昧さを解消するか指定できるオプションが存在します。
:::

### 豊富な日付計算 API

Temporal の `PlainDateTime` や `ZonedDateTime` には今まで日時操作ライブラリで行っていたような `add`/`subtract`/`until`/`since` といった計算 API が存在します。

```js
dt1 = Temporal.PlainDateTime.from("2024-12-07T03:24:30.000003500");
dt2 = Temporal.PlainDateTime.from("2019-01-31T15:30");
dt2.since(dt1);
```

### タイムゾーンとカレンダーのサポート

Temporal では Wall-Clock Time と Exact Time を変換するためにタイムゾーンとカレンダーの情報が必要です。そのため Date では曖昧だったタイムゾーンとカレンダーを明確にサポートしています。

タイムゾーンは IANA Time Zone Database に準拠しており、`Asia/Tokyo` のような IANA Time Zone の ID から利用できます。

```js
const inTokyo = utc.withTimeZone("Asia/Tokyo");
```

同様にカレンダーでは `Intl.DateTimeFormat` でサポートされているカレンダーシステムの範囲でサポートしています。

```js
date.withCalendar("hebrew");
```

### 明確な日時文字列(タイムスタンプ)の定義とパース処理

既存の Date オブジェクトの問題点として文字データからパースする処理の信頼性が低いことやタイムスタンプにカレンダーやタイムゾーンの情報を入れることができないと言う問題がありました。

タイムスタンプの標準としては ISO-8601 および RFC 3339 が業界標準となっています。ISO-8601 は日付と時刻に `-` 区切りを入れない基本形式と区切りを入れる拡張形式があり、ECMAScript や HTMLStandard の準拠する RFC 3339 はこの ISO-8601 の拡張形式元にしています。

```
20240101T123456Z // 基本形式
2024-01-01T12:34:56Z // 拡張形式
```

Temporal ではこの RFC 3339 を拡張し以下の２つのタグ(`[]` で囲まれた部分の総称)を追加したフォーマットを入出力としてサポートします。

- タイムゾーン : `[Asia/Tokyo]` (例 : 東京のタイムゾーンを指定する場合)
- カレンダー情報 : `[u-ca=japanese]` (例 : 和暦を利用する場合)

これらを組み合わせると `2024-01-01T12:34:56+09:00[Asia/Tokyo][u-ca=japanese]`　のようになります。

## 2024 前半のアップデート

このような Temporal に関して 2024 年 4 月以降入った大きめのアップデートを２つ紹介します。

### 新しいタイムスタンプの構文 RFC 9557 が Approve された

前項で説明した Temporal が拡張したタイムスタンプの形式が RFC 9557 : Date and Time on the Internet: Timestamps with Additional Information として Approve されました。RFC 9557 は RFC 3339 互換でありながらタイムゾーンやその他 key-value 情報をタグ形式で追加できるようになっており、Internet Extended Date/Time Format (IXDTF)と呼ばれます。

この仕様に至る経緯や他のタイムスタンプの仕様との関係など、より詳しくは以下の記事を参考にすると良さそうです。
https://zenn.dev/pixiv/articles/23b726da2236cd

### Custom TimeZone と Custom Calendar を一旦削減予定

もう１つ、 Temporal 仕様策定中での大きな変更点として Custom TimeZone と Custom Calendar を一度仕様から落とす方向へ進むという話がありました。

https://github.com/tc39/proposal-temporal/issues/2826

#### `Temporal.TimeZone` と `Temporal.Calendar`

Temporal の提案では `Temporal.TimeZone` と `Temporal.Calendar` というクラスが存在しています(2024/06 現在)。既存のタイムゾーンやカレンダーを指定するだけであれば識別子文字列での指定が可能ですが、以下のようなケースでこれらのクラスを利用することが想定されています。

- 指定するタイムゾーンやカレンダーをカスタマイズしたい場合
- 特定のタイムゾーンやカレンダーの情報を共通化して管理したい場合

特に１つ目の用途を想定し、 `Temporal.TimeZone` と `Temporal.Calendar` には細かく時差や暦のシステムを設定できる API が用意されています。

#### Temporal のコードサイズ・パフォーマンス問題

現在の Temporal では Stage4 へ提案を進める上で、実装時のコードサイズとメモリ使用量が問題になっています。

JS が実行される環境は幅広く、特に低スペックのモバイル端末やスマートウォッチなどで実行される場合は許容されるコードサイズやメモリサイズは限られています。また広告の埋め込まれた iframe が多く表示されるような環境では JS の実行環境ごとにコードサイズやメモリサイズ増加の影響を受けます。このような事情を考慮した結果、Temporal 全体として実装するメソッドやクラスを削減し、コードサイズや使用メモリサイズを削減するように求められているようです。

<!-- 要 note -->

#### `Temporal.TimeZone` と `Temporal.Calendar` が対象になった理由

`Temporal.TimeZone` と `Temporal.Calendar` が対象になった理由として以下のような理由が挙げられています。

- 使用されるユースケースが少ない
- 機能が複雑である

そもそも Temporal で既存のタイムゾーン・カレンダーを指定する場合、それぞれの識別子を用いて指定する方法が推奨されています。

```js
const inTokyo = utc.withTimeZone("Asia/Tokyo");
date.withCalendar("hebrew");
```

そのため `Temporal.TimeZone` と `Temporal.Calendar` を使うユースケースは主にカスタムされたカレンダーやタイムゾーンを使う時などに限られています。このようにユースケースが限定的な割に機能も複雑(特に `Calendar`)であることから今回削減の対象として検討されたようです。

#### 今後

今回は一度 `Temporal.TimeZone` と `Temporal.Calendar` を削除する方向で話は進みそうですが、これは恒久的な決定でないとも言及されています。あくまで Stage4(仕様としての採択)を早く目指すために落とす考えのようで、以下のようなことも同時に考えられています。

- 今後改めて `Temporal.TimeZone` と `Temporal.Calendar` を追加できるような余地を残しておくべきであること
- 他に提案されている仕様に準拠したデータ形式で自由度の高いタイムゾーンやカレンダーを指定する方法を探るべきであること

また純粋に今後 JS エンジンの最適化によってコードサイズの問題やパフォーマンス問題が改善する可能性もあります。

## その他削減される API など

`Temporal.TimeZone` と `Temporal.Calendar` 以外でも前述のコードサイズ削減の観点から API を削る予定のようです。

以下のような簡単なワークアラウンドで代替可能なヘルパーメソッド群も削除される可能性があります。

- `PlainDateTime` や `ZonedDateTime` にある `toPlainYearMonth()` と `toPlainMonthDay()`
- `PlainDateTime` や `ZonedDateTime` にある `withPlainDate()`
- etc...

また `valueOf` と `toJSON` の実装を共通化することで効率化することも検討しています。これらの削減と前述の `Temporal.TimeZone` 、 `Temporal.Calendar` を合わせて全体の 1/3 ほどの API を削る予定があるようです。

## まとめ

この記事では Temporal についてのおさらいと 2024 年 4 月以降の近況についてまとめました。具体的には以下のようなアップデートが行われています。

- Temporal から生まれた新しいタイムスタンプの構文が RFC 9557 となって Approve された
- Temporal ではコードサイズや複雑性の観点からスコープを狭めることが計画されている
  - `Temporal.TimeZone` と `Temporal.Calendar` がなくなりそう (既存のタイムゾーン・カレンダーの指定はできる)
  - その他にも API が削減される予定であり、 最大で 1/3 ほどの API が削減されるかもしれない

長い間検討され、既存の Date オブジェクトの不満を解消すべく意欲的な機能が盛り込まれている Temporal の動向を今後も注視していきたいところです。

## 参考

- https://docs.google.com/presentation/d/1HxocWS0WfIZPanCAhsabSDOPx9LCjw6upMfMP9ogEqE/edit?usp=sharing
