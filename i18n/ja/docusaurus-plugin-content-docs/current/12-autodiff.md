# 12. ケーススタディ: 自動微分

本日は、複雑な数学的概念を避けつつ、自動微分（autodiff）に関する別のケーススタディについて話します。

微分はコンピュータサイエンスにおいて重要な操作です。機械学習では、勾配降下法に基づくニューラルネットワークが訓練のために局所的最小値を見つける際に微分を適用します。関数を解き、ニュートン法を用いて零点を近似することにもっと馴染みがあるかもしれません。簡単に復習しましょう。ここでは関数をプロットし、初期値を1（数直線上の点A）に設定しています。

![](/pics/geogebra-export-0.webp)

![](/pics/geogebra-export-1.webp)

この点の近くにある零点を近似したいと考えています。この点のx座標に対応する関数上の点Bを計算し、その点における微分係数（その点での接線の傾き）を求めます。

![](/pics/geogebra-export-2.webp)

![](/pics/geogebra-export-3.webp)

接線とx軸の交点を見つけることで、零点に近い値を得ます。

![](/pics/geogebra-export-4.webp)

その後、このプロセスを繰り返し、関数に対応する点を見つけ、微分係数を計算し、接線とx軸の交点を求めます。

![](/pics/geogebra-export-5.webp)

![](/pics/geogebra-export-6.webp)

このようにして、徐々に零点に近づき、近似解を得ることができます。最後にコード実装を提供します。

![](/pics/geogebra-export-7.webp)

今日は、加算と乗算のみを含む次のような単純な関数の組み合わせを見ていきます。例えば、5倍の$x_0$の2乗に$x_1$を加えたものを計算する場合、$x_0$が10で$x_1$が100のとき、関数の値600、$x_0$に関する偏微分係数100、$x_1$に関する偏微分係数1を計算する必要があります。

*例:* $f(x_0, x_1) = 5{x_0}^2 + {x_1}$

- $f(10, 100) = 600$
- $\frac{\partial f}{\partial x_0}(10, 100) = 100$
- $\frac{\partial f}{\partial x_1}(10, 100) = 1$

## 微分法

関数を微分する方法にはいくつかあります。最初の方法は手動微分で、紙とペンを自然な計算機として使用します。欠点は、複雑な式では間違いを犯しやすく、24時間手動で計算し続けることはできないことです。2番目の方法は数値微分です: $\frac{ \texttt{f}(x + \delta x) - \texttt{f}(x) }{ \delta x }$。ここでは、微分したい点に小さな値（ゼロに近い）を加え、差分を計算し、その小さな値で割ります。問題は、コンピュータが小数を正確に表現できないこと、絶対値が大きいほど精度が低くなること、そして無限級数を完全に解くことができないことです。3番目の方法は記号微分で、関数を式木に変換し、その木を操作して微分係数を得ます。例えば、$\textit{Mul(Const(2), Var(1))} \to \textit{Const(2)}$: ここでは定数2にxを掛けたものの微分結果は定数2になります。記号微分の問題点は、計算結果が十分に簡略化されていない可能性があり、冗長な計算が含まれる可能性があることです。さらに、条件分岐やループなどのネイティブな制御フローを直接使用することが難しいことです。現在の値に基づいて大きい方を見つける関数を定義したい場合、単純に値を比較するのではなく、演算子を定義する必要があります。

```moonbit no-check
// Need to define additional native operators for the same effect
fn max[N : Number](x : N, y : N) -> N {
 if x.value() > y.value() { x } else { y }
}
```

最後に、4番目の方法は自動微分です。自動微分は合成関数の微分規則を使用して、基本演算を組み合わせることで計算と微分を実行します。これはモジュール化思考にも合致しています。自動微分は前方微分と後方微分に分かれます。これらを一つずつ紹介していきます。

## 記号微分

まず記号微分を見ていきましょう。enum型を使用して式を定義します。式は定数、ゼロから始まるインデックス付き変数、または2つの関数の和や積になります。ここではシンプルなコンストラクタを定義し、演算子をオーバーロードしてより簡潔な式を生成します。最後に15行目で、パターンマッチングを使用して記号に基づいて関数値を計算するメソッドを定義します（入力はベクトルで、ここでは省略されています）。

```moonbit
enum Symbol {
  Constant(Double)
  Var(Int)
  Add(Symbol, Symbol)
  Mul(Symbol, Symbol)
} derive(Show)

// Define simple constructors and overload operators
fn Symbol::constant(d : Double) -> Symbol { Constant(d) }
fn Symbol::var(i : Int) -> Symbol { Var(i) }
fn Symbol::op_add(f1 : Symbol, f2 : Symbol) -> Symbol { Add(f1, f2) }
fn Symbol::op_mul(f1 : Symbol, f2 : Symbol) -> Symbol { Mul(f1, f2) }

// Compute function values
fn Symbol::compute(self : Symbol, input : Array[Double]) -> Double {
  match self {
    Constant(d) => d
    Var(i) => input[i] // get value following index
    Add(f1, f2) => f1.compute(input) + f2.compute(input)
    Mul(f1, f2) => f1.compute(input) * f2.compute(input)
    }
}
```

任意の定数関数、自分自身に関する偏微分を行う任意の変数、2つの関数の和と積の微分規則を復習しましょう。例えば、$f \times g$の微分は$f$の微分に$g$を掛けたものと$g$の微分に$f$を掛けたものの和です。これらの規則を使用して、パターンマッチングを通じて記号を微分します。偏微分であるため、パラメータにはどの変数に関して微分を行うかを示すインデックスも含まれます。

- $f$が定数関数の場合、$\frac{\partial f}{\partial x_i} = 0$
- $\frac{\partial x_i}{\partial x_i} = 1, \frac{\partial x_j}{\partial x_i} = 0, i \neq j$
- $\frac{\partial (f + g)}{\partial x_i} = \frac{\partial f}{\partial x_i} + \frac{\partial g}{\partial x_i}$
- $\frac{\partial (f \times g)}{\partial x_i} = \frac{\partial f}{\partial x_i} \times g + f \times \frac{\partial g}{\partial x_i}$

前の定義を使用して、例の関数を構築します。見ての通り、乗算と加算の操作は非常に自然に見えます。これはMoonBitがいくつかの演算子をオーバーロードできるためです。

```moonbit
fn differentiate(self : Symbol, val : Int) -> Symbol {
  match self {
      Constant(_) => Constant(0.0)
    Var(i) => if i == val { Constant(1.0) } else { Constant(0.0) }
    Add(f1, f2) => f1.differentiate(val) + f2.differentiate(val)
    Mul(f1, f2) => f1 * f2.differentiate(val) + f1.differentiate(val) * f2
  }
}
```

式を構築した後、それを微分して対応する式を取得します（7行目に示されています）。そして入力に基づいて偏微分を計算します。簡略化を行わない場合、得られる微分式は以下のように非常に複雑になる可能性があります。

```moonbit
fn example() -> Symbol {
  Symbol::constant(5.0) * Symbol::var(0) * Symbol::var(0) + Symbol::var(1)
}

test "Symbolic differentiation" {
  let input : Array[Double] = [10.0, 100.0]
  let symbol : Symbol = example() // Abstract syntax tree of the function
  assert_eq!(symbol.compute(input), 600.0)
  // Expression of df/dx
  inspect!(symbol.differentiate(0),
  content="Add(Add(Mul(Mul(Constant(5.0), Var(0)), Constant(1.0)), Mul(Add(Mul(Constant(5.0), Constant(1.0)), Mul(Constant(0.0), Var(0))), Var(0))), Constant(0.0))")
  assert_eq!(symbol.differentiate(0).compute(input), 100.0)
}
```

もちろん、いくつかの簡略化関数を定義したり、コンストラクタを変更して関数を簡略化したりできます。例えば、加算の結果を簡略化できます。0を任意の数に加えてもその数は変わらないので、その数を保持するだけです。また、2つの数を加算する場合、他の変数と計算する前にそれらを簡略化できます。最後に、右側に整数がある場合、左側に移動して各最適化ルールを2回記述するのを避けられます。

```moonbit
fn Symbol::op_add_simplified(f1 : Symbol, f2 : Symbol) -> Symbol {
  match (f1, f2) {
    (Constant(0.0), a) => a
      (Constant(a), Constant(b)) => Constant(a + b)
      (a, Constant(_) as const) => const + a
      (Mul(n, Var(x1)), Mul(m, Var(x2))) =>
        if x1 == x2 {
          Mul(m + n, Var(x1))
        } else {
          Add(f1, f2)
        }
      _ => Add(f1, f2)
  } }
```

同様に、乗算も簡略化できます。0を任意の数で乗算しても0のまま、1を任意の数で乗算してもその数自体のまま、そして2つの数の乗算を簡略化するなどです。

```moonbit
fn Symbol::op_mul_simplified(f1 : Symbol, f2 : Symbol) -> Symbol {
  match (f1, f2) {
    (Constant(0.0), _) => Constant(0.0) // 0 * a = 0
    (Constant(1.0), a) => a             // 1 * a = 1
    (Constant(a), Constant(b)) => Constant(a * b)
    (a, Constant(_) as const) => const * a
    _ => Mul(f1, f2)
  } }
```

このような簡略化の後、より簡潔な結果が得られます。もちろん、私たちの例は比較的単純です。実際には、同類項の結合など、さらなる簡略化が必要です。

```moonbit
let diff_0_simplified : Symbol = Mul(Constant(5.0), Var(0))
```

## 自動微分

では、自動微分を見ていきましょう。まず、インターフェースを通じて実装したい操作を定義します。これには定数コンストラクタ、加算、乗算が含まれます。また、現在の計算の値を取得することも求めます。

```moonbit
trait Number  {
  constant(Double) -> Self
  op_add(Self, Self) -> Self
  op_mul(Self, Self) -> Self
  value(Self) -> Double // Get the value of the current computation
}
```

このインターフェースを使用すると、言語のネイティブな制御フローを計算に使用でき、動的に計算グラフを生成できます。次の例では、$y$の現在値に基づいて計算する式を選択でき、微分する際には対応する式を微分します。

```moonbit
fn max[N : Number](x : N, y : N) -> N {
  if x.value() > y.value() { x } else { y }
}

fn relu[N : Number](x : N) -> N {
max(x, N::constant(0.0))
}
```

### 前方微分

まず、前方微分から始めます。これは比較的単純で、導関数の規則を直接使用して$f(a)$と$f'(a)$を同時に計算します。導関数だけではなく両方を計算する理由は単純です：2つの関数の積を微分する場合、計算のために両方の関数の現在の値が必要になるため、値と導関数を同時に計算する必要があるからです。数学的には、これは線形代数における`双対数`の概念に対応します。興味があれば、さらに深く掘り下げることをお勧めします。双対数を含む構造体を構築しましょう。1つのフィールドは現在のノードの値、もう1つは現在のノードの導関数です。定数からの構築は非常に簡単です：値は定数そのもので、導関数はゼロです。現在の値を取得するのも非常に直感的で、対応する変数にアクセスするだけです。ここではヘルパー関数を追加します。変数の場合、その値に加えて、微分対象の変数かどうかを判定する必要があります。微分対象であれば導関数は1、そうでなければ0です。これは前述の通りです。

```moonbit
struct Forward {
  value : Double      // Current node value f
  derivative : Double // Current node derivative f'
} derive(Show)

fn Forward::constant(d : Double) -> Forward { { value: d, derivative: 0.0 } }
fn Forward::value(f : Forward) -> Double { f.value }

// determine if to differentiate the current variable
fn Forward::var(d : Double, diff : Bool) -> Forward {
  { value : d, derivative : if diff { 1.0 } else { 0.0 } }
}
```

次に、加算と乗算のメソッドを定義します。導関数の規則を使用して導関数を直接計算します。例えば、2つの関数$f$と$g$の和の値はそれらの値の和であり、導関数はそれらの導関数の和です（4行目に示されています）。2つの関数$f$と$g$の積の場合、値はそれらの値の積であり、導関数は前に紹介した通り：$f \times g' + g \times f'$です。このようにして、中間データ構造を作成することなく、直接導関数を計算します。

```moonbit
fn Forward::op_add(f : Forward, g : Forward) -> Forward { {
  value : f.value + g.value,
  derivative : f.derivative + g.derivative // f' + g'
} }

fn Forward::op_mul(f : Forward, g : Forward) -> Forward { {
  value : f.value * g.value,
  derivative : f.value * g.derivative + g.value * f.derivative // f * g' + g * f'
} }
```

最後に、前に定義した条件付きの例を使用して導関数を計算します。前方微分は一度に1つの入力パラメータに関する導関数しか計算できないことに注意してください。これは、入力パラメータよりも出力パラメータが多い場合に適しています。しかし、ニューラルネットワークでは通常、多数の入力パラメータと1つの出力があります。そのため、次に紹介する後方微分を使用する必要があります。

```moonbit
test "Forward differentiation" {
// Forward differentiation with abstraction
  inspect!(relu(Forward::var(10.0, true)), content="{value: 10.0, derivative: 1.0}")
  inspect!(relu(Forward::var(-10.0, true)), content="{value: 0.0, derivative: 0.0}")
// f(x, y) = x * y => df/dy(10, 100)
  inspect!(Forward::var(10.0, false) * Forward::var(100.0, true), ~content="{value: 1000.0, derivative: 10.0}")
}
```

### 後方微分

後方微分は計算に連鎖律を利用します。$x$、$y$、$z$などの関数である$w$があり、$x$、$y$、$z$などが$t$の関数であるとします。このとき、$w$の$t$に関する偏微分は、$w$の$x$に関する偏微分と$x$の$t$に関する偏微分の積、$w$の$y$に関する偏微分と$y$の$t$に関する偏微分の積、$w$の$z$に関する偏微分と$z$の$t$に関する偏微分の積、などを足し合わせたものになります。

- $w = f(x, y, z, \cdots), x = x(t), y = y(t), z = z(t), \cdots$ が与えられたとき
  $\frac{\partial w}{\partial t} = \frac{\partial w}{\partial x} \frac{\partial x}{\partial t} + \frac{\partial w}{\partial y} \frac{\partial y}{\partial t} + \frac{\partial w}{\partial z} \frac{\partial z}{\partial t} + \cdots$

例えば、$f(x_0, x_1) = x_0 ^ 2 \times x_1$ の場合、$f$ を $g$ と $h$ の関数として考えることができます。ここで、$g$ と $h$ はそれぞれ $x_0 ^ 2$ と $x_1$ です。各成分を微分します：$f$ の $g$ に関する偏微分は $h$ です；$f$ の $h$ に関する偏微分は $g$ です；$g$ の $x_0$ に関する偏微分は $2x_0$ で、$h$ の $x_0$ に関する偏微分は 0 です。最後に、連鎖律を使ってこれらを組み合わせると、結果 $2x_0x_1$ が得られます。逆方向微分は、$f$ の $f$ に関する偏微分から始まり、中間関数に対する $f$ の偏微分とそれらの中間関数に対する偏微分を計算していくプロセスです。入力パラメータに関する偏微分に到達するまで、このように逆方向にトレースし、$f$ の計算グラフを逆順に作成することで、各入力ノードの微分を計算できます。これは、出力パラメータよりも入力パラメータが多い場合に適しています。

- 例: $f(x_0, x_1) = {x_0} ^ 2 x_1$
  - 分解: $f = g h, g(x_0, x_1) = {x_0} ^ 2, h(x_0, x_1) = x_1$
  - 微分: $\frac{\partial f}{\partial g} = h = x_1, \frac{\partial g}{\partial x_0} = 2x_0, \frac{\partial f}{\partial h} = g = {x_0}^2, \frac{\partial h}{\partial x_0} = 0$
  - 結合: $\frac{\partial f}{\partial x_0} = \frac{\partial f}{\partial g} \frac{\partial g}{\partial x_0} + \frac{\partial f}{\partial h} \frac{\partial h}{\partial x_0} = x_1 \times 2x_0 + {x_0}^2 \times 0 = 2 x_0 x_1$

ここでは、MoonBitでの実装例を示します。逆方向微分ノードは、現在のノードの値と `backward` という名前の関数で構成されます。`backward` 関数は、結果から現在のノードまでの累積微分（パラメータ）を使用して、現在のノードを構築するすべてのパラメータの微分を更新します。例えば、以下では入力を表すノードを定義します。すべての経路で計算された微分を累積するために `Ref` を使用します。逆方向計算プロセスが終端に達すると、関数の現在の変数に関する偏微分をアキュムレータに加算します。この偏微分は、計算グラフ内の1つの経路の偏微分にすぎません。定数の場合、入力パラメータがないため、`backward` 関数は何も行いません。

```moonbit
struct Backward {
  value : Double              // Current node value
  backward : (Double) -> Unit // Update the partial derivative of the current path
} derive(Show)

fn Backward::var(value : Double, diff : Ref[Double]) -> Backward {
  // Update the partial derivative along a computation path df / dvi * dvi / dx
  { value, backward: fn { d => diff.val = diff.val + d } }
}

fn Backward::constant(d : Double) -> Backward {
  { value: d, backward: fn { _ => () } }
}

fn Backward::backward(b : Backward, d : Double) -> Unit { (b.backward)(d) }

fn Backward::value(backward : Backward) -> Double { backward.value }
```

次に、加算と乗算について見ていきましょう。関数$g$と$h$が計算に関与し、現在の関数が$f$、最終結果が$y$で、$x$がパラメータであると仮定します。これまでに$f$の$g$と$h$に関する偏微分について説明しましたが、ここでは省略します。$y$の$x$に関する累積偏微分について、$f$と$g$の経路を通る偏微分は、$y$の$f$に関する偏微分×$f$の$g$に関する偏微分×$g$の$x$に関する偏微分となります。ここで、$y$の$f$に関する偏微分は、`backward`関数のパラメータ$\textit{diff}$に対応します。したがって、4行目で$g$に渡すパラメータが$\textit{diff} \times 1.0$であることがわかります。これは$y$の$f$に関する偏微分×$f$の$g$に関する偏微分に対応します。同様のパラメータを$h$にも渡します。11行目では、微分規則に従い、$g$に渡すパラメータは$\textit{diff}$×$h$の現在値、$h$に渡すパラメータは$\textit{diff}$×$g$の現在値となります。

```moonbit
fn Backward::op_add(g : Backward, h : Backward) -> Backward {
  {
    value: g.value + h.value,
    backward: fn(diff) { g.backward(diff * 1.0); h.backward(diff * 1.0) },
  }
}

fn Backward::op_mul(g : Backward, h : Backward) -> Backward {
  {
    value: g.value * h.value,
    backward: fn(diff) { g.backward(diff * h.value); h.backward(diff * g.value) },
  }
}
```

最後に、その使用方法を見ていきます。$x$と$y$の微分を保存するための2つの`Ref`を構築します。入力値がそれぞれ10と100である2つの入力ノードを作成するために、2つのアキュムレータを使用します。その後、前の例を使用して計算を行い、順方向計算が完了した後、backward関数を呼び出します。パラメータ`1.0`は、$f$の$f$に関する微分に対応します。この時点で、両方の`Ref`の値が更新され、すべての入力パラメータの微分を同時に取得できます。

```moonbit
test "Backward differentiation" {
  let diff_x = Ref::{ val: 0.0 } // Store the derivative of x
  let diff_y = Ref::{ val: 0.0 } // Store the derivative of y
  let x = Backward::var(10.0, diff_x)
  let y = Backward::var(100.0, diff_y)
  (x * y).backward(1.0) // df / df = 1
  inspect!(diff_x, content="{val: 100.0}")
  inspect!(diff_y, content="{val: 10.0}")
}
```

これで、逆方向微分を使用してニューラルネットワークを記述することができます。この講義では、自動微分とニュートン法を使用して零点を近似する方法のみをデモンストレーションします。最初に見た関数をインターフェースを使用して定義しましょう。

次に、ニュートン法を使用して値を求めます。パラメータが1つしかないため、順方向微分を使用します。

- $f = x^3 - 10 x^2 + x + 1$

  ```moonbit
  fn example_newton[N : Number](x : N) -> N {
    x * x * x + N::constant(-10.0) * x * x + x + N::constant(1.0)
  }
  ```

ニュートン法で零点を近似するには:

- まず、反復変数$x$を初期値1.0で定義します。$x$は微分対象の変数であるため、2番目のパラメータをtrueに設定します。
- 次に、無限ループを定義します。
- 3番目に、5行目で$x$に対応する関数の値と微分を計算します。
- 4番目に、6行目で、値÷微分（つまり近似したいステップサイズ）が十分に小さい場合、零点に非常に近いことを示し、ループを終了します。
- 最後に、7行目で条件が満たされない場合、$x$の値を前の値から値÷微分を引いた値に更新し、ループを続けます。

このようにして、ループを反復することで最終的に近似解を得ます。

- $x_{n+1} = x_n - \frac{f(x_n)}{f'(x_n)}$

```moonbit
test "Newton's method" {
  (loop Forward::var(1.0, true) { // initial value
    x => {
      let { value, derivative } = example_newton(x)
      if (value / derivative).abs() < 1.0e-9 {
        break x.value // end the loop and have x.value as the value of the loop body
      }
      continue Forward::var(x.value - value / derivative, true)
    }
  } |> assert_eq!(0.37851665401644224))
}
```

## まとめ

まとめとして、この講義では自動微分の概念を紹介しました。記号微分と2種類の自動微分の実装について説明しました。さらに学びたい学生向けには、*3Blue1Brown*の深層学習シリーズ（[勾配降下法](https://www.youtube.com/watch?v=IHZwWFHWa-w)、[誤差逆伝播法](https://www.youtube.com/watch?v=Ilg3gGewQ5U)などのトピックを含む）を視聴し、独自のニューラルネットワークを実装してみることをお勧めします。