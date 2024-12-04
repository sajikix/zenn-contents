---
title: "ロケール識別子(BCP47)とUnicode拡張について(#2)"
emoji: "🏷️"
type: "tech"
topics: ["Intl", "i18n", "frontend"]
published: true
---

この記事は「[1 人 Intl Advent Calendar 2024](https://adventar.org/calendars/10555)」の 2 日目の記事です。

今回はロケール識別子(言語タグ)とその仕様 BCP47 について解説します。

## ロケール識別子について

[1 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-01)で解説した通り、Intl のコンストラクタプロパティは必ず第 1 引数にロケール識別子か Intl.Locale オブジェクト（またはそれらの配列）を受け取ります。この記事では、その中でも一般的に使われる「ロケール識別子」について解説します。（Intl.Locale オブジェクトについては、[3 日目](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-03)と [4 日目の記事](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-04)で詳しく解説します。）

2 日目にして Intl 本体の話ではないのですが、Intl のロケール指定やオプション指定の挙動を詳しく理解するためには、このロケール識別子の仕様についても理解しておく必要があります。

### ロケール識別子の仕様 : BCP47

ロケール識別子は「言語タグ」とも呼ばれる、ロケールを識別するための文字列です。具体的には `"ja-JP"` や `"en-US"` といったよく目にする文字列がこれに該当します。

このロケール識別子の仕様は、IETF という標準化団体が策定する 2 つの仕様、[RFC 5646](https://www.rfc-editor.org/rfc/rfc5646.html) と [RFC 4647](https://www.rfc-editor.org/rfc/rfc4647.html) で定義されています。この 2 つの仕様を合わせて [**BCP47**](https://www.rfc-editor.org/rfc/bcp/bcp47.txt)と呼びます。（BCP は Best Current Practice の略で、IETF が発行する RFC の種類の 1 つです）。このように IETF が標準化していることから、IETF 言語タグとも呼ばれます。

### BCP47 の基本 : サブタグ

ロケール識別子は「サブタグ」と呼ばれる要素を `-` でつないだ文字列で構成されます。サブタグには次の 6 種類が存在し、上から順にサブタグをつなげていくことで 1 つの言語タグが構成されます。

- 言語サブタグ
  - 言語を表す文字列
  - 例: `ja` や `en` など
- 文字体系サブタグ
  - 文字体系を表す文字列
  - 例: `Hant`（繁体字） / `Hans`（簡体字）など
- 地域（または国）サブタグ
  - 国や地域を表す文字列
  - 例: `CN` , `US` など
- 変化形サブタグ
  - 言語の方言や変化系などを表す文字列
  - 例: `emodeng`（"Early modern English" 方言）など
  - 複数指定できるが、すべて固有でなければならない
- BCP 47 拡張シーケンス
  - 単一の数字または文字（"x" 以外）と、ハイフンで区切られた 1 つ以上の 2 から 8 文字の文字または数字によるサブタグ
  - 例: `u-ca-japanese`
  - 複数指定できる
- 私的に使用する拡張シーケンス
  - 私的利用部分

実際にこれらを `"-"` でつなぐと、以下のような形になります。

```
[言語]-[文字体系]-[地域]-[変化形]-[BCP 47 拡張]-[私的に使用する拡張]
```

たとえば、1 ～ 3 つ目のサブタグを使って「台湾で利用される繁体字の中国語」を表したい場合、`zh-Hant-TW` のようになります。

ちなみに、言語、文字体系、地域、変化形サブタグで使える値に関しては、IANA[^1] の[Language Subtag Registry](https://www.iana.org/assignments/language-subtag-registry/language-subtag-registry)で管理されています。これらの値は定期的に更新されていますが、ブラウザに反映されるには時間差があることに注意しましょう。

### BCP 47 拡張シーケンスと u 拡張

この中でもあまり見慣れないであろう 5 番目のサブタグ、「BCP 47 拡張シーケンス」についてもう少し掘り下げましょう。

BCP 47 拡張シーケンスは `[1文字(x以外)]-[2~8文字]-[2~8文字(任意)]` という形で構成されるサブタグで、ロケール識別子に情報を追加し拡張することを目的に作られたサブタグです。ただし、ユーザーが自由に設定できるわけではなく、（x 拡張を除いて）[RFC5226](https://www.rfc-editor.org/rfc/rfc5226.html)で定義されている "IETF Review" ポリシーを使用して IANA によって割り当てられたものしか指定できません。現在 IANA に登録されているものは以下の 2 種類です。

- "u" (Unicode) 拡張
  - ロケールを識別するために必要な追加情報を含めることができる
  - `u-[key]-[value]` の形式
  - 例: `u-ca-japanese` → 日時書式で和暦を使用
- "t" (transformed) 拡張
  - 他のロケールから翻訳されたテキストなど、変換されたコンテンツを示す
  - `t-[value]`
  - 例: `t-it` → イタリア語からの翻訳

また、`x-` で始まる"x"拡張は事前にユーザーが自由に使えるよう予約されています。

- "x"拡張
  - ユーザーが自由に使える拡張シーケンス
  - 例: `ja-JP-x-saji`

このうち、Unicode 拡張部分の細かい仕様（実際にどのような key と value を持つのか）は Unicode 側の仕様に任されています。実際にどのような key と value が入るかは以下のドキュメントを参考にしてください。

https://www.unicode.org/reports/tr35/#Locale_Extension_Key_and_Type_Data

また、この中で "t" (transformed) 拡張と "x" 拡張は Intl ではすべて無視されますが、Unicode 拡張は `Intl` の各所で解釈され、挙動のカスタマイズに使用されます。たとえば、`Intl.DateTimeFormat()` を利用する際に `ja-JP-u-ca-japanese` というロケール識別子でロケールを指定すると、「日時書式に和暦を使用する」という意味になります。

```ts
const formatter = new Intl.DateTimeFormat("ja-JP-u-ca-japanese"); // u-ca-japanese がUnicode拡張
formatter.format(new Date(Date.UTC(2023, 7, 7)));
// → 'R5/8/7'（和暦）
```

今度の記事の説明で `u-[key]-[value]` のようなサブタグが出てきた時はこの記事の説明を思い出し、「Unicode 拡張のことだな」と読み進めてください。

## まとめ

今回はケール識別子(言語タグ)とその仕様 BCP47 について解説しました。また合わせて Intl で利用される Unicode 拡張についても補足しました。次回 [3 日目](https://zenn.dev/sajikix/articles/intl-advent-calendar-24-03)では Intl.Locale オブジェクトについてと Intl Locale Info proposal について解説します。

## 参考文献

- Unicode 側の BCP 47 U 拡張の定義
  - https://www.unicode.org/reports/tr35/#Locale_Extension_Key_and_Type_Data
- RFC のカテゴリについて
  - https://www.nic.ad.jp/ja/rfc-jp/RFC-Category.html

[^1]: Internet Assigned Numbers Authority の略。ドメイン名、IP アドレスなどのインターネット資源を管理している機能のこと。元々は団体名だったが、ICANN(The Internet Corporation for Assigned Names and Numbers) に統合されて ICANN の１機能となっている。
