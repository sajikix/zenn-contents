---
title: "Intl.DateTimeFormatとTemporalの関連(#9)"
emoji: ""
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: false
publication_name: "cybozu_frontend"
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 9 日目の記事です。

今回は日時の操作に関する ECMAScript Proposal、Temporal と Intl における日時系フォーマット機能の関連について解説します。

## Temporal とは

Temporal は ECMAScript で提案されている新しい日時操作のための標準オブジェクトで、現在 Stage 3 の Proposal です。Temporal は既存の Date オブジェクトの以下のような不満点を解消、日時操作をより簡単に行えるようにすることを目指しています。

- ユーザーの現地時間と UTC 以外のタイムゾーンがサポートされていない
- 日時文字列のパース動作がわかりづらく不安定
- 日付オブジェクトが何を指しているのかわかりづらい
- サマータイム動作が不安定
- 日時計算の API が扱いにくい
- 非グレゴリオ暦がサポートされていない

具体的に Temporal は以下のようなコンセプトを持って設計されています。

- Wall-Clock Time と Exact Time の分離
- タイムゾーン・カレンダーのサポート
- 便利な計算メソッドの追加

### Wall-Clock Time と Exact Time の分離

### タイムゾーン・カレンダーのサポート

### 便利な計算メソッドの追加

## Intl.DateTimeFormat との関連

Temporal は Intl.DateTimeFormat 同様日時というドメインを扱うため、両者の仕様は密接に関連しています。より具体的に Temporal の持つ日時のフォーマットや指定できるオプションなどについては Intl に準拠した仕様策定が進められています。

- Temporal の toLocaleString()メソッド
  - Intl.DateTimeFormat と同じ引数を受け取る
- Temporal.Calendar で対応するカレンダー
  - Intl.DateTimeFormat でサポートするものと一緒
  - カレンダーの解決などは Intl 側の仕様を参照している
- Temporal.Duration
  - Intl.DurationFormat(詳しくは[12 日目]()に解説)と同じ Option になる予定

また逆に一部の Intl の仕様に関しても Temporal が入ることで変更されることが予定されています。

- Intl.DateTimeFormat が Date オブジェクト以外にも Temporal に対応する
- 一部の ECMA402 にあった仕様が ECMA262 側に移行される

## まとめ

## 参考文献

- Temporal Proposal
  - spec : https://tc39.es/proposal-temporal
  - github : https://github.com/tc39/proposal-temporal
