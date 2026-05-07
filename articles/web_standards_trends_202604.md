---
title: "Web 標準動向 2026年4月版"
emoji: "🌱"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["cybozuwebstandards", "frontend"]
publication_name: "cybozu_frontend"
published: true
---

こんにちは！ サイボウズ株式会社 フロントエンドエンジニアの [Saji (@sajikix)](https://twitter.com/sajikix) です。

## はじめに

サイボウズは 2025 年 4 月より、W3C のメンバーに加入しました。

https://blog.cybozu.io/entry/joining-w3c

標準化プロセスに関わることができるようになるための最初の一歩として、フロントエンドエンジニアの一部のメンバーは積極的に Web 標準のキャッチアップを行っています。

そこで、毎月メンバーが興味を持った Web 標準に関する話題や、実際に標準化プロセスに関わることができた場合にはその報告などを 1 つの記事としてまとめ、紹介していきます。
また、ここでは W3C に限らず、TC39 や WHATWG などの標準化団体のトピックについても扱います。

:::message
今月の執筆者は以下の 4 名です。

- [Saji](https://x.com/sajikix)
  - ECMA402 (Intl) 周りのトピックを執筆
- [saku](https://x.com/sakupi01)
  - HTML, CSS に関連するトピックを執筆
- [くらっち](https://x.com/kuracchi04)
  - ECMA262 周りのトピックを執筆
- [mehm8128](https://x.com/mehm8128)
  - 主にアクセシビリティに関連するトピックを執筆

:::

## HTML

### Improved Japanese phonetic name support in Chrome autofill

https://developer.chrome.com/blog/japanese-phonetic-name-autofill?hl=en

Chromeにおいて氏名を入力するときに、振り仮名のオートフィルが改善されました。

主にautocomplete属性、フィールドのラベル、name属性に基づくヒューリスティックを使用して、振り仮名であることを判定するとのことです。

### Ready for Developer Testing: Overscroll Gestures

https://groups.google.com/a/chromium.org/g/blink-dev/c/YYe6ZhaMDrc

Overscroll 領域を用いる UI パターン（ハンバーガーメニューの表示、Pull to Refresh など）がデベロッパーテスト可能になりました。

これまでこれらのジェスチャーはブラウザや OS ごとに実装が異なり、Web開発者がカスタマイズすることが困難でした。この機能により、Web アプリケーションでも一貫したオーバースクロールジェスチャーを実装できるようになります。

### Upcoming WHATNOT meeting on 2026-04-02

https://github.com/whatwg/html/issues/12322

WHATNOTミーティングにおいて、Chris Wilsonが4月19日をもってGoogleを退職するため、WHATNOTのチェアをJohnny Stenbeckが引き継ぐことが発表されました。

Chris Wilsonは長年Web標準の発展に貢献してきた人物であり、今月開かれた BlinkOn 21でも「The past, present, and future of Web standards」と題したAMAセッションを行っていた人物です。

https://cwilso.com/2026/04/17/tick-2/

### Blink: Intent to Ship: Focusgroup

https://groups.google.com/a/chromium.org/g/blink-dev/c/53PP0LgddCQ/m/Be_6nT9VBgAJ

[9月版の記事](https://zenn.dev/cybozu_frontend/articles/web_standards_monthly_202509#intent-to-prototype%3A-focusgroup)で紹介した FocusGroupが、BlinkでIntent to Shipとなりました。v150でリリースが予定されています。

## CSS

### Blink: Intent to Prototype: CSS `text-decoration-inset`

https://groups.google.com/a/chromium.org/g/blink-dev/c/fU0_ERirUC8/m/W16pFuviAgAJ

`text-decoration`の線が、テキストの端からどのくらい内側/外側まで表示されるかを調整するプロパティです。

今までも背景色のグラデーションや疑似要素などを用いて無理やり調整することができましたが、これによって一貫性のある調整が可能になります。

Firefoxでは既に実装されていて、以下の記事が参考になります。

[Firefox 146 で \`text-decoration-inset\` プロパティが実装](https://blog.w0s.jp/entry/757)

### Ship: CSS URL request modifiers

https://groups.google.com/a/chromium.org/g/blink-dev/c/EKkOr6M6A8g

CSS の`url()`関数内で fetch のオプションを指定できるようになります。

```
background-image: url("image.png" cross-origin(anonymous));
```

これまでも CSS で読み込んだ画像は表示できましたが、CORS の設定ができなかったため、同じ画像を JavaScript から`CanvasRenderingContext2D`などで操作しようとするとセキュリティエラーになることがありました。この機能により、CSS 側で CORS 設定を宣言できるようになり、JavaScript からも同じ画像を扱えるようになります。

### Ship: `Popover=hint` behavior changes

https://groups.google.com/a/chromium.org/g/blink-dev/c/QkVJ1GYjLKk

ネストされた場合の`popover=hint`の仕様の問題で期待しない挙動が起こっており、仕様と Blink での実装が修正されました。

- Ship: Popover=hint behavior changes
  - https://groups.google.com/a/chromium.org/g/blink-dev/c/QkVJ1GYjLKk

従来の仕様では、hint popover が auto popover の中にネストされると、hint が auto に変換されてしまうなど、不自然な動作になっていました。これに対して、hint が「どの auto popover から分岐したか」を追跡するモデルが新しく定義されました。

また、hint 内で auto を出すことは禁止されます。`showPopover(auto)`を hint 内で呼び出すと例外がスローされ、HTML 属性での指定は無視されてコンソール警告が出るようになります。

ただし、カスタマイザブルな `<select>`要素が auto として振る舞うため、hint とのネスト時の競合解決については別途議論が続いています。

### Prototype: `overscroll-behavior: chain`

https://groups.google.com/a/chromium.org/g/blink-dev/c/hC2LUZ3XEBE

`overscroll-behavior`プロパティに新しいキーワード`chain`が追加されます。

既存の`overscroll-behavior`はスクロールの「伝搬」と境界での「ブラウザアニメーション」（バウンス効果など）を制御できますが、アニメーションのみをオフにするキーワードがありませんでした。

サイドメニューなどで親要素へのスクロール伝搬は許可しつつ、ローカルのバウンスアニメーションはオフにしたいユースケースに対応するものです。

### Implement and Ship: CSS container style queries

https://groups.google.com/a/mozilla.org/g/dev-platform/c/WpRW4SetBL8/m/HN1yyFWpAgAJ

CSS コンテナスタイルクエリ（`@container style(...)`）が Firefox でも実装・Ship されました。これにより、全主要ブラウザで利用可能となり、Baseline Newly Available となります。

コンテナスタイルクエリを使うと、コンテナ要素のカスタムプロパティ（CSS 変数）の値に基づいてスタイルを適用可能になります。

### Proposal: Declarative Web Haptics

https://github.com/w3c/csswg-drafts/issues/13728

CSS から宣言的に Haptic（触覚）フィードバックを指定する Edge チームからの提案です。

現在、モバイルデバイスでの触覚フィードバックは OS/ブラウザに依存しており、Web 開発者がコントロールすることができません。この提案により、ボタンクリックやトグル操作などのインタラクションに対して、ハックを用いず CSS で宣言的に触覚フィードバックの種類を指定できるようになるという提案が進んでいます。

### \[css-fonts-5\] `<meta text-scale>` limits

https://github.com/w3c/csswg-drafts/issues/13557

iOS などでユーザーがテキストサイズを拡大設定した場合、指定の仕方によっては極端に文字サイズが大きくなり、レイアウトが破綻する可能性があります。この問題に対処するため、スケールの上限を設定する仕組みを導入してはどうかという議論が行われました。

CSSWG の RESOLUTION では、`scale`キーワードを廃止し、上限を指定できるようにすることが検討されました。これにより、著者は「どこまでのスケーリングに対応したか」を明示できるようになります。

しかし、「上限以上に拡大したいユーザは一定数いるはずで、それを著者側で制限すべきではない」というアクセシビリティの観点からの反論が大きく、議論が続いています。

### Proposal: CSS Text Transitions & Animations

https://github.com/w3c/csswg-drafts/issues/13689

テキストの文字自体をアニメーションさせる提案です。これまで`<span>`などで文字を個別に区切って実装していたテキストアニメーションに対する、標準側のソリューションとなります。

この提案は BlinkOn 21 でも「Declarative Text Animations」として発表されており、実装に向けた具体的な議論が進んでいます。

### \[css-conditional\] Layout State Container Queries: `column-position` feature

https://github.com/w3c/csswg-drafts/issues/13729

flexbox、grid、multi-column レイアウト内で、要素が何番目のカラム/ローにあるかをコンテナクエリで取得できるようにする提案です。

現在の CSS では、グリッドアイテムが何列目にあるかによってスタイルを変えることができません。この機能が実装されれば、カラム位置に応じた条件付きスタイリングが可能になります。例えば、マルチカラムレイアウトの最初のカラムと最後のカラムで異なるマージンを適用するといったことが宣言的に記述できるようになります。

## JavaScript

### ECMA402

https://github.com/tc39/ecma402/blob/main/meetings/notes-2026-04-16.md

#### Updates from the Editors

ECMAScript 2026カットオフに向けた進捗の確認がされました。 Intl Era/Month Code および Temporal は今回のカットには含まれないことになります。

#### Updates from the MessageFormat Working Group

MessageFormat 2.0 (MF2) の仕様策定と W3C での Message Resources の策定について進捗報告がなされました。MessageFormat 2.0 に関してはICU4C での実装がレビュー待ちで停滞していることを鑑みて、Message Resources進展やその他のエコシステムでの実装など含めてある程度出揃ってからTC39の本会議に提出するのが良いのではという結論になっています。

#### Intl Unit Protocol: continuation on fallback logic for existing, non-protocol implementing, object parameters

https://github.com/tc39/proposal-intl-unit-protocol/issues/5

３月の議論から引き続き、Intl Unit Protocol Proposalにおいて「プロトコルを実装していない既存のオブジェクトに対するフォールバック」の挙動について議論がなされました。

Intl Unit Protocol Proposal は現在の Intl.NumberFormatを拡張し、「`unit` と `value` プロパティを持つオブジェクトをformat時に受け取れる」ようにするProposalです。これによりフォーマット時に単位を明示的に指定できるだけでなく、現在提案されている`Amount`のような値と単位をセットで持つデータを扱いやすくなります。

今回の(並びに前回も)議論の争点はオブジェクトが特定のプロパティ（`.value` と `.unit`）を持っているかをどのように判別するかです。方法としては大きく次の２点が考慮されています。

- プロトタイプチェーンを辿ってプロパティ（`.value` と `.unit`）の存在を確認する新しい抽象操作を導入する
- 既存の `Get` 操作で `undefined` チェックを行う

これに対して、値と単位をセットで持つ`Amount`の仕様において、「単位がない時」の`.unit`が `null` なのか `undefined` なのかが確定しないと議論を進めることが難しいという指摘があり、まずは `Amount` のProposalのissueでこの問題を考えることになりました。

#### Sequence Units follow-up

https://github.com/tc39/ecma402/issues/398

「NフィートXインチ」のような複合単位の書式化について３月の議論から引き続き議論がおこなわれました。

３月の議論では「フィートかつインチ（foot-and-inch）」のような複合単位において、与えられた数値がどちらの単位に基づいてるとするか(フィートorインチ)について話し合われていました。

提案されている案としては次の３つがあります。

- 最大単位をインデックスとする(CLDRが採用)
- 最小単位をインデックスとする
- 識別子文字列の最初の単位をインデックスとする

今回はその議論に加えて、新しく「シーケンス・フォーマッタ」のようなものを追加するのか、はたまたNumberFormatを拡張するのかなどが話し合われました。

結論としてはIntl.NumberFormat で複合単位をサポートする方向で、Amountプロポーザルの挙動とも調整しつつStage2を目指すことが合意されました。

#### String.toTitleCase, String.toLocaleTitleCase, with a focus on recent comments about titlecaseSegment

https://github.com/tc39/ecma402/issues/294

文字列のタイトルケース化（先頭文字の大文字化）を行う API の提案について話し合われました。

ICU4Xではすでに「セグメントの先頭のみをタイトルケース化する」APIがあるのでそれに従うのが良いという意見が出た一方で、CSS の `text-transform: capitalize` との整合性や、CSS 側での解決を優先すべきではないかという意見も出ました。

結論としては（CSS/HTML）と統合した形で、より包括的な議論が必要でありまずはCSS側との調整が必要であるということになりました。

### ECMA262

#### ECMAScript 2026のRelease Candidateが公開

https://github.com/tc39/ecma262/releases/tag/es2026-candidate-2026-03-31

2026年3月31日にES2026のRelease Candidateが公開されました。  
ES2025からの主な新機能は以下の通りです。

**Error.isError()**

引数がErrorオブジェクトかどうかを判定します。

```
Error.isError(new Error()) // true Error.isError(new TypeError()) // true Error.isError({ message: "err" }) // false
```

**Iterator.concat() / Iterator.from()**

`Iterator.concat()`は複数のイテラブルを連結した単一のイテレータを返し、

`Iterator.from()`は任意のイテラブル/イテレータを Iterator.prototype を継承するオブジェクトにラップします。

```
// 複数のイテラブルを連結 Iterator.concat([1, 2], [3, 4]) // Iterator: 1, 2, 3, 4 // イテラブル/イテレータをIterator.prototypeを継承するオブジェクトにラップ Iterator.from({ next() { return { value: 1, done: false } } })
```

**Math.sumPrecise()**

Numberのイテラブルが与えられると、この関数はイテラブル内の各値を合計し、その合計を返します。

```
// 従来の問題 0.1 + 0.2 + 0.3 // 0.6000000000000001 // Math.sumPrecise で解決 Math.sumPrecise([0.1, 0.2, 0.3]) // 0.6
```

**Array.fromAsync()**

非同期イテラブルやPromiseの配列から配列を作成します。

```
await Array.fromAsync(asyncGenerator()) await Array.fromAsync([Promise.resolve(1), Promise.resolve(2)]) // [1, 2]
```

**Uint8Array Base64/Hex**

バイナリ⇔文字列の相互変換をします。

```
// Base64 Uint8Array.fromBase64("SGVsbG8=") // Uint8Array [72, 101, 108, 108, 111] new Uint8Array([72, 101, 108, 108, 111]).toBase64() // "SGVsbG8=" // Hex Uint8Array.fromHex("48656c6c6f") // Uint8Array [72, 101, 108, 108, 111] new Uint8Array([72, 101, 108, 108, 111]).toHex() // "48656c6c6f"
```

**Map/WeakMap.prototype.getOrInsert**

キーが存在しない場合に値を挿入して返します。

```
const map = new Map() // キーがなければvalueを挿入して返す map.getOrInsert("key", "default") // "default" // キーがなければcallbackを実行して値を挿入 map.getOrInsertComputed("key2", (k) => fetchData(k))
```

**JSON.parse source text access**

`JSON.parse` の reviver 関数に第3引数として `context` オブジェクトが渡されるようになりました。プリミティブ値の場合、`context.source` でパース前の元のJSON文字列にアクセスできます。

```
const json = '{"value": 12345678901234567890}'; JSON.parse(json, (key, value, context) => { if (key === 'value') { console.log(value); // 12345678901234567000（精度が失われる） console.log(context.source); // "12345678901234567890"（元の文字列） return BigInt(context.source); // BigIntとして正確に復元可能 } return value; });
```

**JSON.rawJSON()**

生のJSONテキストを表すオブジェクトを作成します。

```
const raw = JSON.rawJSON('12345678901234567890'); // { rawJSON: "12345678901234567890" } を持つ凍結オブジェクト const obj = { bigNumber: raw }; JSON.stringify(obj); // '{"bigNumber":12345678901234567890}' // ※ 文字列ではなく数値としてそのまま出力される
```

**JSON.isRawJSON()**

オブジェクトが `JSON.rawJSON()` で作成されたものかどうかを判定します。

```
const raw = JSON.rawJSON('123'); JSON.isRawJSON(raw); // true JSON.isRawJSON({ rawJSON: '123' }); // false（手動で作っても偽物） JSON.isRawJSON(123); // false
```

### Prototype: Main thread Atomics.wait

https://groups.google.com/a/chromium.org/g/blink-dev/c/xQ5udr_sL7Q

ブラウザのメインスレッドをブロックしないために、メインスレッドでの `Atomics.wait`は禁止されています。

しかし、そのせいでスピンロック(ループでの停止)が広く使われてしまうため、メインスレッドでも `Atomics.wait` を有効にする提案がされています。

## Baseline

### 📃 April 2026 release notes

https://web-platform-dx.github.io/web-features-explorer/release-notes/april-2026

## Misc

### Fix a regression and turn on correct composition event ordering by default

https://github.com/WebKit/WebKit/pull/62264

IMEの候補が表示されている間は`e.isComposing`が`true`になり、onKeyDownイベントハンドラーで`e.isComposing === true`のチェックを挟むことで、IMEの候補を確定したときに誤ってonKeyDownイベントハンドラーを実行してしまうのを防ぐことができます。しかし、本来keydownイベントの後にcompositionendイベントが発火するべきであるのに、WebKitでは順番が逆になってしまっているという問題がありました。これにより、`e.isComposing === true`ではなくて`keyCode===229`という確認を挟む必要がありました。

[165004 – The event order of keydown/keyup events and composition events are wrong on macOS](https://bugs.webkit.org/show_bug.cgi?id=165004)

この問題が先日修正され、正しい順番で発火するようになったようです。

これらの問題については、以下のスライドで詳細が解説されています。

[IME vs Input Field Shortcuts: Enhancing Text Input Accessibility - Slidev](https://ef81sp.github.io/ime-vs-input-keyboard-shortcut/1)

### Feed Menus

https://www.ietf.org/archive/id/draft-nottingham-feed-menu-00.html

RSS feedを`<link>`要素から発見させる形式をやめて、well-known URIにJSONを配置する形式にすることで、既存の問題を解決する提案です。

詳細は以下の記事をご覧ください。

[Feed Menus による新しい RSS / Atom フィード自動検出の構想](https://blog.w0s.jp/entry/776)

### Final Soft Navigations origin trial starting in Chrome 147

https://developer.chrome.com/blog/final-soft-navigations-origin-trial

SPAでは、画面の遷移をDOM上で行う、Soft Navigationが行われます。

Soft Navigationは明示的な画面遷移が発生しないため、ブラウザはいつ画面が遷移したのかわかりませんでした。

これを、以下のような条件をSoft Navigationと定義して検知することで、Core Web Vitalsの検出などに活用する検証がすすめられており、最終段階に入りました。

条件

- ユーザが操作をする
- URLが変わる
- 画面が書き換わる

### Servo 0.1.0 が [Crates.io](http://Crates.io) で公開

https://servo.org/blog/2026/04/13/servo-0.1.0-release/

Rust 製ブラウザエンジンの Servo が、Rust 公式レジストリ「[Crates.io](http://Crates.io)」でリリースされました。これにより、Rust 製アプリケーションへの組み込みが容易になります。

2025 年 10 月の初回 GitHub 公開から 5 回のリリースを経て、埋め込み API の成熟度が向上したことを受けての [crates.io](http://crates.io) 公開となりました。なお、デモブラウザの「servoshell」は公開予定がないとのことです。

また、LTS（Long Term Support）版の提供も開始されます。6 ヶ月ごとに LTS 版がリリースされ、9 ヶ月間のサポート期間が設定されます。この期間中は API の仕様変更がなく、セキュリティパッチの提供が約束されるとのことです。

### Gemini in Chrome expanding to Asia-Pacific markets

https://blog.google/products-and-platforms/products/chrome/chrome-expands-apac/

Gemini in Chrome がアジア太平洋地域で順次ロールアウトを開始しました。日本も対象地域に含まれています。

### JetStream 3: A modern benchmark for high-performance, compute-intensive Web applications | Chromium Blog

https://webkit.org/blog/17899/introducing-the-jetstream-3-benchmark-suite/

https://blog.chromium.org/2026/03/jetstream-3-a-modern-benchmark.html

Apple、Google、Mozilla が共同で開発しているクロスブラウザベンチマークスイート「JetStream」がバージョン 3 にアップデートされました。

WebKit チームによると、Safari 26.0 から 26.4 への更新で約 10%の性能向上が実現されたとのことです。

### BlinkOn 21

https://www.chromium.org/events/blinkon-21/

https://www.youtube.com/playlist?list=PL9ioqAuyl6UIc-S8EuPjqkNvsMMmL1flZ

2026 年 4 月 20-21 日に開催された Chromium プロジェクトの会議です。今回は初めて Microsoft がホストとなり、ワシントン州レッドモンドで開催されました。
