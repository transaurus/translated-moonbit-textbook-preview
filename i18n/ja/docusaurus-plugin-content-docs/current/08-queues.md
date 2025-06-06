# 8. キュー

キューはコンピュータサイエンスにおける基本的なデータ構造で、先入れ先出し（FIFO）の原則に従います。つまり、キューに最初に追加された要素が最初に取り除かれます。この章では、キューの実装方法として主に2つの方法、循環キューと単方向リンクリストを探求します。また、再帰関数の最適化に不可欠な末尾呼び出しと末尾再帰についても紹介します。

## 基本操作

キューは以下の基本操作をサポートする必要があります：

```moonbit no-check
struct Queue { .. }

fn make() -> Queue // Creates an empty queue.
fn push(self: Queue, t: Int) -> Queue // Adds an integer element to the queue.
fn pop(self: Queue) -> Queue // Removes an element from the queue.
fn peek(self: Queue) -> Int // Views the front element of the queue.
fn length(self: Queue) -> Int // Obtains the length of the queue.
```

`pop`と`push`操作は元のキューを直接変更します。利便性のため、変更後のキューを返すようにしており、これによりチェーン呼び出しが容易になります。

```moonbit no-check
make().push(1).push(2).push(3).pop().pop().length() // 1
```

## 循環キュー

循環キューは通常、配列を使用して実装されます。配列は連続した記憶領域を提供し、各位置を変更可能です。一度割り当てられた配列の長さは固定されます。

以下のコードスニペットは、MoonBitの組み込み`Array`の基本的な使用方法を示しています。インデックスが0から始まることに注意してください。この利点については後で学びます。

```moonbit expr
let a: Array[Int] = Array::make(5, 0)
a[0] = 1
a[1] = 2
println(a) // Output: [1, 2, 0, 0, 0]
```

循環キューの実装では、`start`と`end`インデックスを追跡します。新しい要素がプッシュされるたびに、`end`インデックスを1つ進めます。`pop`操作は`start`位置の要素をクリアし、`start`を1つ進めます。インデックスが配列の長さを超えた場合、先頭に戻ります。

以下の図はこれらの操作のデモンストレーションです。まず、`make()`を呼び出して空のキューを作成します。この時点で、`start`と`end`インデックスはともに最初の要素を指しています。次に、`push(1)`を呼び出して要素`1`をキューに追加します。要素`1`は`end`が指す位置に追加され、`end`インデックスが更新されます。その後、`push(2)`を呼び出して別の要素をキューに追加し、最後に`pop()`を呼び出して最初にプッシュした要素を取り出します。

![](/pics/circle_list.drawio.webp)

次に、配列の終端に近い場合の別の状況を見てみましょう。この時点で、`end`は配列の最後の要素の位置を指しています。要素をプッシュする際、`end`はこれ以上進めないため、配列の先頭に戻ります。その後、2回の`pop`操作を実行します。同様に、`start`が配列の長さを超えた場合、リストの先頭に戻ります。

![](/pics/circle_list_back.drawio.webp)

上記の基本的な考え方を踏まえると、`Queue`構造体を簡単に定義し、`push`操作を以下のように実装できます：

```moonbit
struct Queue {
  mut array: Array[Int]
  mut start: Int
  mut end: Int // end points to the empty position at the end of the queue
  mut length: Int
}

fn push(self: Queue, t: Int) -> Queue {
  self.array[self.end] = t
  self.end = (self.end + 1) % self.array.length() // wrap around to the start of the array if beyond the end
  self.length = self.length + 1
  self
}
```

ここで、配列の開始インデックスとして`0`を使用する利点が明らかになります：`%`演算を用いて簡単に「循環」効果を実現できます。

ただし、上記の実装には潜在的な問題があります。それは、キュー内の要素数が配列の長さを超える可能性があることです。要素数の上限がわかっている場合、十分な長さの配列を定義することでこの問題を回避できます。事前に上限がわからない場合は、より長い配列に「拡張」することで対応できます。サンプル実装は以下の通りです：

```moonbit
fn _push(self: Queue, t: Int) -> Queue {
  if self.length == self.array.length() {
    let new_array: Array[Int] = Array::make(self.array.length() * 2, 0)
    let mut i = 0
    while i < self.array.length() {
      new_array[i] = self.array[(self.start + i) % self.array.length()]
      i = i + 1
    }
    self.start = 0
    self.end = self.array.length()
    self.array = new_array
  }
  self.push(t)
}
```

要素を取り出す際、`start`が指す要素を削除し、`start`を進め、長さを更新します。

```moonbit
fn pop(self: Queue) -> Queue {
  self.array[self.start] = 0
  self.start = (self.start + 1) % self.array.length()
  self.length = self.length - 1
  self
}
```

`length`関数は、動的に維持されているキューの現在の`length`フィールドを単に返します。

```moonbit
fn length(self: Queue) -> Int {
  self.length
}
```

### ジェネリックな循環キュー

時には、キュー内の要素を`Int`型以外にしたい場合があります。MoonBitのジェネリック機構を利用すれば、`Queue`構造体のジェネリック版を簡単に定義できます。

```moonbit no-check
struct Queue[T] {
  mut array: Array[T]
  mut start: Int
  mut end: Int // end points to the empty position at the end of the queue
  mut length: Int
}
```

しかし、`make`操作を実装する際に、配列の初期値をどのように指定すればよいでしょうか？選択肢は2つあります。1つは`Option`で値をラップし、`Option::None`を初期値として使用する方法。もう1つは型自体のデフォルト値を使用する方法です。このデフォルト値をどのように定義し利用するかについては、第9章で議論します。

```moonbit no-check
fn make[T]() -> Queue[T] {
  {
    array: Array::make(5, T::default()), // Initialize the array with the default value of the type
    start: 0,
    end: 0,
    length: 0
  }
}
```

## 単方向リンクリスト

単方向リンクリストはノードで構成されており、各ノードは値と次のノードへの可変参照を含んでいます。

`Node[T]`はリンクリストのノードを表すジェネリック構造体です。`val`と`next`の2つのフィールドを持ちます。`val`フィールドはノードの値を格納するために使用され、その型は任意の有効なデータ型である`T`です。`next`フィールドはリンクリストにおける次のノードへの参照を表します。これはオプショナルフィールドで、次のノードへの参照を保持するか、空（`None`）にすることができ、後者の場合はリンクリストの終端を示します。

```moonbit
struct Node[T] {
  val : T
  mut next : Option[Node[T]]
}
```

`LinkedList[T]`はリンクリストを表すジェネリック構造体です。`head`と`tail`という2つの可変フィールドを持ちます。`head`フィールドはリンクリストの最初のノード（先頭）への参照を表し、リンクリストが空のときは最初に`None`に設定されます。`tail`フィールドはリンクリストの最後のノード（末尾）への参照を表し、同様に最初は`None`に設定されます。`tail`フィールドがあることで、リンクリストの末尾への新しいノードの追加を効率的に行うことができます。

```moonbit
struct LinkedList[T] {
  mut head : Option[Node[T]]
  mut tail : Option[Node[T]]
}
```

`make`の実装は自明です：

```moonbit
fn LinkedList::make[T]() -> LinkedList[T] {
  { head: None, tail: None }
}
```

要素をプッシュするとき、まずリンクリストが空かどうかを確認します。空でない場合、キューの末尾に追加し、リンクリストの関係を維持します。

```moonbit
fn push[T](self: LinkedList[T], value: T) -> LinkedList[T] {
  let node = { val: value, next: None }
  match self.tail {
    None => {
      self.head = Some(node)
      self.tail = Some(node)
    }
    Some(n) => {
      n.next = Some(node)
      self.tail = Some(node)
    }
  }
  self
}
```

以下の図は簡単なデモンストレーションです。`make()`を呼び出してリンクリストを作成すると、`head`と`tail`の両方が空になります。`push(1)`を使用して要素をプッシュすると、新しいノードを作成し、`head`と`tail`の両方をこのノードに向けます。さらに要素をプッシュする場合、例えば`push(2)`、そして`push(3)`を呼び出すと、現在の`tail`ノードの`next`フィールドを更新して新しいノードを指すようにする必要があります。リンクリストの`tail`ノードは常に最新のノードを指すようにします。

![](/pics/linked_list.drawio.webp)

![](/pics/linked_list_2.drawio.webp)

リストの長さを取得するには、循環キューで行ったように構造体に記録するか、あるいは単純な再帰関数を使用して計算することができます。

```moonbit
fn length[T](self : LinkedList[T]) -> Int {
  fn aux(node : Option[Node[T]]) -> Int {
    match node {
      None => 0
      Some(node) => 1 + aux(node.next)
    }
  }
  aux(self.head)
}
```

### スタックオーバーフロー

しかし、リストが長すぎる場合、上記の実装ではスタックオーバーフローが発生する可能性があります。

```moonbit no-check
fn init {
  let list = make()
  let mut i = 0
  while i < 100000 {
    let _ = list.push(i)
    i += 1
  }
  println(list.length())
}
```

![](/pics/overflow.webp)

スタックオーバーフローの原因は、関数を再帰的に呼び出すたびに、現在の関数呼び出し層で計算待ちのデータ（または環境）が「スタック」と呼ばれるメモリ空間に一時的に保存されることです。より深い再帰から現在の層に戻るとき、スタック空間に保存されたデータが復元され、現在の関数の計算が続行されます。このケースでは、`aux`を再帰的に呼び出すたびに、部分式`1 + ...`の情報をスタック空間に保存するため、再帰呼び出しから戻った後に計算を続行できるようになります。

しかし、スタック領域のサイズには限界があります。そのため、再帰のたびにデータをスタックに保存すると、再帰の階層が十分に深くなった時点でスタックオーバーフローが発生します。

### 末尾呼び出しと末尾再帰

スタックオーバーフローの問題を回避するためには、再帰関数をループに書き換えてスタック領域を使用しないようにする方法があります。あるいは、再帰関数の実装を調整して、各再帰呼び出しでスタックに環境を保存する必要がないようにすることも可能です。これを実現するには、関数の最後の操作が関数呼び出し（末尾呼び出し）であることを保証する必要があります。特に、最後の操作が関数自身の再帰呼び出しである場合、それは末尾再帰と呼ばれます。関数呼び出しの結果が計算の最終結果となるため、現在の計算環境を保存する必要がなくなります。

以下のコードに示すように、`length`関数を末尾再帰になるように書き換えることができます：

```moonbit
fn length_[T](self: LinkedList[T]) -> Int {
  fn aux2(node: Option[Node[T]], cumul: Int) -> Int {
    match node {
      None => cumul
      Some(node) => aux2(node.next, 1 + cumul)
    }
  }
  // tail recursive
  aux2(self.head, 0)
}
```

最適化された`length_`関数は、末尾再帰を使用して連結リストの長さを計算し、スタックオーバーフローのリスクを回避します。`aux2`関数は、アキュムレータ`cumul`を使用してリストを走査しながら長さを追跡します。各反復の最後の関数呼び出しは`aux2`自身であるため、スタックオーバーフローの問題を起こすことなく、必要な回数だけ再帰的に呼び出すことができます。

## まとめ

この章では、循環キューと単方向連結リストという2つの基本的なキュー型の設計と実装について説明しました。また、再帰関数を最適化し、スタックオーバーフローを防ぐために末尾呼び出しと末尾再帰を理解・活用することの重要性にも触れ、より効率的で安定したプログラム性能を実現する方法を紹介しました。