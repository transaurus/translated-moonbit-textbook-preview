# 14. ケーススタディ: スタックマシン

この章では、WebAssemblyをベースにしたシンプルなスタックベースの仮想マシンを実装します。

## コンパイル vs インタプリテーション

仮想マシンの概念を説明する前に、まずコンパイルとインタプリテーションという2つの概念を説明します。ご存知の通り、私たちが日常的に書くコードはテキスト形式ですが、コンピュータが実際に実行できるのはバイナリ命令です。この間にはコンパイルプロセスが存在します。コンパイルとは、コンパイラを使用してソースコードをターゲットプログラムに変換するプロセスであり、様々な入力を与えてプログラムを実行し、期待する出力を得ることができます。C言語など多くの言語はコンパイル型言語です。一方、JavaScriptやPythonなどの言語では、ソースコードを直接コンパイルして実行するのではなく、インタプリタに入力し、コードを読みながら同時にそれに応じた動作を実行させます。このような言語はインタプリタ型言語と呼ばれます。広義には、CPUも一種のインタプリタと言えます。

興味のある学生向けに、話題を少し広げましょう。実は、インタプリタとコンパイラは完全に分離した概念ではありません。双方向のマッピングを通じて、インタプリタをコンパイラに変換することができます。ここで使用される概念は部分計算であり、既知の情報に基づいて計算を特殊化するプログラム最適化技術です。極端な例を挙げると、インタプリタが電卓で、プログラムが算術式である場合、これら2つの情報を使用してプログラムに対応する値を直接計算し、ターゲットプログラムを得ることができます。このターゲットプログラムはコンパイルされたプログラムと等価であり、データを入力するだけで出力プログラムを得ることができます。

## 仮想マシン

コンパイルとインタプリテーションに加えて、もう1つの方法はこれら2つを組み合わせることです。典型的な例はJavaです。Java仮想マシン（JVM）は「一度書けばどこでも実行できる」という目的を達成するために作成されました。プラットフォームに依存しない命令セットと、異なるプラットフォーム用の異なるインタプリタを持っています。Javaプログラムを実行するには、まずソースコードから命令セットにコンパイルし、その後インタプリタを使用して命令を解釈します。

仮想マシンには主に2つのタイプがあります。1つはスタックベースの仮想マシンで、オペランドが後入れ先出し（LIFO）の原則に従ってスタックに格納されます。もう1つはレジスタベースの仮想マシンで、オペランドが通常のコンピュータと同様にレジスタに格納されます。スタックベースの仮想マシンは実装が簡単でコードサイズが小さい一方、レジスタベースの仮想マシンは実際のコンピュータの構成に近く、パフォーマンスが高いです。

`max`関数を例にとると、

- Lua VM（レジスタベース）:

  ```text
  MOVE   2 0 0 ; R(2) = R(0)
  LT     0 0 1 ; R(0) < R(1)?
  JMP    1     ; JUMP -> 5 (4 + 1)
  MOVE   2 1 0 ; R(2) = R(1)
  RETURN 2 2 0 ; return R(2)
  RETURN 0 1 0 ; return
  ```

- WebAssembly VM（スタックベース）:

  ```wasm
  local.get $a local.set $m                     ;; let mut m = a
  local.get $a local.get $b i32.lt_s            ;; if a < b {
  if local.get $b local.set $m end              ;; m = b }
  local.get $m                                  ;; m
  ```

## WebAssembly

ここで、WebAssemblyについて簡単に紹介します。その名が示す通り、これは仮想命令セットであり、当初はウェブ上で使用されていました。しかし、命令セットであるため、仮想マシンが実装されていれば他のプラットフォームでも使用できます。そのため、[Wasmtime](https://github.com/bytecodealliance/wasmtime)、[WAMR](https://github.com/bytecodealliance/wasm-micro-runtime)、[WasmEdge](https://wasmedge.org/)などのランタイムも存在します。また、これはMoonBitの最初のバックエンドでもあります。その主な特徴の1つは、命令セットにも型システムがあり、安全性が保証されていることです。

ここでは、WebAssemblyのサブセットのみを考慮します。データ型は32ビット符号付き整数のみとし、条件文では非ゼロ整数を`true`、ゼロを`false`として表現します。また、以下の非常に限られた命令セットのみを扱います：

- 静的定数の作成: `const`
- 算術演算: `add`, `minus`, `equal`, `modulo`
- 関数呼び出し: `call`
- ローカル変数の値の取得/設定: `local.get`, `local.set`
- 条件文: `if/else`

これらはMoonBitで以下のように表現できます：

```moonbit
enum Value { I32(Int) }

enum Instruction {
  Const(Value)                         // Create a static constant
  Add; Sub; Modulo; Equal              // Arithmetic operations
  Call(String)                         // Function call
  Local_Get(String); Local_Set(String) // Get/set the values of local variables
  If(Int, @immut/list.T[Instruction], @immut/list.T[Instruction]) // Conditional statement
}
```

上記の定義は基本的にWebAssemblyから一対一でコピーしたものです。注目すべきは、`If`が整数と2つの命令リストをパラメータとして取ることです。整数は`if/else`ブロック終了後にスタックに置かれる結果の数を表し、2つの命令リストはそれぞれ条件が`true`と`false`の場合に対応します。

同様に、関数とプログラムの構造も簡単に定義できます：

```moonbit
struct Function {
  name : String
  params : @immut/list.T[String]; result : Int; locals : @immut/list.T[String]
  instructions : @immut/list.T[Instruction]
}

struct Program {
  functions : @immut/list.T[Function]
  start : Option[String]
}
```

関数には名前、パラメータリスト、結果、ローカル変数リスト、および関数本体を表す命令リストがあります。プログラムには複数の関数定義と、エントリポイントとしてのオプションの関数が含まれます。

### 例

それでは、いくつかの例を見てみましょう。

#### 基本的な算術計算

`1 + 2`を例にとると、最初は空のスタックがあります。最初に行うことは、`Const`命令を使用してオペランドを静的定数としてスタックにプッシュすることです。次に、`Add`命令を使用してそれらを加算します。これはスタックのトップから2つのオペランドを消費し、それらの和をスタックのトップに戻します。したがって、操作後、スタックのトップ要素は`3`になります。

```moonbit no-check
@immut/list.of([ Const(I32(1)), Const(I32(2)), Add ])
```

![](/pics/add.drawio.webp)

#### 関数とローカル変数

関数内では、その引数の値を取得する必要があります。次の例は、2つのパラメータを取り、それらの和を結果として返す関数です：

```moonbit no-check
add(a : Int, b : Int) { a + b }
```

`a`と`b`の値を取得してスタックにプッシュするには、`Local_Get`命令を使用する必要があります。その後、前の例で行ったように、`Add`命令を使用して計算を実行できます。

```moonbit no-check
@immut/list.of([ Local_Get("a"), Local_Get("b"), Add ])
```

![](/pics/local.drawio.webp)

ローカル変数の値を設定するには、`Local_Set`命令を使用できます。

#### 関数呼び出し

関数`add`を定義した後、それを呼び出していくつかの計算を実行できます。最初の例で行ったように、まず`1`と`2`をスタックに置きます。次に、`Add`命令を使用する代わりに、`Call`命令を使用して定義した`add`関数を呼び出します。この時、関数パラメータの数に応じて、スタックのトップから対応する数の要素が消費され、順番にローカル変数にバインドされ、関数呼び出しを表す要素がスタックにプッシュされます。これは元のスタック要素と関数自身のデータを分離し、その戻り値の数も記録します。関数呼び出しが終了すると、戻り値の数に応じて、スタックのトップから要素を取り出し、関数呼び出しの要素を削除し、元のトップ要素を戻します。その後、関数が呼び出された場所で計算結果を取得します。

```moonbit no-check
@immut/list.Ts::[ Const(I32(1)), Const(I32(2)), Call("add") ]
```

![](/pics/return.drawio.webp)

#### 条件文

条件文については、先に紹介した通り、32ビット整数を用いて`true`または`false`を表現します。`If`文を実行する際には、スタックの最上位の要素を取り出します。それが非ゼロの場合、`then`ブランチが実行され、ゼロの場合には`else`ブランチが実行されます。注目すべき点は、Wasmにおける各コードブロックにはパラメータ型と戻り値型があり、コードブロックに入る際にスタックの最上位から消費される要素と、コードブロックを抜ける際にスタックの最上位に置かれる要素に対応していることです。例えば、`if/else`ブロックに入る際には入力がないため、ブロック内の計算を行う際にはスタックが空であると仮定します。元々スタック上に何があったとしても、現在のコードブロックとは無関係です。また、整数を1つ返すと宣言しているため、正常に実行を終了する際には、現在の計算環境に整数が1つだけ存在している必要があります。

```moonbit no-check
@immut/list.of([
 Const(I32(1)), Const(I32(0)), Equal,
 If(1, @immut/list.of([Const(I32(1))]), @immut/list.of([Const(I32(0))]))
])
```

![](/pics/if.drawio.webp)

#### 完全なA + Bプログラム

ここまでに紹介した知識を使って、2つの整数の和を計算するプログラムを実装します。`add`関数の定義は既に示しました。ここではさらに、プログラムのメインエントリとして`test_add`関数を追加します。この関数で唯一注意すべき点は、`add`関数を呼び出した後に`print_int`関数を呼び出すことです。これは特別な関数であり、Wasmで入出力をどのように定義するかについては言及しませんでした。これらの関数は外部関数によって実装される必要があり、Wasm自体はサンドボックス内で動作するプログラムと考えることができます。

```moonbit expr
let program = Program::{

  start: Some("test_add"), // Program entry point

  functions: @immut/list.of([
    Function::{
      name: "add", // Addition function
      params: @immut/list.of(["a", "b"]), result: 1, locals: @immut/list.of([]),
      instructions: @immut/list.of([Local_Get("a"), Local_Get("b"), Add]),
    },
    Function::{
      name: "test_add", // calculate add and output
      params: @immut/list.of([]), result: 0, locals: @immut/list.of([]), // no input or output
      // "print_int" is a special function
      instructions: @immut/list.of([Const(I32(0)), Const(I32(1)), Call("add"), Call("print_int")]),
    },
  ]),
}
```

## コンパイラの実装

以下は、私たちが得たいWebAssemblyのターゲットプログラムです。

```wasm
;; Multiple functions
;; Wasm itself only defines operations; interaction depends on external functions
(func $print_int (import "spectest" "print_int") (param i32))

(func $add (export "add") ;; Export function to be directly used by runtime
  (param $a i32) (param $b i32) (result i32 ) ;; (a : Int, b : Int) -> Int
  local.get $a local.get $b i32.add ;;
)

(func $test_add (export "test_add") (result ) ;; Entry function with no input or output
  i32.const 0 i32.const 1 call $add call $print_int
)

(start $test_add)
```

これを以前MoonBitで書いたプログラムと比較すると、対応がほぼ1対1であることがわかります。

次に必要なのはコンパイラを書くことですが、これは簡単なはずです。なぜなら、MoonBitで定義した命令はWebAssemblyの命令とほぼ1対1で対応しているからです。

| Instruction                                 | WebAssembly Instruction                            |
| ------------------------------------------- | -------------------------------------------------- |
| `Const(I32(0))`                             | `i32.const 0`                                      |
| `Add`                                       | `i32.add`                                          |
| `Local_Get("a")`                            | `local.get $a`                                     |
| `Local_Set("a")`                            | `local.set $a`                                     |
| `Call("add")`                               | `call $add`                                        |
| `If(1, @immut/list.of([Const(I32(0))]), @immut/list.of([Const(I32(1))]))` | `if (result i32) i32.const 0 else i32.const 1 end` |

必要なのは単なる文字列変換です。ただし、コンパイラを実装する際には、文字列連結を直接使用するのではなく、組み込みの`Buffer`データ構造を利用すべきです。新しい内容を追加する際に毎回新しいメモリを割り当てる必要がなく、単純な文字列連結と比べてメモリ割り当て操作を減らすことができます。

```moonbit no-check
fn Function::to_wasm(self : Function, buffer : Buffer) -> Unit
fn Program::to_wasm(self : Program, buffer : Buffer) -> Unit
fn Instruction::to_wasm(self : Instruction, buffer : Buffer) -> Unit {
  match self {
    Add => buffer.write_string("i32.add ")
    Local_Get(val) => buffer.write_string("local.get $\{val} ")
    _ => buffer.write_string("...")
  }
}
```

もちろん、WebAssemblyにはテキスト形式（WAT）だけでなく、バイナリ形式もあります。[こちら](https://webassembly.github.io/wabt/demo/wat2wasm/)はWATをバイナリWASMに変換する便利なツールです。興味のある方は、[WebAssembly仕様](https://webassembly.github.io/spec/core/index.html)も参照してください。

| Text Format   | Binary Format                                          |
| ------------- | ------------------------------------------------------ |
| `i32.const`   | 0x41                                                   |
| `i32.add`     | 0x6A                                                   |
| `local.get`   | 0x20                                                   |
| `local.set`   | 0x21                                                   |
| `call $add`   | 0x10                                                   |
| `if else end` | 0x04 (vec[instructions]) 0x05 (vec[instructions]) 0x0B |

### マルチレベルコンパイル

[第11章](./parser)では、構文解析器を紹介しました。当時は基本的な算術式を解析するために使用しましたが、それらは非常に単純で、定数畳み込みによってコンパイル時に完全に処理できました。しかし、プログラムにとって定数畳み込みは明らかに通常のケースではなく、実行のためにバックエンドにコンパイルする必要があります。WebAssemblyを紹介したことで、最後のピースを埋めることができます。文字列から始め、字句解析を行ってトークンストリームを取得します。次に構文解析器を使用して抽象構文木を取得します。この段階から、WebAssembly命令セットにコンパイルします。最後に、さまざまな実行環境に渡して実行できます。もちろん、紹介した「タグレスファイナル」技術のおかげで、抽象構文木が簡略化される可能性もあります。

<center><p>文字列 → トークンストリーム → (抽象構文木) → Wasm IR → コンパイル/実行</p></center>

以下は、[第11章](./parser)から採用した基本的な算術式の構文木の定義です。

```moonbit
enum Expression {
  Number(Int)
  Plus(Expression, Expression)
  Minus(Expression, Expression)
  Multiply(Expression, Expression)
  Divide(Expression, Expression)
}
```

したがって、ASTに対してパターンマッチングを行い、対応するWebAssembly命令列に変換する単純な再帰関数を使用できます。例えば、整数は単一の定数命令に変換され、二項演算では2つのオペランドを再帰的に変換した後、演算自体の命令を追加します。ここではMoonBitの演算子オーバーロード機能を利用していることがわかります。

```moonbit
fn compile_expression(expression : Expression) -> @immut/list.T[Instruction] {
  match expression {
      Number(i) => @immut/list.of([Const(I32(i))])
      Plus(a, b) => compile_expression(a) + compile_expression(b) + @immut/list.of([Add])
      Minus(a, b) => compile_expression(a) + compile_expression(b) + @immut/list.of([Sub])
      _ => @immut/list.of([])
  }
}
```

## インタプリタの実装

次に、解釈実行について見ていきましょう。前のプログラムを直接解釈するインタプリタを構築します。ここでは、オペランドスタックと命令キューという2つのデータ構造が必要です。オペランドスタックには、計算に使用される値に加えて、関数呼び出し前に環境に保存されていた変数も格納されます。命令キューには実行されるべき命令が格納されます。また、元の命令セットを拡張して、`EndOfFrame`命令などの制御命令を追加します。

`EndOfFrame`命令には、関数の戻り値の数を示す整数パラメータがあります。基本データ型として整数しか持たないため、戻り値の数だけを知る必要があります。プログラム環境全体には、プログラム定義に加えて、オペランドスタック、命令キュー、現在の環境のローカル変数が含まれます。

```moonbit
enum StackValue {
  Val(Value) // Ordinary value
  Func(@immut/hashmap.T[String, Value]) // Function stack, stores previous local variables
}
enum AdministrativeInstruction {
  Plain(Instruction) // Ordinary instruction
  EndOfFrame(Int) // Function end instruction
}
struct State {
  program : Program
  stack : @immut/list.T[StackValue]
  locals : @immut/hashmap.T[String, Value]
  instructions : @immut/list.T[AdministrativeInstruction]
}
```

現在必要なのは、現在の命令とデータスタックに対してパターンマッチングを行い、前の状態から次の状態を計算することです。エラーが発生する可能性があるため、返される状態は`Option`でラップされます。ここで示す`Add`命令のようにマッチが成功した場合、スタックの最上部にオペランドを表す2つの連続した整数があるはずで、次の状態を計算できます。すべてのマッチが失敗した場合、何か問題が発生したことを意味し、ワイルドカードを使用してそのようなケースを処理し、`None`を返します。

```moonbit
fn evaluate(state : State, stdout : Buffer) -> Option[State] {
  match (state.instructions, state.stack) {
    (Cons(Plain(Add), tl), Cons(Val(I32(b)), Cons(Val(I32(a)), rest))) =>
      Some(
        State::{ ..state, instructions: tl, stack: Cons(Val(I32(a + b)), rest) },
      )
    _ => None
  }
}
```

![](/pics/interp_add.drawio.webp)

条件文の場合、スタックから対応する分岐のコードを取り出し、命令キューに追加する必要があります。保存された命令は展開されないように注意が必要で、ここではマッピングを行います。

```moonbit no-check
(Cons(Plain(If(_, then, else_)), tl), Cons(Val(I32(i)), rest)) =>
  Some(State::{..state,
      stack: rest,
      instructions: (if i != 0 { then } else { else_ }).map(
        AdministrativeInstruction::Plain,
      ).concat(tl)})
```

![](/pics/interp_if.drawio.webp)

次に関数呼び出しです。前述のように、外部APIなしではWebAssemblyは入出力を実行できません。この問題を解決するため、`print_int`の関数呼び出しを特別に処理します。呼び出しが検出されると、その値を直接キャッシュに出力します。

```moonbit no-check
(Cons(Plain(Call("print_int")), tl), Cons(Val(I32(i)), rest)) => {
  stdout.write_string(i.to_string())
  Some(State::{ ..state, stack: rest, instructions: tl })
}
```

![](/pics/interp_print_int.drawio.webp)

通常の関数呼び出しでは、現在の環境を保存し、呼び出しのための新しい環境に入る必要があります。そのため`EndOfFrame`命令を追加する必要があります。データに関しては、関数パラメータの数に応じて現在のスタックの最上部から一定数の要素を取り出し、新しい関数呼び出し環境とします。その後、現在の環境変数を保存する関数スタックをスタックに追加します。

![](/pics/interp_call.drawio.webp)

実行後、この時点で関数を返す制御命令に遭遇するはずです。命令に保存されている戻り値の数に応じてスタックの最上部から要素を取り出し、以前に保存されていた関数スタックが見つかるまで現在の環境をクリアします。そこから元の環境を復元し、計算を続行します。

![](/pics/interp_end_call.drawio.webp)

## まとめ

この章では以下のことを行いました:

- スタックベース仮想マシンの構造を学びました
- WebAssembly命令セットのサブセットを紹介しました
- コンパイラを実装しました
- インタプリタを実装しました

興味のある読者は、構文解析器での関数定義の拡張や、命令セットへの`return`命令の追加を試みるとよいでしょう。