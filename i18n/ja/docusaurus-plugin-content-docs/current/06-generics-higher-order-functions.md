# 6. ジェネリクスと高階関数

開発において、私たちは類似したデータ構造や操作に頻繁に遭遇します。そのような場合、適切な抽象化を通じてこの情報を再利用することで、保守性を確保するだけでなく、一部の詳細を無視することも可能になります。優れた抽象化は2つの原則に従うべきです：第一に、コード内で繰り返し現れる同じパターンや構造を表現すること；第二に、適切な意味論を持つことです。例えば、整数リストに対する合計操作を多くの場面で行う必要がある場合、繰り返しが発生します。合計操作には適切な意味論があるため、抽象化に適しています。この操作を関数として抽象化し、同じコードを書く代わりに関数を繰り返し使用します。

プログラミング言語は、関数、ジェネリクス、高階関数、インターフェースなど、さまざまな抽象化手段を提供します。この章ではジェネリクスと高階関数を紹介し、次の章でインターフェースについて議論します。

## ジェネリック関数とジェネリックデータ

まず、スタックデータ構造を見て、なぜそしてどのようにジェネリクスを使用するのかを理解しましょう。

スタックは一連のオブジェクトから構成されるコレクションであり、これらのオブジェクトの挿入と削除は後入れ先出し（LIFO）の原則に従います。例えば、左側の画像に示されている船に積まれたコンテナを考えてみてください。

![](/pics/stack-objects.drawio.webp)

明らかに、新しいコンテナは上に積まれ、コンテナを削除する際には上にあるものが最初に削除されます。つまり、最後に置かれたコンテナが最初に削除されることを意味します。同様に、右側の画像にある石の山を考えた場合、山を崩したくないなら、石を山の頂上に追加するか、最も最近追加された石を削除することしかできません。この構造がスタックです。日常生活にはこのような例が多くありますが、ここではすべてを列挙しません。

データ型スタックに対して、以下のような操作を定義できます。整数スタック`IntStack`を例にとると、新しい空のスタックを作成できます；スタックに整数を追加できます；スタックから要素を削除しようとすることができますが、スタックが空の場合もあるため、Optionでラップします。

```moonbit no-check
empty: () -> IntStack // create a new stack
push : (Int, IntStack) -> IntStack // add a new element to the top of the stack
pop: IntStack -> (Option[Int], IntStack) // remove an element from the stack
```

以下の図に示すように、2を追加してから2を削除します。このスタックの定義を単純に実装します。

![](/pics/stack-push-pop-en.drawio.webp)

```moonbit
enum IntStack {
  Empty
  NonEmpty(Int, IntStack)
}
fn IntStack::empty() -> IntStack { Empty }
fn push(self: IntStack, value: Int) -> IntStack { NonEmpty(value, self) }
fn pop(self: IntStack) -> (Option[Int], IntStack) {
  match self {
    Empty => (None, Empty)
    NonEmpty(top, rest) => (Some(top), rest)
  }
}
```

コードスニペットでは、最初の引数を`IntStack`とし、変数名を`self`としているため、関数呼び出しをチェーンできます。つまり、`pop(push(2, IntStack::empty()))`の代わりに`IntStack::empty().push(2).pop()`と書けます。この構文の深い意味は次の章で説明します。

コードに戻ると、スタック操作に基づいて再帰的なデータ構造を定義しました：スタックは空であるか、または要素とスタックで構成されます。スタックの作成は空のスタックを構築することです。要素の追加は、追加したい要素をトップに持つ非空のスタックを構築し、下のスタックはそのままにします。スタックからの削除にはパターンマッチングが必要で、スタックが空の場合、取得する値はありません；スタックが空でない場合、トップ要素を取得できます。

スタックの定義はリストの定義と非常に似ています。実際、MoonBitの組み込みライブラリでは、リストは本質的にスタックです。

整数用のスタックを定義した後、他の型のスタック、例えば文字列のスタックも定義したいと思うかもしれません。これは簡単で、ここでは説明せずにコードのみを示します。

```moonbit
enum StringStack {
  Empty
  NonEmpty(String, StringStack)
}
fn StringStack::empty() -> StringStack { Empty }
fn push(self: StringStack, value: String) -> StringStack { NonEmpty(value, self) }
fn pop(self: StringStack) -> (Option[String], StringStack) {
  match self {
    Empty => (None, Empty)
    NonEmpty(top, rest) => (Some(top), rest)
  }
}
```

確かに、文字列のスタックは整数のスタックとまったく同じように見えますが、型定義にいくつかの違いがあるだけです。しかし、さらに多くのデータ型を追加したい場合、各型に対してスタックデータ構造を再定義すべきでしょうか？明らかに、これは受け入れられません。

### MoonBitにおけるジェネリクス

したがって、MoonBitは重要な言語機能であるジェネリクスを提供しています。ジェネリクスとは、型をパラメータとして受け取ることで、より抽象的で再利用可能なデータ構造や関数を定義できるようにするものです。例えば、スタックの場合、名前の後に型パラメータ`T`を追加することで、実際に格納されるデータの型を示すことができます。

```moonbit
enum Stack[T] {
  Empty
  NonEmpty(T, Stack[T])
}
fn Stack::empty[T]() -> Stack[T] { Empty }
fn push[T](self: Stack[T], value: T) -> Stack[T] { NonEmpty(value, self) }
fn pop[T](self: Stack[T]) -> (Option[T], Stack[T]) {
  match self {
    Empty => (None, Empty)
    NonEmpty(top, rest) => (Some(top), rest)
  }
}
```

同様に、後で定義される関数にも`T`という型パラメータがあり、操作するスタックに格納されているデータの型と追加したいデータの型を表します。識別子をパラメータに置き換え、`T`を具体的な型に置き換えるだけで、実際のデータ構造と関数を得ることができます。例えば、`T`を`Int`に置き換えると、前に定義した`IntStack`が得られます。

### 例: ジェネリックなペア

すでに構文を紹介しましたが、さらに例を挙げます。

```moonbit
struct Pair[A, B]{ first: A; second: B }
fn identity[A](value: A) -> A { value }
```

例えば、データのペア、つまりタプルを定義できます。ペアには2つの型パラメータがあります。なぜなら、2つの異なる型の要素を持つ可能性があるからです。格納される値`first`と`second`はそれぞれこれらの型になります。別の例として、任意の型に対して動作し、常に入力値を返す関数`identity`を定義します。

`Stack`と`Pair`は、それ自体が型に対する関数と見なすことができます。パラメータは`T`または`A, B`であり、操作の結果は`Stack[T]`や`Pair[A, B]`などの具体的な型です。`Stack`と`Pair`は型コンストラクタと見なせます。ほとんどの場合、MoonBitでは型パラメータは具体的なパラメータの型に基づいて推論できます。

![](/pics/polymorphism-type.webp)

例えば、このスクリーンショットでは、`empty`の型は最初は不明です。しかし、`push(1)`を行った後、整数を保持するために使用されていることがわかり、`push`と`empty`の型パラメータが整数`Int`であると推論できます。

### 例: ジェネリックな関数型キュー

次に、別のジェネリックなデータ構造であるキューを見てみましょう。前回のレッスンで幅優先ソートでキューを使用しました。思い出してください、キューは先入れ先出し（FIFO）のデータ構造で、日常生活で並ぶ列のようなものです。ここでは以下の操作を定義します。キューは`Queue`と呼ばれ、型パラメータを持ちます。

```moonbit no-check
fn empty[T]() -> Queue[T] // Create an empty queue
fn push[T](q: Queue[T], x: T) -> Queue[T] // Add an element to the tail of the queue
// Try to dequeue an element and return the remaining queue; if empty, return itself
fn pop[T](q: Queue[T]) -> (Option[T], Queue[T])
```

すべての操作には型パラメータがあり、保持するデータの型を示します。スタックと同様に3つの操作を定義します。違いは、要素を削除する際に、キューに最初に追加された要素が削除される点です。

キューの実装はリストまたはスタックでシミュレートできます。要素をリストの末尾、つまりスタックの底に追加し、リストの先頭、つまりスタックの頂点から取り出します。削除操作は非常に高速です。なぜなら、パターンマッチングを1回行うだけで済むからです。しかし、要素を追加するにはリストまたはスタック全体を再構築する必要があります。

```moonbit no-check
Cons(1, Cons(2, Nil)) => Cons(1, Cons(2, Cons(3, Nil)))
```

ここに示すように、末尾に要素を追加する、つまり`Nil`を`Cons(3, Nil)`に置き換えるには、`Cons(2, Nil)`全体を`Cons(2, Cons(3, Nil))`に置き換える必要があります。さらに悪いことに、次のステップでは、元のリストの末尾として現れた`[2]`を`[2, 3]`に置き換える必要があり、リスト全体を最初から再構築することを意味します。これは非常に非効率的です。

この問題を解決するために、2つのスタックを使用してキューをシミュレートします。

```moonbit no-check
struct Queue[T] {
  front: Stack[T] // For removing elements
  back: Stack[T] // For storing elements
}
```

一方のスタックは要素の削除操作に使用し、もう一方はデータの格納に使用します。定義上、両方の型は`Stack[T]`であり、`T`はキューの型パラメータです。データを追加する際には、`back`に直接格納します。この操作は高速です。なぜなら、元の構造の上に新しい構造を構築するだけだからです。削除操作も同様に、1回のパターンマッチングのみで済み、遅延は発生しません。`front`の要素がすべて削除された場合、`back`の要素をすべて`front`に移動させる必要があります。各操作後にこのチェックを行い、キューが空でない限り`front`が空にならないことを保証します。このチェックはキュー操作の不変条件（invariant）であり、必ず満たされなければならない条件です。この移動操作はリストの長さに比例した非常にコストのかかる処理ですが、良いニュースはこのコストを償却できることです。なぜなら、一度移動させた後は、続くいくつかの削除操作では移動が必要なくなるからです。

![](/pics/queue_push.drawio.webp)

![](/pics/queue_push_more.drawio.webp)

![](/pics/queue_pop.drawio.webp)

具体的な例を見てみましょう。最初、キューは空なので、両方のスタックも空です。1回追加した後、`back`に数値を追加します。その後、キューを整理し、キューが空でないのに`front`が空であることがわかります。これは前述の不変条件を満たしていないため、`back`のスタックを回転させ、要素を`front`に移動させます。その後、引き続き`back`に要素を追加します。`front`が空でないため、不変条件を満たしており、追加の処理は不要です。

その後、繰り返し追加操作を行っても、`back`への新しい要素の追加は高速です。次に、`front`から要素を削除します。操作後に不変条件をチェックします。キューが空でないのに`front`が空であることがわかるため、再度`back`を回転させ、要素を`front`に移動させます。その後、通常通り`front`から要素を取り出すことができます。

わかるように、1回の回転操作で複数の削除操作をサポートできるため、全体のコストは毎回リストを再構築する場合よりも大幅に少なくなります。

```moonbit
struct Queue[T] {
  front: Stack[T]
  back: Stack[T]
}
fn Queue::empty[T]() -> Queue[T] { {front: Empty, back: Empty} }

// Store element at the end of the queue
fn push[T](self: Queue[T], value: T) -> Queue[T] {
  normalize({ ..self, back: self.back.push(value)}) // By defining the first argument as self, we can use xxx.f()
}

// Remove the first element
fn pop[T](self: Queue[T]) -> (Option[T], Queue[T]) {
  match self.front {
    Empty => (None, self)
    NonEmpty(top, rest) => (Some(top), normalize({ ..self, front: rest}))
  }
}

// If front is empty, reverse back to front
fn normalize[T](self: Queue[T]) -> Queue[T] {
  match self.front {
    Empty => { front: self.back.reverse(), back: Empty }
    _ => self
  }
}

// Helper function: reverse the stack
fn reverse[T](self: Stack[T]) -> Stack[T] {
  fn go(acc, xs: Stack[T]) {
    match xs {
      Empty => acc
      NonEmpty(top, rest) => go((NonEmpty(top, acc) : Stack[T]), rest)
    }
  }
  go(Empty, self)
}
```

以下はキューのコードです。ジェネリクスを広範に適用しているため、このキューは任意の型を含むことができ、他の要素を含むキューも可能です。ここにある関数は、先ほど説明したアルゴリズムの具体的な実装です。`push`関数では、`back.push()`を通じてスタックの`push`関数を呼び出しています。これについては次のレッスンで詳しく説明します。

## 高階関数

このセクションでは、MoonBitが提供する機能を活用して、繰り返しのコードを減らし、コードの再利用性を高める方法に焦点を当てます。それでは、例から始めましょう。

```moonbit
fn sum(list: @immut/list.T[Int]) -> Int {
  match list {
    Nil => 0
    Cons(hd, tl) => hd + sum(tl)
  }
}
```

リストに対するいくつかの操作を考えてみましょう。例えば、整数リストの合計を求める場合、構造的再帰を使用して次のコードを書きます。リストが空の場合は合計は0、それ以外の場合は現在の値に残りのリスト要素の合計を加算します。

```moonbit
fn length[T](list: @immut/list.T[T]) -> Int {
  match list {
    Nil => 0
    Cons(hd, tl) => 1 + length(tl)
  }
}
```

同様に、任意のデータ型のリストの長さを求める場合、構造的再帰を使用して次のように書きます。リストが空の場合は長さは0、それ以外の場合は1に残りのリストの長さを加算します。

これらの2つの構造にはかなりの類似点があることに気づきます。どちらも構造的再帰を使用しており、空の場合のデフォルト値を持ち、空でない場合は現在の値を処理し、残りのリストの再帰的結果と組み合わせます。合計の場合、デフォルト値は0で、二項演算は加算です。長さの場合、デフォルト値も0で、二項演算は現在の値を1に置き換えてから残りの結果に加算します。この構造を再利用するにはどうすればよいでしょうか？デフォルト値と二項演算をパラメータとして受け取る関数として記述できます。

### MoonBitにおける第一級関数

ここで、MoonBitにおいて関数が第一級市民(first-class citizen)であるという点に着目します。これは、関数を引数として渡したり、結果として返したりできることを意味します。例えば、先ほど説明した構造は以下の関数として定義できます。ここで、`f`は引数として渡され、4行目で計算に使用されます。

```moonbit
fn fold_right[A, B](list: @immut/list.T[A], f: (A, B) -> B, b: B) -> B {
  match list {
    Nil => b
    Cons(hd, tl) => f(hd, fold_right(tl, f, b))
  }
}
```

別の例を見てみましょう。関数の操作を繰り返したい場合、1行目に示すように`repeat`を定義できます。`repeat`は関数を引数として受け取り、関数を結果として返します。その操作結果は、元の関数を2回計算する関数となります。

```moonbit
fn repeat[A](f: (A) -> A) -> (A) -> A {
  fn (a) { f(f(a)) } // Return a function as a result
}

fn plus_one(i: Int) -> Int { i + 1 }
fn plus_two(i: Int) -> Int { i + 2 }

let add_two: (Int) -> Int = repeat(plus_one) // Store a function

let compare: Bool = add_two(2) == plus_two(2) // true (both are 4)
```

例えば、`plus_one`と`plus_two`という2つの関数がある場合、`plus_one`を引数として`repeat`を使用すると、結果は1を2回加算する、つまり2を加算する関数になります。この関数を`add_two`に`let`でバインドし、通常の関数構文を使って計算を実行して結果を得ます。

`let add_two: (Int) -> Int = repeat(plus_one)`

&nbsp; `repeat(plus_one)`

$\mapsto$ `fn (a) { plus_one(plus_one(a)) }`

`let x: Int = add_two(2)`

&nbsp; `add_two(2)`

$\mapsto$ `plus_one(plus_one(2))`

$\mapsto$ `plus_one(2) + 1`

$\mapsto$ `(2 + 1) + 1`

$\mapsto$ `3 + 1`

$\mapsto$ `4`

ここでの簡約化を詳しく見ていきましょう。まず、`add_two`は`repeat(plus_one)`にバインドされます。この行では、式内の識別子を引数で置き換える簡約化が行われ、結果として関数が得られます。この時点では、この式についてこれ以上簡約化できません。次に、`add_two(2)`を計算します。同様に式内の識別子を置き換え、`plus_one`を簡約化します。さらに簡約化を進めると、最終的に結果の`4`が得られます。

これまでに関数の型について触れましたが、これは受け取るパラメータから出力パラメータへの対応を示すもので、受け取るパラメータは括弧で囲まれます。

- `(Int) -> Int` 整数から整数への関数
- `(Int) -> (Int) -> Int` 整数から、整数を受け取り整数を返す関数への関数
- `(Int) -> ((Int) -> Int)` 前の行と同じ
- `((Int) -> Int) -> Int` 整数から整数への関数を受け取り、整数を返す関数

例えば、整数から整数への関数型は`(Int) -> Int`となります。2行目は整数から関数への例です。関数のパラメータも括弧で囲む必要があることに注意してください。この関数型は実際には、3行目に見られるように、後続の関数型全体を括弧で囲んだものと等価です。関数から整数への場合、前述のように受け取るパラメータは括弧で囲む必要があるため、2行目のようではなく4行目のように記述すべきです。

### 例: 畳み込み関数

高階関数のさらに一般的な応用例をいくつか紹介します。高階関数とは関数を受け取る関数のことです。先ほど見た`fold_right`はその典型的な例です。以下にその式木を示します。

```moonbit no-check
fn fold_right[A, B](list: @immut/list.T[A], f: (A, B) -> B, b: B) -> B {
  match list {
    Nil => b
    Cons(hd, tl) => f(hd, fold_right(tl, f, b))
  }
}
```

![](/pics/fold_right.drawio.webp)

1から3までのリストに対して、`f`が各要素と残りの要素の結果に適用されていく様子がわかります。これは右から左へと一つずつ畳み込んでいき、最終的に結果を得るような構造です。そのため、この関数は`fold_right`と呼ばれます。もし方向を変えて、リストを左から右に畳み込むと、`fold_left`が得られます。

```moonbit
fn fold_left[A, B](list: @immut/list.T[A], f: (B, A) -> B, b: B) -> B {
  match list {
    Nil => b
    Cons(hd, tl) => fold_left(tl, f, f(b, hd))
  }
}
```

![](/pics/fold_left.drawio.webp)

ここでは、順序を入れ替えるだけで、まず現在の要素を前回の累積結果で処理し、その後処理された結果を次の処理に組み込むだけで済みます。これは4行目に示されている通りです。この関数は左から右へ畳み込みを行います。

### 例: Map関数

高階関数のもう一つの一般的な応用例は、関数の各要素をマッピングすることです。

```moonbit no-check
struct PersonalInfo { name: String; age: Int }
fn map[A, B](self: @immut/list.T[A], f: (A) -> B) -> @immut/list.T[B] {
  match list {
    Nil => Nil
    Cons(hd, tl) => Cons(f(hd), map(tl, f))
  }
}
let infos: @immut/list.T[PersonalInfo] = ???
let names: @immut/list.T[String] = infos.map(fn (info) { info.name })
```

例えば、人物情報があり名前だけが必要な場合、マッピング関数`map`を使用できます。この関数はパラメータとして`f`を受け取り、リスト内の各要素を一つずつマッピングし、最終的に要素の型が`B`になった新しいリストを取得します。この関数の実装は非常にシンプルです。必要なのは構造的再帰だけです。最後の応用例は8行目に示されています。この`map`構造を見覚えがあるかもしれません: 構造的再帰、空の場合のデフォルト値、空でない場合に現在の値を再帰結果と組み合わせて処理する二項演算です。実際、`map`は`fold_right`を使って完全に実装できます。ここでデフォルト値は空リストで、二項演算は`Cons`コンストラクタです。

```moonbit
fn map[A, B](list: @immut/list.T[A], f: (A) -> B) -> @immut/list.T[B] {
  fold_right(list, fn (value, cumulator) { Cons(f(value), cumulator) }, Nil)
}
```

ここで練習問題を残します: `fold_right`を使って`fold_left`をどのように実装するか？ヒント: `Continuation`と呼ばれるものが関わってくるかもしれません。`Continuation`は現在の操作後の残りの計算を表し、一般的にはパラメータが現在の値で戻り値がプログラム全体の結果である関数です。

ジェネリクスと高階関数について学んだので、前回のレッスンで学んだ二分探索木を、整数だけでなく様々なデータ型を格納できるより一般的な二分探索木として定義できるようになりました。

```moonbit no-check
enum Tree[T] {
  Empty
  Node(T, Tree[T], Tree[T])
}

// We need a comparison function to determine the order of values
// The comparison function should return an integer representing the comparison result
// -1: less than; 0: equal to; 1: greater than
fn insert[T](self: Tree[T], value: T, compare: (T, T) -> Int) -> Tree[T]
fn delete[T](self: Tree[T], value: T, compare: (T, T) -> Int) -> Tree[T]
```

ここで、データ構造自体は格納するデータ型を表す型パラメータを受け取ります。二分探索木は順序付けされている必要があるため、この特定の型をどのようにソートするかを知る必要があります。したがって、比較関数をパラメータとして受け取ります。この関数は、比較結果を小なり、等しい、大なりとして表す整数を返す必要があります。コードに示されている通りです。実際、MoonBitの別の機能を使用してこのパラメータを省略することも完全に可能です。これについては次のレッスンで紹介します。

## まとめ

この章では、ジェネリクスと第一級関数としての関数の概念を紹介し、MoonBitでそれらを使用する方法を見てきました。また、データ構造スタックとキューの実装についても議論しました。

さらに詳しく知りたい場合は、以下を参照してください:

- _**Software Foundations, Volume 1: Logical Foundations**_: Poly; または
- _**Programming Language Foundations in Agda**_: Lists