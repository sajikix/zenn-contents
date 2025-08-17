---
title: "Set Objectの仕様と実装を調べる"
emoji: "🕵️‍♂️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["JavaScript", "frontend"]
publication_name: "cybozu_frontend"
published: false
---

## はじめに

この記事は JavaScript の `Set` オブジェクトに関して、利用していて疑問に思った以下の２点について調べた記事です。

- `Set` の 一意性の保証について
- `Set` に対する性能要件とその実装

`Set` オブジェクトに対する理解を深めるだけでなく、JavaScript の仕様やランタイムの実装について疑問に思った時に調べるヒントとなれば幸いです。

## JavaScript における Set とは

みなさんは JavaScript の `Set` オブジェクト、利用していますか？

「 `Set` オブジェクトってなんだっけ？」という方向けにまずは簡単におさらいしておきます。Set オブジェクトは ES2015 から提供されているデータのコレクションを扱うビルドインオブジェクトで、コレクション内の値が重複しないという特徴があります。「値が重複しない配列」のように扱えますが、順序は挿入順となっておりインデックスアクセスはできません。

このように `Set` というと「値が重複しない(値の一意性が保証されている)」ことが大きな特徴として挙げられ、この特徴を利用した実装(重複削除など)も多いでしょう。

では「値が重複する」=「同値である」とはどんな状態でしょうか？ 最初に思い浮かぶのは「厳密等価演算子( `===` )で比較して `true` が帰ってくれば同値」というものです。多くのユーザーは `Set` でも `===` で値の同一性が検証されていると期待して使っていそうです。(自分はそうでした。)

でも本当にそうでしょうか？ JavaScript には 等価演算子( `==` )もありますし、もう少し詳しい方は `Object.is()` による比較が思いつくかもしれません。

:::message
JavaScript には等価性を判定するアルゴリズムとして以下の４種類ありそれぞれ挙動が異なります。

- 抽象的な等価 (`==`)
- 厳密な等価 (`===`)
- SameValue (`Object.is()`)
- SameValueZero

詳しくは MDN の[等価性の比較と同一性](https://developer.mozilla.org/ja/docs/Web/JavaScript/Equality_comparisons_and_sameness)などのページがわかりやすいです。
:::

<!-- ### JavaScript における同値 -->

## MDN と仕様書を見てみよう

### MDN の記述について

まずは皆さんがよく見る MDN で [`Set` についての記述](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Set) を見てみましょう。

MDN の該当ページには「値の等価性」という項目があり以下のように記述されています。

> 値の等値性は、Same-value-zero アルゴリズムに基づいています。

さらに、Same-value-zero アルゴリズムについての補足として以下のように書かれています。

> つまり、 NaN は NaN と同じとみなされ (例え NaN !== NaN であっても)、それ以外の値は === 演算子の挙動に従って等しいとみなされます。

よって「`Set` はどうやって一意性を保証しているのか?」という問いに対する答えは「 `Same-value-zero` アルゴリズムで比較されており、`NaN` の扱い以外は `===` と同じ」だと言えそうです。めでたしめでたし...

...

...と今回のような例であれば MDN を見れば解決しましたが、場合によっては記述が古かったり日本語版では翻訳が追従していなかったりする場合があります。となると万全を期すのであればやはり仕様書を確認しに行きたいところです。

:::message
実際執筆者が最初に MDN の `Set` の記述を確認したタイミングでは日本語版が原文(英語版)に追従しておらず、等価性や性能についての記述がここまで明確ではありませんでした。
現在(2023/04 時点)は[執筆者が送った翻訳追従の PR](https://github.com/mdn/translated-content/pull/11875) をマージして頂き、英語版と同じ内容になっています。

MDN の記述が少し古く感じたり、内容が少ないように感じたらぜひ英語版も参照してみてください。

また MDN のコンテンツは現在、英語版・各言語版ともに github で管理運用されています。英語版の記述に間違いや古い記述を見つけたり、日本語版と英語版に乖離を見つけたりした場合は積極的に貢献していけると良いですね。

参考リンク

- [MDN のコンテンツ(英語版)を管理するリポジトリ](https://github.com/mdn/content)
- [MDN の各言誤訳を管理するリポジトリ](https://github.com/mdn/translated-content/)

:::

### ECMAScript の記述を読んでみる

ご存知の通り JavaScript の仕様は [ECMAScript](https://tc39.es/ecma262/) という名前で公開されており、 2015 年以降は毎年更新されています。

仕様書のサイトから直接検索してもいいのですが、非常に長いドキュメントなので目的の仕様に辿り着くまでが大変です。 MDN 経由の場合、大抵は「仕様書」というセクションがあり、ここに書かれているリンクからであれば直接該当箇所に飛ぶことができます。

今回の `Set` オブジェクトに関しては https://tc39.es/ecma262/multipage/keyed-collections.html#sec-set-objects から記述が始まります。

ECMAScript によると `Set` は以下のように定義されています。

> Set objects are collections of ECMAScript language values. A distinct value may only occur once as an element of a Set's collection. Distinct values are discriminated using the SameValueZero comparison algorithm.

ここでも 「 `SameValueZero` アルゴリズムで区別している」と明確に書かれています。この時点で当初の疑問はほぼ完全に解決したのですが、せっかくなので値の同一性をチェックしていそうな `Set.has()` の記述も見てみましょう。

`Set.has()` については[この部分](https://tc39.es/ecma262/multipage/keyed-collections.html#sec-set.prototype.has)から始まります。

```
1. Let S be the this value.
2. Perform ? RequireInternalSlot(S, [[SetData]]).
3. For each element e of S.[[SetData]], do
  a. If e is not empty and SameValueZero(e, value) is true, return true.
4. Return false.
```

`1.` で「S を this とする」と言っているので S が `Set` オブジェクト自体を指していると読めば良さそうです。ポイントは `3.` の操作で、 `Set` の全部の要素に対して `SameValueZero()` を利用して `has()` でもらった引数を比較しています。ここでも `SameValueZero` アルゴリズムが使われていることが確認できました。

この `SameValueZero` アルゴリズムの特徴として `+0` と `-0` の区別をつけないというものがあります。この部分の挙動も `Set` は考慮しているようです。 [`Set.add()`](https://tc39.es/ecma262/multipage/keyed-collections.html#sec-set.prototype.add) の記述を見てみましょう。

```
1. Let S be the this value.
2. Perform ? RequireInternalSlot(S, [[SetData]]).
3. For each element e of S.[[SetData]], do
  a. If e is not empty and SameValueZero(e, value) is true, then
    i. Return S.
4. If value is -0𝔽, set value to +0𝔽.
5. Append value to S.[[SetData]].
6. Return S.
```

基本的には `Set.has()` の記述と近い形ですが、`5.` で要素を追加する前に `4.` で `-0` を `+0` にキャストしています。この仕様のため `Set` にはそもそも `-0` が入らないようになっていることがわかります。

## `Set` の性能に関する要件

ここまで `Set` の大きな特徴である「値の一意性」を保証する仕組みについてみてきました。もう 1 つの `Set` の特徴として「配列よりも探索の性能面で優れる」というものがあります。

この性能についても ECMAScript には記述があります。具体的には先ほど引用した [`Set` オブジェクトの説明部分](https://tc39.es/ecma262/multipage/keyed-collections.html#sec-set-objects)に以下のように書かれています。

> Set objects must be implemented using either hash tables or other mechanisms that, on average, provide access times that are sublinear on the number of elements in the collection.

特に後半が重要で、「平均アクセス時間がコレクション内の要素数に対して線形以下」であることを求めています。もう少し噛み砕くと仕様書側で各 JS ラインタイムに対し、平均してアクセス性能が `O(n)` 以下であるような実装を要求していることになります。

`Set` はしばしば「配列に比べて探索の性能面で優れている」という紹介がされますが、この特徴はちゃんと仕様書レベルで保証(要求)されているものだったのですね。

## 各 JS エンジンにおける `Set` のデータ保持

こうなると気になってくるのが「各 JS エンジンの実装はどうなっているのか?」ということです。 仕様書では性能要件を提示していますが実装方法は各 JS ランタイム側に任されています。今回は主要ブラウザの JS ランタイムである `V8`, `SpiderMonkey`, `JavaScriptCore` の実装を見てみましょう。(ここからは自分が初めて読むものがほとんどです。間違っているところなどありましたらコメントいただけると嬉しいです。)

### V8 のコードリーディング

V8 のコードは [Google Git のページ](https://chromium.googlesource.com/v8/v8/+/refs/heads/main) から閲覧可能ですが、検索やコードジャンプなどが使えないので clone してきた方が読みやすそうです。

以下を参考に clone します。(コードを読むだけなら [github の公式ミラーリポジトリ](https://github.com/v8/v8)から clone だけしても良さそうですが、注意書きにある通りビルドなどをしたい場合は手順に従ってください。)

https://v8.dev/docs/source-code

#### V8 の `Set` 実装を読む

V8 のコードを読むにあたっては少し前の記事になりますが、[brn さん](https://twitter.com/brn227) の [V8 javascript engine についての細かい話](https://abcdef.gets.b6n.ch/entry/2017/12/25/120000) などがとても参考になります。(大まかな構造はそこまで変わっていないはず。)

まずは js の api 層との繋ぎきこみを実装している `src/api/api.cc` から追っていきます。[8217 行目](https://github.com/v8/v8/blob/fbb08351c4c7d845f6658c125ef7cf5c7c6ae43f/src/api/api.cc#L8217)以降から `Set` の API 実装部分です。具体的に `new()` のつなぎこみ部分を見てみましょう。

```cpp
Local<v8::Set> v8::Set::New(Isolate* v8_isolate) {
  i::Isolate* i_isolate = reinterpret_cast<i::Isolate*>(v8_isolate);
  API_RCS_SCOPE(i_isolate, Set, New);
  ENTER_V8_NO_SCRIPT_NO_EXCEPTION(i_isolate);
  i::Handle<i::JSSet> obj = i_isolate->factory()->NewJSSet();
  return Utils::ToLocal(obj);
}
```

`i_isolate->factory()->NewJSSet();` の部分で factory を読んでいます。この factory の実装は `src/heap/factory.cc` の [3111 ~ 3116 行目](https://github.com/v8/v8/blob/fbb08351c4c7d845f6658c125ef7cf5c7c6ae43f/src/heap/factory.cc#L3111-L3116)にあります。

```cpp
Handle<JSSet> Factory::NewJSSet() {
  Handle<Map> map(isolate()->native_context()->js_set_map(), isolate());
  Handle<JSSet> js_set = Handle<JSSet>::cast(NewJSObjectFromMap(map));
  JSSet::Initialize(js_set, isolate());
  return js_set;
}
```

このコードで言うところの `JSSet` の定義元を探していきます。`JSSet` の定義が含まれるヘッダーファイルは `src/objects/js-collection.h` で、以下のように定義されています。

```cpp
class JSSet : public TorqueGeneratedJSSet<JSSet, JSCollection> {
 public:
  static void Initialize(Handle<JSSet> set, Isolate* isolate);
  static void Clear(Isolate* isolate, Handle<JSSet> set);
  void Rehash(Isolate* isolate);

  // Dispatched behavior.
  DECL_PRINTER(JSSet)
  DECL_VERIFIER(JSSet)

  TQ_OBJECT_CONSTRUCTORS(JSSet)
};
```

今回はどんなデータ構造で管理しているかを知りたいので　`src/heap/factory.cc`　で呼び出されていた `Initialize` メソッドを見てみましょう。実装自体は `src/objects/objects.cc` の [6629 ~ 6632 行目](https://github.com/v8/v8/blob/fbb08351c4c7d845f6658c125ef7cf5c7c6ae43f/src/objects/objects.cc#L6629-L6632) 部分です。

```cpp
void JSSet::Initialize(Handle<JSSet> set, Isolate* isolate) {
  Handle<OrderedHashSet> table = isolate->factory()->NewOrderedHashSet();
  set->set_table(*table);
}
```

`factory` の `NewOrderedHashSet` を呼び出して `table` を生成しています。この `table` が保持しているデータの本体みたいですね。では先ほどと同じように `src/heap/factory.cc` の `NewOrderedHashSet` 定義元を見てみましょう。具体的には [586 ~ 590 行目](https://github.com/v8/v8/blob/fbb08351c4c7d845f6658c125ef7cf5c7c6ae43f/src/heap/factory.cc#L586-L590)になります。

```cpp
Handle<OrderedHashSet> Factory::NewOrderedHashSet() {
  return OrderedHashSet::Allocate(isolate(), OrderedHashSet::kInitialCapacity,
                                  AllocationType::kYoung)
      .ToHandleChecked();
}
```

ここでは `OrderedHashSet::Allocate` を実行しています。では `OrderedHashSet` の定義部分を見にいきましょう。 `OrderedHashSet` クラスは `src/objects/ordered-hash-table.h` の[271 行目から](https://github.com/v8/v8/blob/fbb08351c4c7d845f6658c125ef7cf5c7c6ae43f/src/objects/ordered-hash-table.h#L271)定義されています。

```cpp
class V8_EXPORT_PRIVATE OrderedHashSet
    : public OrderedHashTable<OrderedHashSet, 1> {
      // ...
    }

```

このように `OrderedHashSet` は `OrderedHashTable` を継承しています。`OrderedHashTable`は同じファイルの[67 行目から](https://github.com/v8/v8/blob/fbb08351c4c7d845f6658c125ef7cf5c7c6ae43f/src/objects/ordered-hash-table.h#L67)定義されています。

```cpp
class OrderedHashTable : public FixedArray {
  // ...
}
```

この `OrderedHashTable` の冒頭には以下のようなコメントがあります。

> OrderedHashTable is a HashTable with Object keys that preserves
> insertion order. There are Map and Set interfaces (OrderedHashMap
> and OrderedHashTable, below). It is meant to be used by JSMap/JSSet.
>
> Only Object keys are supported, with Object::SameValueZero() used as the
> equality operator and Object::GetHash() for the hash function.
>
> Based on the "Deterministic Hash Table" as described by Jason Orendorff at
> https://wiki.mozilla.org/User:Jorend/Deterministic_hash_tables
> Originally attributed to Tyler Close.

まさに、求めていた情報が書かれていますね。[コメントで言及されている文書](https://wiki.mozilla.org/User:Jorend/Deterministic_hash_tables)では「任意順序のハッシュテーブルの性能を維持しつつ、エントリの追加順序を保存するアルゴリズム」について前提と実装方法、性能調査の結果が書かれています。

これで V8 では `Set` をある種の**ハッシュテーブルで実装している**ことがわかりました。

### SpiderMonkey のコードリーディング

SpiderMonkey のコードは以下のページで読むことができます。こちらはコードの検索やジャンプにも対応しているのでこのページ上で読んでいきましょう。

https://searchfox.org/mozilla-central/source/js/src

#### SpiderMonkey の Set 実装を読む

V8 と同様、 JS の API 層との繋ぎ込み部分から探していきましょう。Set の繋ぎ込みは [source/js/public/MapAndSet.h の 52 行目](https://searchfox.org/mozilla-central/source/js/public/MapAndSet.h#52) から書かれています。今回は `add` メソッドの定義から追ってみましょう。

```cpp
extern JS_PUBLIC_API bool SetAdd(JSContext* cx, HandleObject obj,
                                 HandleValue key);
```

`SetAdd` の定義元は [source/js/src/builtin/MapObject.cpp の 1915 行目](https://searchfox.org/mozilla-central/source/js/src/builtin/MapObject.cpp#1915) になります。

```cpp
JS_PUBLIC_API bool JS::SetAdd(JSContext* cx, HandleObject obj,
                              HandleValue key) {
  CHECK_THREAD(cx);
  cx->check(obj, key);

  // Unwrap the object, and enter its compartment. If object isn't wrapped,
  // this is essentially a noop.
  RootedObject unwrappedObj(cx);
  unwrappedObj = UncheckedUnwrap(obj);
  {
    JSAutoRealm ar(cx, unwrappedObj);

    // If we passed in a wrapper, wrap key before adding to the set
    RootedValue wrappedKey(cx, key);
    if (obj != unwrappedObj) {
      if (!JS_WrapValue(cx, &wrappedKey)) {
        return false;
      }
    }
    return SetObject::add(cx, unwrappedObj, wrappedKey);
  }
}
```

ここでは最終的に `SetObject::add()` を呼んでいるのでこの実装元を見ましょう。実装しているのは[source/js/src/builtin/MapObject.cpp の 1356 行目](https://searchfox.org/mozilla-central/source/js/src/builtin/MapObject.cpp#1356)です。

```cpp
bool SetObject::add(JSContext* cx, HandleObject obj, HandleValue k) {
  ValueSet* set = obj->as<SetObject>().getData();
  if (!set) {
    return false;
  }

  Rooted<HashableValue> key(cx);
  if (!key.setValue(cx, k)) {
    return false;
  }

  if (!PostWriteBarrier(&obj->as<SetObject>(), key.get()) ||
      !set->put(key.get())) {
    ReportOutOfMemory(cx);
    return false;
  }
  return true;
}
```

ここをみると、内部的には `ValueSet` という型でデータを持っていそうです。`ValueSet`は[source/js/src/builtin/MapObject.h の 106 行目](https://searchfox.org/mozilla-central/source/js/src/builtin/MapObject.h#106) で以下のように宣言されています。

```cpp
using ValueSet = OrderedHashSet<PreBarriered<HashableValue>,
                                HashableValueHasher, CellAllocPolicy>;
```

さらに `OrderedHashSet` の[定義元](https://searchfox.org/mozilla-central/source/js/src/ds/OrderedHashTable.h#973)を見ると、以下のようになっています。

```cpp
class OrderedHashSet {
 private:
  struct SetOps;
  using Impl = detail::OrderedHashTable<T, SetOps, AllocPolicy>;

  //...

  Impl impl;

 public:
 // ....
};
```

内部的には `OrderedHashTable` でデータを保持しています。この `OrderedHashTable` の定義部分上部には以下のようなコメントが書かれいます。

> OrderedHashTable is the underlying data structure used to implement both OrderedHashMap and OrderedHashSet. Programs should use one of those two templates rather than OrderedHashTable.

`OrderedHashTable`は `Set` と Map の内部データ構造の継承元になっているようです。

さらに [/source/js/src/ds/OrderedHashTable.h](https://searchfox.org/mozilla-central/source/js/src/ds/OrderedHashTable.h#63)全体の先頭には以下のようなコメントがあります。

> Define two collection templates, js::OrderedHashMap and js::OrderedHashSet.
> They are like js::HashMap and js::HashSet except that:
>
> - Iterating over an Ordered hash table visits the entries in the order in which they were inserted. This means that unlike a HashMap, the behavior of an OrderedHashMap is deterministic (as long as the HashPolicy methods are effect-free and consistent); the hashing is a pure performance optimization.
>   ...

ここでもパフォーマンスのために基本的な実装をハッシュテーブルとしつつ、順序は保証するように実装していることが明記されています。

これで SpiderMonkey でも `Set` を**ハッシュテーブルで実装している**ことがわかりました。

### JavaScriptCore のコードリーディング

JavaScriptCore の実装は以下の github で公開されています。

https://github.com/WebKit/WebKit/tree/main/Source/JavaScriptCore

適宜 Clone して読むと読みやすいです。

#### JavaScript Core の `Set` の実装を追う

まずは　 JS の API 層との繋ぎ込み部分から探していきましょう。

`Set` は JS のグローバルオブジェクトの 1 つですが、このグローバルオブジェクトは [JavaScriptCore/API/JSAPIGlobalObject.h](https://github.com/WebKit/WebKit/blob/main/Source/JavaScriptCore/API/JSAPIGlobalObject.h) ヘッダファイルで定義されています。

```cpp
class JSAPIGlobalObject final : public JSGlobalObject {}
```

この Class 定義を見ると `JSGlobalObject` を継承しています。`JSGlobalObject` の実装を見に行くと、[定義されたマクロ内](https://github.com/WebKit/WebKit/blob/0646de88e5b9e9e657741c81b75e51e353ff35ae/Source/JavaScriptCore/runtime/JSGlobalObject.h#L146)に `Set` の記述があります。

```cpp
macro(Set, set, set, JSSet, Set, object, typeExposedByDefault)
```

このマクロの呼び出し元(複数)を見ると引数に渡される`macro`は以下のようになっています。

```cpp
#define DECLARE_SIMPLE_BUILTIN_TYPE(capitalName, lowerName, properName, instanceType, jsName, prototypeBase, featureFlag) \
    class JS ## capitalName; \
    class capitalName ## Prototype; \
    class capitalName ## Constructor;
```

したがって `instanceType`　にあたる `JSSet` 定義元に `Set` オブジェクトの実装がありそうです。

では JSSet の[ヘッダファイルでの定義部分](https://github.com/WebKit/WebKit/blob/cc0701ad6d8c29f5b0469b2ca944818f5856d538/Source/JavaScriptCore/runtime/JSSet.h#L33-L66)を見てみましょう。

```cpp
class JSSet final : public HashMapImpl<HashMapBucket<HashMapBucketDataKey>> {
    using Base = HashMapImpl<HashMapBucket<HashMapBucketDataKey>>;
  // ...
}
```

この定義から `JSSet` は `HashMapImpl` を継承していることがわかりました。`HashMapImpl` のヘッダファイルでの定義は[runtime/HashMapImpl.h の 244 行目](https://github.com/WebKit/WebKit/blob/a40563bfe366254cad9399917bee1b691f3ecafe/Source/JavaScriptCore/runtime/HashMapImpl.h#L244)にあります。

```cpp
class HashMapImpl : public JSNonFinalObject {
    using Base = JSNonFinalObject;
    using HashMapBufferType = HashMapBuffer<HashMapBucketType>;

public:
    using BucketType = HashMapBucketType;
    // ...
}
```

もう少し詳しく `add()` メソッドの実装場所なども参照してみましょう。`add()` メソッドは[runtime/HashMapImplInlines.h の 248 行目](https://github.com/WebKit/WebKit/blob/a40563bfe366254cad9399917bee1b691f3ecafe/Source/JavaScriptCore/runtime/HashMapImplInlines.h#L248)にあります。

```cpp
ALWAYS_INLINE void HashMapImpl<HashMapBucketType>::add(JSGlobalObject* globalObject, JSValue key, JSValue value)
{
    key = normalizeMapKey(key);
    addNormalizedInternal(globalObject, key, value, [&] (HashMapBucketType* bucket) {
        return !isDeleted(bucket) && areKeysEqual(globalObject, key, bucket->key());
    });
}
```

この [`addNormalizedInternal`部分の実装](https://github.com/WebKit/WebKit/blob/a40563bfe366254cad9399917bee1b691f3ecafe/Source/JavaScriptCore/runtime/HashMapImplInlines.h#L376)を見てみると、hash の生成や key/value のセット、さらに `HashMapBucket` を利用して順序を記録する実装が入っています。

このことから JavaScriptCore でも `Set` を**順序保証をしたハッシュテーブル**で実装していることがわかりました。

## まとめ

今回の調査の結果をまとめると以下になります。

1. `Set` は `SameValueZero` アルゴリズムで一意性を保証しており、 `NaN` の扱い以外は `===` と同じ挙動である。
2. `Set` は 仕様でアクセス性能が O(n) 以下であることを要求しており、主要ブラウザ (Chrome, Firefox, Safari) の JS エンジンは順序付き HashTable で実装している。

JavaScript の機能や挙動に疑問がある際は、この記事のような形で仕様書や実装を見に行くとより理解が深まりそうです。

## 参考文献

この記事を書くにあたって参考とさせて頂いたページ・記事になります。

- MDN
  - [等価性の比較と同一性](https://developer.mozilla.org/ja/docs/Web/JavaScript/Equality_comparisons_and_sameness)
  - [Set](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Set)
- ECMAScript
  - [24.2 Set Objects](https://tc39.es/ecma262/multipage/keyed-collections.html#sec-set-objects)
- V8
  - [Checking out the V8 source code](https://v8.dev/docs/source-code)
  - [chromium/v8/v8/refs/heads/main (Google Git)](https://chromium.googlesource.com/v8/v8/+/refs/heads/main)
  - [V8 JavaScript Engine (github.com)](https://github.com/v8/v8)
- SpiderMonkey
  - [mozilla-central](https://searchfox.org/mozilla-central/source/js/src)
- JavaScriptCore
  - [Webkit JavaScriptCore](https://github.com/WebKit/WebKit/tree/main/Source/JavaScriptCore)
- [V8 javascript engine についての細かい話](https://abcdef.gets.b6n.ch/entry/2017/12/25/120000)
- [User:Jorend/Deterministic hash tables](https://wiki.mozilla.org/User:Jorend/Deterministic_hash_tables)
