# 10. ハッシュマップとクロージャ

## 復習

マップ（またはテーブル）は、キーと値を結びつけるキー・バリューペアの集合であり、各キーは一意です。マップの単純な実装としては、タプルのリストがあり、各タプルがキー・バリューペアとなっています。新しいキー・バリューペアはリストの先頭に追加され、検索操作ではリストの先頭から走査します。

別の実装方法として、[第5章](./trees)で紹介した平衡二分木（BBT）を基にしたものがあります。BBTを修正して、各ノードがキー・バリューペアを保持するようにします。木の操作では、キー・バリューペアの最初のパラメータと操作対象のキーを比較します。

## ハッシュマップ

### ハッシュ関数

まず、ハッシュ関数またはハッシュとは何でしょうか？ハッシュ関数は、任意の長さのデータを固定長のデータにマッピングまたは結びつけます。例えば、MD5アルゴリズムは、任意のサイズや形式のファイルを短い128ビットのダイジェスト（圧縮されたデータ表現）にマッピングします。

MoonBitの`Hash`インターフェースでは、データは整数の範囲の値にマッピングされます。例えば、文字列"ThisIsAVeryVeryLongString"は整数-321605584にマッピングされます。

```moonbit no-check
trait Hash { hash(Self) -> Int }
"ThisIsAVeryVeryLongString".hash() == -321605584
```

### ハッシュマップのデータ構造

ハッシュマップはこのメカニズムを利用して、データをハッシュ値にマッピングし、そのハッシュ値を配列のインデックスにマッピングすることで、効率的にデータを処理します。これにより、データの追加、検索、更新が高速に行えます。なぜなら、配列へのランダムアクセスは現代のコンピュータで最も効率的な操作だからです。ハッシュマップの操作は理想的には定数時間で行われます。つまり、実行時間はデータ入力サイズの増加に伴って増加しません（入力サイズに依存しません）。一方、平衡二分木の操作は対数時間です。以下は、ハッシュマップのマッピングメカニズムを示す疑似コードの例です。

```moonbit no-check
// For a: Array[(Key, Value)], key: Key, value: Value
let index = key.hash() % a.length() // key value--hashing-->hash value--modulo operation-->index in array
a[ index ] = value // add or update data
let value = a[ index ] // look up data
```

キー・バリューペアの配列があり、データを追加、更新、または検索したいとします。まず、キーに基づいてハッシュ値を計算します。ハッシュ値は任意の整数になり得るため、モジュロ演算を使用してハッシュ値を配列のインデックスにマッピングし、対応する配列インデックスでデータを検索または更新します。ただし、前述のように、これは理想的なシナリオであり、ハッシュ衝突が発生する可能性があります。

## ハッシュ衝突

[鳩の巣原理](https://en.wikipedia.org/wiki/Pigeonhole_principle)や[誕生日の問題](https://en.wikipedia.org/wiki/Birthday_problem)によると、マッピングするデータ量が整数の範囲を超える可能性があり、ハッシュ値が有効な配列インデックスをはるかに超える場合があります。例えば、明らかに21億スロットの配列を直接割り当てることはできず、異なるデータが同じ配列インデックスを持つ衝突が発生します（異なるデータが同じハッシュ値を持つ場合や、異なるハッシュ値が同じ配列インデックスにマッピングされる場合）。ハッシュ衝突を処理する方法はいくつかあります。

1つのアプローチは**直接アドレッシング**です。データを計算した配列インデックスに対応するスロットに格納する必要がある場合、異なるデータが同じスロットに格納され問題が発生する可能性があります。そのため、各スロットで別のデータ構造を使用して、同じインデックスにハッシュされたアイテムを格納します。使用可能なデータ構造にはリストや平衡二分木があり、元の配列はリストや木の配列に変わります。

別のアプローチは**開放アドレッシング**で、配列の型を変更せず、配列が直接データを格納しますが、データを格納する空きスロットを見つけるための特定のルールに従います。例えば、線形探査法を使用して、元のスロットから開始して次の空きスロットを見つけてデータを格納します。二次探査法では、インデックスを$1^2$、$2^2$、$3^2$...と増やして空きスロットを見つけます。この講義では、リストを使用した直接アドレッシングと線形探査法を使用した開放アドレッシングに焦点を当てます。

### 直接アドレッシング

まず直接アドレッシングから始めましょう。ハッシュ/インデックスの衝突が発生した場合、同じインデックスのデータをリストなどのデータ構造に格納します。例えば、長さ5の配列に0と5（それぞれハッシュ値が0と5）を追加する場合、0と5 mod 長さ5はどちらも0になるため、インデックス0のリストに追加されます：

![](/pics/separate_chaining.drawio.webp)

実装では、2つの追加データ構造を定義します：1) インプレースでの値更新を可能にするキーと値のペア、2) null値が空のリストを意味し、それ以外の場合タプルが先頭要素と残りのリストとなる可変リスト。最後に、ハッシュマップを定義します。これはキーと値のペアのリストの配列を含み、配列の長さとキーと値のペアの数を動的に維持します。

```moonbit
struct Entry[K, V] { // Struct for key-value pair storage
  key : K
  mut value : V // In-place update enabled
}

struct Bucket[V] { // Collection that can store key-value pairs
  mut val : Option[(V, Bucket[V])] // In-place addition, removal enabled
}

struct HT_bucket[K, V] {
  mut values : Array[Bucket[Entry[K, V]]] // List of key-value pairs, array of lists
  mut length : Int // Length of array dynamically maintained
  mut size : Int // Number of key-value pairs in the hash map dynamically maintained
}
```

追加/更新操作では、まずキーのハッシュ値に基づいて格納位置を計算します。次に、リストを走査してキーが既に存在するかどうかを調べます。キーが存在する場合は値を更新し、存在しない場合はキーと値のペアを追加します。同様に、対応するリストをチェックし、削除操作ではリストを更新します。

![height:200px](/pics/separate_chaining_op_en.drawio.webp)

以下のコードはデータの追加と更新を示しています。1行目の`K : Hash`で指定されたハッシュインターフェースを使用して、2行目でキーのハッシュ値を計算します。次に対応するデータ構造を見つけて走査します。4行目で無限whileループを使用した可変データ構造を使用しています。キーが既に存在するか、リストの終端に到達した場合にループを抜けます。キーが見つかった場合、データをインプレースで更新します。見つからなかった場合、バケットを残りのリストに更新してループを終了させます。リストの終端に到達してもキーが見つからなかった場合、リストの末尾に新しいデータペアを追加します。最後に、現在の負荷係数に基づいてリサイズが必要かどうかをチェックします。

```moonbit
let load = 0.75
fn resize() -> Unit {} // placeholder for resize implementation

fn put[K : Hash + Eq, V](map : HT_bucket[K, V], key : K, value : V) -> Unit {
  let index = key.hash() % map.length // Calculate the index
  let mut bucket = map.values[index] // Get the corresponding data structure
  while true {
    match bucket.val {
      None => { // If doesn't exist, add and exit loop
        bucket.val = Some(({ key, value }, { val: None }))
        map.size = map.size + 1
        break
      }
      Some((entry, rest)) => {
        if entry.key == key { // If exists, update the value
          entry.value = value
          break
        } else { // Otherwise, update bucket so the loop terminates
          bucket = rest
  } } } }
  if map.size.to_double() / map.length.to_double() >= load { // Resize according to the load factor
    resize()
  }
}
```

なぜリサイズが必要なのか疑問に思うかもしれません。リストは際限なく成長できるからです。まず負荷係数の概念を紹介しましょう：キーと値のペアの数と配列の長さの比率です。基本的に、負荷係数が高いほど衝突が多くなり、リンクリストが長くなり、追加、検索、更新、削除の操作が遅くなります。しかし、定数時間での効率的な操作のために配列を選択しました。そのため、しきい値を設定し、負荷係数がそれを超えた場合により大きな配列を再割り当てしてリストの長さを短くします。ただし、しきい値が高すぎると長いリストが発生しやすくなり、リストの走査が遅くなりパフォーマンスが低下します。逆にしきい値が低すぎると、配列のリサイズと各要素のハッシュ値の再計算による再割り当てに時間がかかります。

次に、削除操作を簡単に見ていきましょう。追加/更新操作と同様に、ハッシュとインデックスを計算し、対応するリストを見つけて走査します。走査が終了したらループを抜けます。各ループで、探しているデータかどうかをチェックします。見つかった場合、格納された値を削除してリストをインプレースで変更し、ハッシュマップのサイズを更新します。見つからなかった場合、走査を続けます。直接アドレッシングの検索とリサイズ操作は比較的単純なので、練習として試してみてください。

```moonbit
fn remove[K : Hash + Eq, V](map : HT_bucket[K, V], key : K) -> Unit {
  let index = key.hash() % map.length // Calculate the index
  let mut bucket = map.values[index] // Get the corresponding data structure
  while true {
    match bucket.val {
      None => break // Exit after finishing traversal
      Some((entry, rest)) => {
        if entry.key == key { // Remove if exists
          bucket.val = rest.val // { Some(entry, { val }) } -> { val }
          map.size = map.size - 1
          break
        }
        else { // Otherwise, continue traversal
          bucket = rest
  } } } } }
```

### オープンアドレッシング

オープンアドレッシングについて続けて説明します。線形探査法とは、ハッシュ衝突が発生した場合、インデックスをインクリメントしながら次の空きスロットを見つけ、衝突したキーを配置する方法です。次の例では、まずハッシュ値が0の0をスロット0に追加します。次に、ハッシュ値が1の1をスロット1に追加します。最後に、ハッシュ値が5の5を追加しますが、インデックスの範囲を超えているためモジュロ演算で0を得ます。理論的には5をスロット0に格納すべきですが、このスロットは既に占有されています。そのため、インデックスをインクリメントして次の空きスロット（スロット2、スロット1も占有されていたため）を見つけます。プログラム全体を通じて不変条件を維持する必要があります：元のスロットとキーと値のペアが実際に格納されているスロットの間に空きスロットがあってはなりません。これにより、あるキーと値のペアが既に存在するかどうかを確認するためにハッシュマップ全体を走査する無駄な時間を省けます。また、スロット間に隙間がないことを確認しているため、次の空きスロットが見つかればループを終了できます。

![height:300px](/pics/open_address_en.drawio.webp)

オープンアドレッシングを実装するために、前回の講義で紹介した循環キューと同様に、デフォルト値を持つ配列を使用します。Optionを使用した実装にも挑戦してみてください。キーと値のペアを格納する配列に加えて、現在のスロットが空であるかどうかを判断するブール値の配列も用意します。通常通り、配列の長さとキーと値のペアの数を動的に管理します。

```moonbit no-check
struct Entry[K, V] { // Struct for key-value pair storage
  key : K
  mut value : V // In-place update enabled
} derive(Default)

struct HT_open[K, V] {
  mut values : Array[Entry[K, V]] // Array of key-value pairs
  mut occupied : Array[Bool] //  Array denoting whether the current slot is empty
  mut length : Int // Length of array dynamically maintained
  mut size : Int // Number of key-value pairs in the hash map dynamically maintained
}
```

追加/更新操作では、キーのハッシュ値に基づいてデータを追加/更新するインデックスを計算します。スロットが空でない場合、探しているキーかどうかをさらに確認します。キーが見つかった場合は値を更新し、そうでない場合は後方に探査を続け、空きスロットを見つけたらキーと値のペアを格納します。ここでは、必要に応じて配列のサイズを変更するため、空きスロットが存在すると仮定できます。「後方」への走査は、循環キューでの走査と同じです。インデックスが配列の長さを超えた場合、配列の先頭に戻ります。

キーが既に存在するかどうかを確認するヘルパーメソッドを定義できます。キーが存在する場合はそのインデックスを直接返し、存在しない場合は次の空きスロットのインデックスを返します。

```moonbit no-check
// Probe to the right of the index of the original hash, return the index of the first empty slot
fn find_slot[K : Hash + Eq, V](map : HT_open[K, V], key : K) -> Int {
  let hash = key.hash() // Hash value of the key
  let mut i = hash % map.length // Index to be stored at if there's no hash collision
  while map.occupied[i] {
    if map.values[i].key == key { // If a key already exists, return its index
      return i
    }
    i = (i + 1) % map.length
  }
  return i // Otherwise, return when an empty slot occurs
}
```

次に、このヘルパー関数を利用して追加/更新、検索、削除操作を定義します。追加/更新操作では、まず計算されたインデックスのスロットが空かどうかを確認してキーが存在するかどうかを判断します。キーが見つかった場合は対応する値を更新し、見つからなかった場合は空きスロットにキーと値のペアを追加し、占有状態とハッシュマップのサイズを更新します。最後に、サイズ変更が必要かどうかを確認します。

```moonbit no-check
fn put[K : Hash + Eq + Default, V : Default](map : HT_open[K, V], key : K, value : V) -> Unit {
  let index = find_slot(map, key) // Use helper method to look up the key
  if map.occupied[index] { // Check for key or empty slot
    map.values[index].value = value // Update if the key already exists
  } else { // Otherwise, add the key-value pair into the empty slot
    map.occupied[index] = true
    map.values[index] = { key, value }
    map.size = map.size + 1
  }
  // Check the load factor to determine if resizing is needed
  if map.size.to_double() / map.length.to_double() >= 0.75 {
    resize(map) // fn resize(map) -> Unit
  }
}
```

削除操作はより複雑です。維持すべき不変条件を思い出してください：元のスロットとキーと値のペアが実際に格納されているスロットの間に空きスロットがあってはなりません。以下に示すように、0、1、5、3を順番に追加した後1を削除すると、0と5の位置の間に隙間が生じ、不変条件に違反し、5を正しく検索できなくなります。

簡単な解決策は、スロットを「削除済み」としてマークする特別な状態を定義し、後続のデータが到達可能で検索可能であることを保証することです。もう一つの解決策は、データ削除のスロットから次の空きスロットまでの要素が位置を移動する必要があるかどうかを確認し、不変条件を維持することです。ここではより簡単なマーキング方法、いわゆる「墓石」法を紹介します。

![height:320px](/pics/open_address_delete_en.drawio.webp)

`Empty`、`Occupied`、`Deleted`からなる新しい`Status`列挙型を定義し、占有配列の型をブール値からStatusに更新します。

```moonbit
enum Status {
  Empty
  Deleted // Add the "deleted status
  Occupied
}

struct HT_open[K, V] {
  mut values : Array[Entry[K, V]]
  mut occupied : Array[Status] // Change from boolean to Status
  mut length : Int
  mut size : Int
}
```

ヘルパー関数も更新して、キーまたは空きスロットの検索時に、ステータスが`Empty`または`Deleted`の最初の空きスロットを記録し、データ削除後にそのスロットを再利用できるようにします。ただし、キーが存在しないかどうかを判断するためには、次の`Empty`スロットを見つける必要があります。ここでは`empty`というシンプルな変数を使用して記録します。負の値はまだ空きスロットが見つかっていないことを意味し、空きスロットが見つかった場合はそのインデックスに更新します。また、ループが終了した時点で空きスロットに遭遇したことを意味し、変数`empty`に基づいて返す値を決定します。

```moonbit
// Probe to the right of the index of the original hash, return the index of the first empty slot
fn find_slot[K : Hash + Eq, V](map : HT_open[K, V], key : K) -> Int {
  let index = key.hash() % map.length
  let mut i = index
  let mut empty = -1 // Record the first empty slot occurred: status Empty or Deleted
  while (physical_equal(map.occupied[i], Empty)).not() {
    if map.values[i].key == key {
      return i
    }
    if physical_equal(map.occupied[i], Deleted) && empty != -1 { // Update empty slot
      empty = i
    }
    i = (i + 1) % map.length
  }
  return if empty == -1 { i } else { empty } // Return the first empty slot
}
```

削除操作はよりシンプルで、ヘルパー関数の結果に従ってステータスインジケーターを更新するだけです。このアプローチでは、追加と削除を繰り返すと多くの`Deleted`スロットが生じるため、追加の検索時間/オーバーヘッドがかかることに注意が必要です。そのため、後で要素を再配置する必要があります。

```moonbit no-check
fn remove[K : Hash + Eq + Default, V : Default](map : HT_open[K, V], key : K) -> Unit {
  let index = find_slot(map, key)
  if physical_equal(map.occupied[index], Occupied) {
    map.values[index] = default()
    map.occupied[index] = Deleted
    map.size = map.size - 1
  }
}
```

次に、オープンアドレッシングの別の実装として、削除ごとに要素を再配置して検索パスを圧縮する方法を紹介します。0、1、5、3を順番に追加した後、1を削除する場合を考えます。1より前の要素については不変条件が維持されていますが、1より後の要素については不変条件が維持されているかどうか確信が持てません。これらの要素は元々このスロットに格納されていたか、元のスロットが占有されていたため次の空きスロットに格納された可能性があります。そのため、チェックが必要です。

まず要素5をチェックすると、5はインデックス0にマップされるべきですが、ハッシュ衝突を処理するために現在のスロットに格納されています。要素1が削除されたため、インデックス0と2の間に空きスロットが生じ、不変条件が維持されていません。これを解決するには、要素5を前に移動して要素1が格納されていたインデックスに配置します。次に要素3をチェックすると、マップされるべきスロットに既にあるため移動しません。要素3の後に空きスロットがあるため、それ以降の要素は影響を受けないのでチェックを終了します。

![](/pics/rearrange_en.drawio.webp)

別の例を見てみましょう：サイズ10の配列があり、*n*で終わる数値はモジュロ演算によりインデックス*n*にマップされます（例：要素0はインデックス0、要素11はインデックス1、要素13はインデックス3など）。インデックス1のデータを削除し、ハッシュマップ内の要素を再配置します。インデックス1から5の要素をチェックします：

要素11はハッシュ衝突がなければインデックス1に格納されるべきです。インデックス1のデータを削除した後、インデックス1に空きスロットができたため、要素11をそこに移動できます。次に要素3をチェックすると、既にマップされるべきスロットにあります。次に要素21をチェックすると、インデックス1に格納されるべきですが、要素11を前に移動したため、実際の格納スロットまでにギャップが生じています。そのため、要素21も前に移動します。最後に要素13をチェックすると、インデックス3に格納されるべきですが、要素21を移動したためギャップが生じているので、要素13も前に移動します。

これで不変条件が再び成立します：元のスロットと実際にキーと値のペアが格納されているスロットの間に空きスロットがあってはなりません。詳細な実装は演習として残しておきますので、ぜひ挑戦してみてください！

![](/pics/rearrange_2_en.drawio.webp)

## クロージャ

この講義の最後のトピックに移りましょう！クロージャとは何でしょうか？クロージャとは、関数とその周囲の状態への参照を組み合わせたものです。その周囲の状態はレキシカル環境によって決定されます。例えば、以下のコードでは、3行目で関数を定義する際、この`i`は2行目の`i`に対応します。そのため、後で3行目で`println_i`を呼び出すと、2行目の`i`の値が出力されます。次に4行目で`i`を更新すると、出力もそれに応じて更新されます。

しかし、7行目で別の`i`を導入すると、変数名は同じですが、新しい変数`i`はクロージャとは無関係なので、8行目の出力は変更されません。クロージャによって捕捉される環境はプログラムの構造に対応し、コード定義時に決定されますが、実行時には決定されません。

```moonbit
fn init {
  let mut i = 2
  fn println_i() { println(i) } // Capturing i
  i = 3
  println_i() // Output 3
  {
    let i = 4 // A different i variable
    println_i() // Output 3
  }
}
```

### データのカプセル化

クロージャを使用してデータと振る舞いをカプセル化することができます。関数内で定義された変数は、関数のスコープ内でのみ有効であるため、関数の外部からアクセスすることはできません。4行目と5行目に示すように、値を返り値として捕捉する2つの関数を定義することで、ユーザーが値を取得および設定できるようにします。
また、関数内でデータの検証を追加することができます。構造体で直接可変フィールドを定義するとユーザーの操作は無制限になりますが、検証を追加することで不正な入力をフィルタリングできます。
最後に、これら2つの関数を返します。`get()`の結果から、正当な入力は関数を介して捕捉された変数の値を更新し、不正な入力はフィルタリングされることがわかります。

```moonbit
fn natural_number_get_and_set()
  -> ( () -> Int, (Int) -> Unit) { // (get, set)
  let mut i = 0 // Does not have direct access
  fn get() -> Int { i }
  fn set(new_value: Int) -> Unit { if new_value >= 0 { i = new_value } } // Can add data validation
  (get, set)
}

fn init {
  let (get, set) = natural_number_get_and_set()
  set(10)
  println(get()) // 10
  set(-100)
  println(get()) // 10
}
```

クロージャを構造体と共に使用して、ハッシュマップの振る舞いをカプセル化し、抽象データ構造を定義することもできます。以前にオープンアドレッシングとダイレクトアドレッシングの実装を示しましたが、ユーザーにとっては同じ効果を持つため、これは問題になりません。
この場合、4つの関数を持つ構造体`MyMap`を定義できます。これらの関数はすべて同じハッシュマップを捕捉し、変更を許可します。次に、この構造体を構築するための2つの関数を提供し、オープンアドレッシングとダイレクトアドレッシングの両方の実装を提供します。練習として、単純なリストやツリーなどでどのように実装できるか考えてみてください。
最後に、この構造体を使用してみましょう。異なる実装を使用する場合、初期化関数を置き換えるだけで、残りのコードは変更されません。

```moonbit
struct MyMap[K, V] {
  get : (K) -> Option[V]
  put : (K, V) -> Unit
  remove : (K) -> Unit
  size : () -> Int
}
```

```moonbit no-check
// Implementation of open addressing
fn MyMap::hash_open_address[K : Hash + Eq + Default, V : Default]() -> MyMap[K, V] { ... }
// Implementation of direct addressing
fn MyMap::hash_bucket[K : Hash + Eq, V]() -> MyMap[K, V] { ... }
// Implementation with a simple list or tree, etc.

fn init {
  // Replace the initialization function, rest of the code unchanged
  let map : MyMap[Int, Int] = MyMap::hash_bucket()
  // let map : MyMap[Int, Int] = MyMap::hash_open_address()
  (map.put)(1, 1)
  println((map.size)())
}
```

以下は主要なコードスニペットです。`hash_bucket`内に`map`テーブルを実装し、それを複数の関数で捕捉し、これらの関数を構造体に格納して返します。

```moonbit no-check
fn MyMap::hash_bucket[K : Hash + Eq, V]() -> MyMap[K, V] {
  let initial_length = 10
  let load = 0.75
  let map = {
    values: @array.new(5, fn() { { val : None } }) // Aliasing
    size: 0,
    length: initial_length,
  }

  fn resize() { ... }

  fn get(key : K) -> Option[V] { ... }
  fn put(key : K, value : V) -> Unit { ... }
  fn remove(key : K) -> Unit { ... }
  fn size() -> Int { map.size }

  { get, put, remove, size }
}
```

さらに、構造体を基に便利に使用するため、より多くのメソッドを拡張して構築できます。例えば、構造体がキーと値のペアの数を取得する関数を提供する場合、ハッシュマップが空かどうかを追加で判断できます。構造体が値を取得する関数を提供する場合、それを使用してハッシュマップに対応するキーが含まれているかどうかを判断できます。このようにして、異なる実装に一度に同じロジックを追加できます。

```moonbit
fn MyMap::is_empty[K, V](map : MyMap[K, V]) -> Bool {
  (map.size)() == 0
}

fn MyMap::contains[K, V](map : MyMap[K, V], key : K) -> Bool {
  match (map.get)(key) {
    Some(_) => true
    None => false
  }
}
```

```moonbit no-check
fn init {
  let map : MyMap[Int, Int] = MyMap::hash_bucket()
  println(map.is_empty()) // true
  println(map.contains(1)) // false
}
```

## まとめ

ダイレクトアドレッシングとオープンアドレッシングを使用してハッシュマップを実装する2つの方法を紹介しました。同時に、クロージャの概念とそれをカプセル化に使用する方法について説明しました。アルゴリズムをよりよく理解するために、以下の文献を推奨します：

- ***アルゴリズムイントロダクション***: 第11章 - ハッシュテーブル; または
- ***アルゴリズム***: セクション3.4 - ハッシュテーブル