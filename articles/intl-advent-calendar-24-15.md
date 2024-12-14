---
title: "NumberFormatの新しいオプションとCurrency Display Choices Proposal(#15)"
emoji: "💲"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: false
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 15 日目の記事です。

今回は Intl.NumberFormat の比較的新しいオプションと、Currency Display Choices Proposal について解説します。

## NumberFormat の新しいオプション

Intl.NumberFormat は最初の ECMA-402 策定時から度々機能がアップデートされています。その中でも ES2023 に正式に仕様として標準化された「Intl.NumberFormat v3」という提案は新しくメソッドや様々なオプションを追加する大きなアップデートでした。この「Intl.NumberFormat v3」もとい、ES2023 から追加された Intl.NumberFormat の仕様群はその後各ランタイムで実装が進み、2024 年 12 月現在全てのブラウザで利用可能になっています。

具体的に、ES2023 での大幅アップデートで以下のような機能・メソッド・オプションが追加されました。

- `formatRange()` / `formatRangeToParts()` メソッドの追加
- `format` 系のメソッドで BigInt や数値文字列を受け付けるように
- 表示桁数や値の丸めに関する細かいオプションの追加
- 桁区切りのグルーピングに関するオプションや、数値の正負の表現に関するオプションの追加

今回はその中でも「表示桁数や値の丸めに関する細かいオプション」と「桁区切りのグルーピングに関するオプションや数値の正負の表現に関するオプション」について詳しく解説します。(`formatRange()` / `formatRangeToParts()` メソッドや `format` 系のメソッドで BigInt を扱えることついては、まとまりの関係上すでに[13 日目](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-13)で解説済みです。)

### 表示桁数や値の丸めに関するオプション

NumberFormat v3 で追加された表示する桁数や値の丸めに関するオプションは以下の 2 つに分類できます。

- 整数部分や小数部分の桁数に関するオプション
- 数値の丸め挙動に関するオプション

#### 整数部分や小数部分の桁数に関するオプション

桁数指定のオプションに関しては２つの「モード」があり、それぞれのオプションがどちらかのモードに属しているというように考えると理解しやすいです。

- モード 1 : 整数部分と小数部分の桁数をそれぞれ指定する
  - `minimumIntegerDigits` : 整数部分の最小桁数
  - `minimumFractionDigits` : 小数部分の最小桁数
  - `maximumFractionDigits` : 小数部分の最大桁数
- モード 2 : 整数部分と小数部分合わせた桁数を指定する
  - `minimumSignificantDigits` : 全体の最小桁数
  - `maximumSignificantDigits` : 全体の最大桁数

モードの判定では、モード 2 に属するオプションが１つでも指定されている場合はモード 2 として扱われます。この場合モード１に属するオプションは無視されます。

##### モード 1 の場合

モード 1 に属するオプションでは、特に少数部分の桁数を固定したいような場合に便利です。例えば以下のように `minimumFractionDigits` と `maximumFractionDigits` を指定することで、小数部分を必ず 2 桁にできます。

```ts
const formatter = new Intl.NumberFormat("en", {
  maximumFractionDigits: 2, // 小数部分の最大桁を2桁
  minimumFractionDigits: 2, // 小数部分の最小桁を2桁
});
formatter.format(123.456); // → 123.46
formatter.format(123.4); // → 123.40
```

ただし `style` が `"currency"` の場合は通貨の規定によって小数部分の桁数が指定通りにならないことがあるので場合は注意が必要です。この場合、基本的に [ISO 4217](https://www.iso.org/iso-4217-currency-codes.html)の通貨コード一覧における minor unit(副単位 : セントとか銭とか) に従って小数部分の桁数が決まり、何も情報がない場合は 2 桁になります。

##### モード 2 の場合

モード 2 に属するオプションでは、全体の桁数を調整できるので、表示スペースが限られている場合などに便利です。例えば以下のように `minimumSignificantDigits` と `maximumSignificantDigits` を指定することで、全体の桁数を必ず 5 桁にできます。

```ts
const formatter = new Intl.NumberFormat("en", {
  minimumSignificantDigits: 5, // 全体の最大桁を5桁
  maximumSignificantDigits: 5, // 全体の最小桁を5桁
  useGrouping: false, // ","による桁区切りを無効化
});
formatter.format(123.4567); // → 123.46
formatter.format(1234567); // → 12346
```

#### 数値の丸め挙動に関するオプション

次に数値の丸めに関するオプションを見ていきましょう。数値の丸めに関するオプションは以下の 3 つがあります。

- `roundingMode` : 丸め方法を指定する
- `roundingIncrement` : 丸めの単位を指定する
- `roundingPriority` : 丸めの優先順位を指定する

##### `roundingMode`

`roundingMode` は数値の丸め方に関するオプションで、全部で９種類存在します。

`"ceil"`/ `"floor"`/ `"expand"`/ `"trunc"` はともに切り捨て or 切り上げをする方法で以下のような違いがあります。

- `"ceil"` : 常に +∞ 側に丸める
- `"floor"` : 常に -∞ 側に丸める
- `"expand"` : 常に 0 から離れる方向に丸める
- `"trunc"` : 常に 0 に近い方向に丸める

`"halfCeil"`/ `"halfFloor"`/ `"halfExpand"`/ `"halfTrunc"`/ `"halfEven"` はより近い方の丸め値に丸める方法です。ちょうど真ん中の値(整数で丸めようとした時の `.5` の状態)の場合にどちらで丸めるかの違いで以下のように分かれています。

- `"halfCeil"` : ちょうど真ん中の値は +∞ 側に丸める
- `"halfFloor"` : ちょうど真ん中の値は -∞ 側に丸める
- `"halfExpand"` : ちょうど真ん中の値は 0 から離れる方向に丸める
- `"halfTrunc"` : ちょうど真ん中の値は 0 に向かう方向で丸める
- `"halfEven"` : ちょうど真ん中の値は偶数になる方向で丸める

ちなみにいわゆる「四捨五入」は　`"halfExpand"` を指定することで実現でき、`roundingMode` のデフォルト値も `"halfExpand"` です。一方、ISO/JIS の規則や浮動小数点の仕様などでも採用されている、コンピュータ分野でよく使われる丸め方法は `"halfEven"` なので一応注意しておきましょう。(基本的に `"halfEven"` の方が「丸め誤差が偏らない」というメリットがあるので多くの処理や仕様で採用されていると考えられますが、書式化ではより馴染みの深い？形を優先したのではないかと筆者は勝手に考えています。)

##### `roundingIncrement`

`roundingIncrement` はどの単位で丸めるかのオプションで、デフォルトは `1` です。使うことは少ないかもしれませんが、クォーターで丸めたい場合なども考慮されており、1, 2, 5, 10, 20, 25, 50, 100, 200, 250, 500, 1000, 2000, 2500, and 5000 のいずれかの値を指定できます。

```ts
const formatter = new Intl.NumberFormat("en", {
  roundingIncrement: 25, // クォーターで丸める
});
formatter.format(123456); // → 123.50
```

##### `roundingPriority`

#### roundingPriority と桁数オプションの関係

### 桁区切りのグルーピングに関するオプションや数値の正負の表現に関するオプション

それぞれ `signDisplay` オプションと `useGrouping` オプションが新たに追加されました。

#### `signDisplay` オプション

`singDisplay` オプションは数値の正負の表現を指定するオプションです。以下の５つの値のいずれかを指定でき、デフォルトは `"auto"` です。

- `"auto"` : 数値が負の場合のみ `-` が付く。(`-0` を含む)
- `"always"` : 正負に関わらず `+/-` が付く
- `"exceptZero"` : 0 以外の数値で `+/-` が付く
- `"negative"` : 数値が負の場合のみ `-` が付くが、`-0` は `0` と表示される
- `"never"` : 正負に関わらず `+/-` が付かない

#### `useGrouping` オプション

また「`1,000,000`」のようにカンマで桁を区切るグループ化の挙動を指定する `useGrouping` オプションも追加されました。`useGrouping` オプションは以下の 5 つの値(文字列 or 真偽値)のいずれかを指定できます。

- `"auto"` : ロケールに応じてグループ化する
- `"always"` : 常にグループ化する(ロケールに関わらず)
- `"min2"` : グループ内に少なくとも 2 桁の数字がある場合にグループ化する
- `true` : `"always"` と同じ
- `false` : グループ化しない(ロケールに関わらず)

## Currency Display Choices Proposal

## まとめと次回予告
