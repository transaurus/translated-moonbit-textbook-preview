# 11. ケーススタディ: パーサー

MoonBitプログラミング言語の基本的な理解を得た今、より複雑なプログラムを探求し、いくつかの興味深いケースを紹介します。この講義では、パーサーを紹介します。

世界にはさまざまな種類の言語が存在し、プログラミング言語やその他の記号言語が含まれます。四則演算を例にとりましょう。`"(1+ 5) * 7 / 2"`のような文字列の場合、最初のステップはそれをトークンのリストに分割することです。例えば、左括弧、整数1、プラス記号、整数5、右括弧、乗算記号、整数7、除算記号、整数2のようにトークン化できます。整数1と括弧、プラス記号の間にスペースがなくても、字句規則に従ってこれらは3つのトークンに分割されるべきです。このステップは字句解析として知られています。

トークンのストリームを得たら、構文/文法に基づいて抽象構文木（AST）に変換します。例えば、整数1と5の和を最初に求め、その和を7で乗算し、最後にその積を2で除算するべきです。5と7の積に1を加えるような解釈は構文規則に従いません。このステップは構文解析として知られています。

最後に、意味論に従って最終結果を計算します。例えば、`1 + 5`は整数1と5の和を求めることを意味します。

構文解析/パーシングはコンピュータサイエンスの重要な分野です。すべてのプログラミング言語はソースコードを解析して実行するためにパーシングを必要とするため、多くの成熟したツールが存在します。この講義では、字句解析と構文解析の両方を扱うパーサーコンビネーターを紹介します。興味があれば、最後に紹介する推奨読書を参照してさらに深く学ぶこともできます。すべてのコードはコースウェブサイトで利用可能です。それでは始めましょう！

## 字句解析

まず、字句解析について話しましょう。その目的は入力をトークンに分割することです。入力は文字列で、出力はトークンストリームです。例えば、"12 +678"は整数12、プラス記号、整数678に分割されるべきです。字句解析は一般的に単純で、有限状態機械を使用して行うことができます。lexやflexのようなドメイン固有言語はプログラムを自動生成できます。ここでは、パーサーコンビネーターを使用します。まず、整数、括弧、演算子、空白を含む算術式の字句規則を定義します。

```abnf
Number     = %x30 / (%x31-39) *(%x30-39)
LParen     = "("
RParen     = ")"
Plus       = "+"
Minus      = "-"
Multiply   = "*"
Divide     = "/"
Whitespace = " "
```

整数とプラス記号を例にとりましょう。字句規則の各行はパターンマッチング規則に対応します。引用符内のコンテンツは同じ内容の文字列にマッチすることを意味します。規則`a b`は、最初に規則`a`をマッチし、成功した場合に規則`b`をマッチし続けることを意味します。規則`a / b`は、規則`a`または`b`をマッチすることを意味し、最初に`a`を試し、失敗した場合に`b`を試します。アスタリスクが前に付いた規則`*a`は、ゼロ回以上のマッチを指します。最後に、`%x`はUTFエンコードされた文字にマッチすることを意味し、`x`は16進数であることを示します。例えば、`0x30`は48番目の文字`0`に対応し、16進数では`30`です。この理解をもとに、定義規則を確認しましょう。プラスは単純にプラス記号を表します。数字は、1-9の文字がゼロ回以上続き、その後0-9の文字がゼロ回以上続くことを表します。

![](/pics/lex_rail.drawio.webp)

MoonBitでは、トークンを列挙型として定義し、トークンは整数を含む値や演算子、括弧を含むことができます。空白は単純に破棄されます。

```moonbit
enum Token {
  Value(Int); LParen; RParen; Plus; Minus; Multiply; Divide
} derive(Show)
```

### パーサーコンビネーター

次に、組み合わせ可能なパーサーを構築します。このパーサーは文字列を入力として受け取り、null許容型の値`Option[T]`を出力する関数です。`Option[T]`が空の値の場合、パターンマッチングが失敗したことを示し、非空の値は結果と残りの文字列を含みます。理想的には、パースが失敗した場合に、なぜどこで失敗したかといったエラーメッセージを提供すべきですが、簡略化のためここでは省略しています。`Result[A, B]`を使用して実装することも自由です。また、利便性のために`parse`メソッドも提供しています。

```moonbit
// V represents the value obtained after parsing succeeds
// Lexer[V] == (String) -> Option[(V, String)]
type Lexer[V] (String) -> Option[(V, String)]

fn parse[V](self : Lexer[V], str : String) -> Option[(V, String)] {
  (self.0)(str)
}
```

まず、最も単純なパーサーを定義します。これは単一の文字にマッチするだけのものです。このパーサーを構築するには、入力文字が条件を満たすかどうかをチェックする関数が必要です。3行目では、入力が空でなく、最初の文字が条件を満たす場合、その文字を値として読み取り、残りの文字列とともに返します。それ以外の場合は、マッチング失敗を示す空の値を返します。このパーサーは以下のコードで使用します。例えば、文字が`a`かどうかを判定する無名関数を定義し、それを使って文字列`"asdf"`をパースします。"asdf"は`a`で始まるため、パースは成功し、結果として`a`と残りの文字列`"sdf"`が得られます。同じ関数で残りの文字列を再度マッチングさせると、失敗します。

```moonbit
fn pchar(predicate : (Char) -> Bool) -> Lexer[Char] {
  Lexer(fn(input) {
    if input.length() > 0 && predicate(input[0]) {
      Some((input[0], input.substring(start=1)))
    } else {
      None
} },) }

test {
  inspect!(pchar(fn { ch => ch == 'a' }).parse("asdf"), content="Some((\'a\', \"sdf\"))")
  inspect!(pchar(fn {
    'a' => true
    _ => false
  },).parse("sdf"),content="None",)
}
```

この単純なパーサーを使えば、括弧、算術演算子、空白など、ほとんどのトークンを処理できます。ここでは、無名関数を使用してこれらを定義し、直接すべての可能性をパターンマッチングしようとします。マッチさせたい文字であれば`true`を返し、そうでなければ`false`を返します。空白についても同様です。しかし、単に入力文字を文字としてパースするだけでは不十分です。より具体的な列挙型の値を取得したいため、マッピング関数を定義する必要があります。

```moonbit expr
let symbol: Lexer[Char] = pchar(fn{
  '+' | '-' | '*' | '/' | '(' | ')' => true
  _ => false
})
```

```moonbit
let whitespace : Lexer[Char] = pchar(fn{ ch => ch == ' ' })
```

`map`関数は、パースが成功した際に結果を変換できます。そのパラメータにはパーサー自体と変換関数が含まれます。値を取得した後、変換関数を適用します。これを使用して、算術演算子や括弧に対応する文字を、対応する列挙型の値にマッピングできます。

```moonbit
fn map[I, O](self : Lexer[I], f : (I) -> O) -> Lexer[O] {
  Lexer(fn(input) {
    // Non-empty value v is in Some(v), empty value None is directly returned
    let (value, rest) = match self.parse(input) {
      Some(v) => v
      None => return None
    }
    Some((f(value), rest))
},) }

let symbol : Lexer[Token] = pchar(
  fn {
    '+' | '-' | '*' | '/' | '(' | ')' => true
    _ => false
  },).map(
  fn {
    '+' => Token::Plus
    '-' => Token::Minus
    '*' => Token::Multiply
    '/' => Token::Divide
    '(' => Token::LParen
    ')' => Token::RParen
},)
```

他のコンビネーターを見てみましょう。`a`の後に`b`をマッチさせる、`a`または`b`をマッチさせる、`a`の0回以上の繰り返しをマッチさせるなど、他のパターンマッチングルールがあります。各コンビネーターは実装が簡単で、一つずつ進めていきましょう。`a`の後に`b`をマッチさせる場合、まず3行目で示すように`self`を使ってパースします。値と残りの文字列を取得した後、7行目で示すように別のパーサーを使って残りの文字列をパースします。2つの出力はタプルとして返されます。次に、`a`または`b`をマッチさせる場合、`self`を使ってパースした結果をパターンマッチします。空の場合は別のパーサーの結果を使用し、それ以外の場合は現在の結果を返します。

```moonbit
fn and[V1, V2](self : Lexer[V1], parser2 : Lexer[V2]) -> Lexer[(V1, V2)] {
  Lexer(fn(input) {
    let (value, rest) = match self.parse(input) {
      Some(v) => v
      None => return None
    }
    let (value2, rest2) = match parser2.parse(rest) {
      Some(v) => v
      None => return None
    }
    Some(((value, value2), rest2))
},) }

fn or[Value](self : Lexer[Value], parser2 : Lexer[Value]) -> Lexer[Value] {
  Lexer(fn(input) {
    match self.parse(input) {
      None => parser2.parse(input)
      Some(_) as result => result
} },) }
```

0回以上の繰り返しをマッチさせる場合、5行目から10行目に示すようにループを使用します。6行目で残りの入力をパースしようとします。失敗した場合はループを抜け、成功した場合はパースされた内容をリストに追加し、残りの入力を更新します。最終的にはパースは常に成功するため、結果を`Some`に格納します。値はリストに格納されており、リストはスタックであるため、正しい順序を得るには逆順にする必要があることに注意してください。

```moonbit
fn many[Value](self : Lexer[Value]) -> Lexer[@immut/list.T[Value]] {
  Lexer(fn(input) {
   loop input, @immut/list.T::Nil {
      rest, cumul => match self.parse(rest) {
        None => Some((cumul.rev(), rest))
        Some((value, rest)) => continue rest, Cons(value, cumul)
      }
    }
},) }
```

最後に、整数の字句解析器を構築します。整数はゼロ、または非ゼロの数字で始まりその後に任意の数の数字が続くものです。まず3つのヘルパー・パーサーを構築します。最初のパーサーは文字`0`にマッチし、それを数値のゼロにマップします。次の2つのパーサーはそれぞれ`1-9`と`0-9`にマッチします。ここではUTFエンコーディングの範囲を使用して判定し、UTFの数字は0から9まで順序付けられているため、文字のエンコーディングと`0`のエンコーディングの差を計算して対応する数値を取得します。最後に、構文規則に従ってコンビネーターを使用してパーサーを構築します。11行目と12行目に示すように、規則を正確に反映しています。ただし、非ゼロ数字と任意の数の数字は単に数字と数字のリストのタプルを形成するため、さらに1つのマッピングステップが必要です。`fold_left`を使用して整数に畳み込みます。リストの先頭に近い数字が左側の桁になるため、数字を10倍して右側の数字を加算することで最終的な整数を形成し、それをenumにマップします。

```moonbit
// Convert characters to integers via encoding
let zero: Lexer[Int] =
  pchar(fn { ch => ch == '0' }).map(fn { _ => 0 })
let one_to_nine: Lexer[Int] =
  pchar(fn { ch => ch.to_int() >= 0x31 && ch.to_int() <= 0x39 },).map(fn { ch => ch.to_int() - 0x30 })
let zero_to_nine: Lexer[Int] =
  pchar(fn { ch => ch.to_int() >= 0x30 && ch.to_int() <= 0x39 },).map(fn { ch => ch.to_int() - 0x30 })

// number = %x30 / (%x31-39) *(%x30-39)
let value : Lexer[Token] = zero.or(
  one_to_nine.and(zero_to_nine.many()).map( // (Int, @immut/list.T[Int])
    fn { (i, ls) => ls.fold_left(fn { i, j => i * 10 + j }, init=i) },
  ),
).map(Token::Value)
```

字句解析を完了するまであと一歩です：入力ストリーム全体を解析します。トークンの間に空白が存在する可能性があるため、2行目で数字や記号を定義した後に任意の長さの空白を許可します。空白を表すタプルの2番目の値をマップして破棄し、パーサー全体を任意の回数繰り返すことができます。最終的に、文字列をマイナス記号、数字、プラス記号、括弧などに分割できます。ただし、この出力ストリームは算術式の構文規則に従っていません。このため、構文解析が必要になります。

```moonbit
let tokens : Lexer[@immut/list.T[Token]] =
  value.or(symbol).and(whitespace.many())
    .map(fn { (symbols, _) => symbols },) // Ignore whitespaces
    .many()

test{
  inspect!(tokens.parse("-10123+-+523 103    ( 5) )  "), content="Some((@list.of([Minus, Value(10123), Plus, Minus, Plus, Value(523), Value(103), LParen, Value(5), RParen, RParen]), \"\"))")
}
```

## 構文解析

前の例では、文字列をトークンのストリームに変換し、重要でない空白を破棄して、文字列を意味のあるenumに分割しました。ここでは、トークンストリームが算術式として構文的に有効かどうかを分析します。簡単な例として、式内の括弧は対になっており、正しい順序で閉じられる必要があります。以下のコードスニペットで簡単な構文規則を定義しました。算術式は、単一の数字、演算を実行する2つの算術式、または括弧で囲まれた式のいずれかです。トークンストリームを以下のような抽象構文木に変換することを目指しています。式`1 + (1 - 5)`の場合、ルートノードはプラス記号で、最後に実行される演算を表します。これは1を右側の式に加算することを意味します。右側のサブツリーにはマイナス記号と整数1および5が含まれ、1から5を引くことを意味します。括弧はそれがより早く実行されることを意味するため、式ツリーのより深い位置にあります。同様に、式`(1 - 5) * 5`の場合、最初に実行される計算は括弧内の減算で、その後に乗算が実行されます。

```abnf
expression = Value / "(" expression ")"
expression =/ expression "+" expression / expression "-" expression
expression =/ expression "*" expression / expression "/" expression
```

![](/pics/ast-example.drawio.webp)

ただし、この構文規則にはいくつかの問題があります。優先順位を区別していないためです。例えば、`a + b * c`は`a`に`b`と`c`の積を加えたものと解釈されるべきですが、現在の構文規則では`a`と`b`の和に`c`を掛けたものも有効であり、曖昧さが生じます。また、結合性も示していません。算術演算子は左結合であるべきで、`a + b + c`は`a`に`b`を加え、その後`c`を加えると解釈されるべきです。しかし、現在の構文では`a`に`b`と`c`の和を加えることも許可されています。したがって、階層化のために構文規則を調整する必要があります。

修正された構文規則は3つの部分に分割されます。最初の部分は`atomic`で、整数か括弧内の式のいずれかです。2番目の部分は`atomic`、または`combine`と`atomic`の乗算・除算です。3番目の部分は`combine`、または`expression`と`combine`の加算・減算です。これら3つの規則に分割する目的は演算子の優先順位を区別するためで、単一規則内の左再帰は左結合性を反映するためのものです。しかし、私たちのパーサーは左再帰を処理できません。左再帰規則をパターンマッチングしようとすると、演算子の側で同じ規則を最初にマッチしようとするため、再帰が発生して処理が進まなくなります。これはあくまで私たちのパーサーの制限です。ボトムアップパーサーは左再帰を処理できますので、興味があれば推薦図書を参照してください。

```abnf
atomic     = Value / "(" expression ")"
combine    = atomic  /    combine "*" atomic  /    combine "/" atomic
expression = combine / expression "+" combine / expression "-" combine
```

修正された構文規則は再帰を排除していますが、マッピングのための追加処理が必要です。簡単に言えば、抽象構文木を定義し、式は前述の通り整数か算術演算式の結果のいずれかになります。

```abnf
atomic     = Value / "(" expression ")"
combine    = atomic  *( ("*" / "/") atomic)
expression = combine *( ("+" / "-") combine)
```

```moonbit
enum Expression {
  Number(Int)
  Plus(Expression, Expression)
  Minus(Expression, Expression)
  Multiply(Expression, Expression)
  Divide(Expression, Expression)
}
```

### 構文解析

以前の定義と同様に構文パーサーを定義しましょう。ただし、入力は文字列ではなくトークンのリストになります。ほとんどのコンビネータは以前のものと同様で、同じように実装できます。課題は相互再帰的な構文パーサーをどう定義するかです。`atomic`は`expression`を参照し、`expression`は`combine`に依存し、`combine`は再び`atomic`に依存するからです。この問題を解決するために、2つの解決策を提供します：定義の遅延または再帰関数です。

```moonbit
type Parser[V] (@immut/list.T[Token]) -> Option[(V, @immut/list.T[Token])]

fn parse[V](self : Parser[V], tokens : @immut/list.T[Token]) -> Option[(V, @immut/list.T[Token])] {
  (self.0)(tokens)
}
```

### 再帰的定義

定義の遅延は、まずデフォルト値で参照を定義することを含みます。依存するコンビネータを定義した後、参照値を更新して、計算時に最新の値を取得できるようにします。ここでは、参照を通常のパーサーとして扱い、計算時にのみ現在の値を取得する単純な`ref`関数を定義します。

```moonbit
fn Parser::ref[Value](ref: Ref[Parser[Value]]) -> Parser[Value] {
  Parser(fn(input) {
    ref.val.parse(input)
  })
}
```

その後、遅延定義を使用して式を解析します。`expression`規則への参照を構築し、どのような状況でも解析に失敗するデフォルト値を設定します。次に、`ref`を含む以前に定義した一連のコンビネータを使用し、構文規則に従って最初の規則`atomic`のパーサーを定義します。左右の括弧のパーサー実装は比較的単純で、ここでは詳細を示しません。次に、同様に2番目の規則のパーサーを定義します。2番目の規則を定義した後、初期参照の値を実際のパーサーで更新します。最後に、実際のパーサーの内容を返します。内容は更新されているので、保存された値に直接アクセスできます。

```moonbit no-check
fn parser() -> Parser[Expression] {
  // First define an empty reference
  let expression_ref : Ref[Parser[Expression]] = { val : Parser(fn{ _ => None }) }

  // atomic = Value / "(" expression ")"
  let atomic =  // Use the reference for the definition
    (lparen.and(ref(expression_ref)).and(rparen).map(fn { ((_, expr), _) => expr}))
      .or(number)

  // combine = atomic *( ("*" / "/") atomic)
  let combine = atomic.and(multiply.or(divide).and(atomic).many()).map(fn {
    ...
  })

  // expression = combine *( ("+" / "-") combine)
  expression_ref.val = combine.and(plus.or(minus).and(combine).many()).map(fn {
    ...
  })
  ref(expression_ref)
}
```

再帰関数の概念も同様です。私たちのパーサーは本質的に、区別しやすくメソッドを追加しやすいように別の型でラップされた関数です。したがって、4行目、10行目、11行目に示すように、3つの相互再帰関数を定義できます。次に、6行目で関数に対応するパーサーを構築します。最後に、関数に対応するパーサーを返します。これでパーサーが完成しました。このパーサーを使用して、以前に解析したトークンストリームを対応する抽象構文木に解析できます。結果を計算するためにパターンマッチングできますが、式を評価するための他のオプションもあります。

```moonbit no-check
fn recursive_parser() -> Parser[Expression] {
  // Define mutually recursive functions
  // atomic = Value / "(" expression ")"
  fn atomic(tokens: @immut/list.T[Token]) -> Option[(Expression, @immut/list.T[Token])] {
    lparen.and(
      Parser(expression) // Reference function
    ).and(rparen).map(fn { ((_, expr), _) => expr})
      .or(number).parse(tokens)
  }
  fn combine(tokens: @immut/list.T[Token]) -> Option[(Expression, @immut/list.T[Token])] { ... }
  fn expression(tokens: @immut/list.T[Token]) -> Option[(Expression, @immut/list.T[Token])] { ... }

  // Return the parser represented by the function
  Parser(expression)
}
```

### 構文木を超えて：タグレスファイナル

以前のアプローチは抽象構文木を生成してから解析するものでした。これは列挙型を使用したため、ラベル付けのようなものです。ここでは、タグレスファイナルを導入し、抽象構文木を構築せずに式を評価する別のアプローチを紹介します。これはMoonBitのインターフェースのおかげで可能です。`trait`を通じて動作を抽象化します。式は整数から構築でき、2つの式間で算術演算が可能です。これに基づいてインターフェースを定義しました。インターフェースには実装が伴い、インターフェースの異なる実装は動作や意味論の異なる解釈に対応します。

```moonbit
trait Expr {
  number(Int) -> Self
  op_add(Self, Self) -> Self
  op_sub(Self, Self) -> Self
  op_mul(Self, Self) -> Self
  op_div(Self, Self) -> Self
}
```

パーサーの動作を変更するために、振る舞いの抽象化を使用します。整数の場合、enum型の整数を構築する代わりに、インターフェースの`number`メソッドを使用します。算術演算では、マッピング関数内でenumコンストラクタの代わりにインターフェースで提供される演算子を使用します。最後に、字句解析と構文解析を組み合わせて、文字列を最終結果に処理するパーサーを生成します。解析タイプを指定しないことに注意してください。式と互換性のある任意のインターフェースが機能します。

```moonbit no-check
fn recursive_parser[E : Expr]() -> Parser[E] {
  let number : Parser[E] = ptoken(fn { Value(_) => true; _ => false})
    .map(fn { Value(i) => E::number(i) }) // Use the abstract behavior

  fn atomic(tokens: @immut/list.T[Token]) -> Option[(E, @immut/list.T[Token])] { ... }
  // Convert to a * b * c * ... and a / b / c / ...
  fn combine(tokens: @immut/list.T[Token]) -> Option[(E, @immut/list.T[Token])] { ... }
  // Convert to a + b + c + ... and a - b - c - ...
  fn expression(tokens: @immut/list.T[Token]) -> Option[(E, @immut/list.T[Token])] { ... }

  Parser(expression)
}
// Put things together
fn parse_string[E : Expr](str: String) -> Option[(E, String, @immut/list.T[Token])] {
  let (token_list, rest_string) = match tokens.parse(str) {
    Some(v) => v
    None => return None
  }
  let (expr, rest_token) : (E, @immut/list.T[Token]) = match recursive_parser().parse(token_list) {
    Some(v) => v
    None => return None
  }
  Some(expr, rest_string, rest_token)
}
```

したがって、異なる実装を定義し、MoonBitでどれを使用するかを指定するだけで済みます。前者は、データ構造に対してインターフェースの要件を満たすために異なるメソッドを定義することを含みます（4行目と5行目の`number`メソッドなど）。後者は、特定の型パラメータを示すために関数の戻り値の型を指定します（8行目と12行目に示されています）。8行目では、enumから構築された式ツリーを取得し、12行目では直接結果を取得できます。また、余分な括弧や空白を削除して式をフォーマットされた文字列に変換するなど、他の解釈を追加することもできます。

```moonbit no-check
enum Expression { ... } derive(Show) // Implementation of syntax tree
type BoxedInt Int derive(Show) // Implementation of integer
// Other interface implementation methods omitted
fn BoxedInt::number(i: Int) -> BoxedInt { BoxedInt(i) }
fn Expression::number(i: Int) -> Expression { Number(i) }
// Parse
test {
  inspect!((parse_string("1 + 1 * (307 + 7) + 5 - 3 - 2") :
    Option[(Expression, String, @immut/list.T[Token])]), content=
    #|Some((Minus(Minus(Plus(Plus(Number(1), Multiply(Number(1), Plus(Number(307), Number(7)))), Number(5)), Number(3)), Number(2)), "", @immut/list.T::[]))
  ) // Get the syntax tree
  inspect!((parse_string("1 + 1 * (307 + 7) + 5 - 3 - 2") :
    Option[(BoxedInt, String, @immut/list.T[Token])]), content=
    #|Some((BoxedInt(315), "", @immut/list.T::[]))
  ) // Get the calculation result
  }
```

## まとめ

要約すると、この講義ではパーサーを紹介しました。字句解析と構文解析の概念を紹介し、パーサーコンビネータの定義と実装を説明し、Tagless Finalの概念と実装について拡張しました。追加すべき点がいくつかあります。算術式の場合、Shunting Yardアルゴリズムと呼ばれる単純なアルゴリズムがあり、式をトークンに分割した後に式の値を計算します。構文解析/パーシングはコンピュータサイエンスの重要な分野であり、完全に理解するには多くの時間がかかります。1回の講義ですべてを網羅することは不可能なので、簡単な紹介のみを行いました。スタンフォード大学のCS143コースの講義1-8、『コンパイラ―原理・技法・ツール』の最初の5章、または『現代コンパイラの実装』の最初の3章を参照してください。また、後者の2冊の本は、表紙のデザインからドラゴンブックとタイガーブックと呼ばれることもあります。

追加の演習問題もあります。文字列とトークンストリームのパーサーの構造がほぼ同一であることに気付いたかもしれません。では、異なるストリームと互換性のあるパーサーコンビネータを実装するために、文字列とトークンストリームを抽象化することは可能でしょうか？ぜひ考えてみてください！

最後に、パーサーコンビネータに体現された重要なモジュールプログラミングの考え方に注目していただきたいと思います。小さいものから大きいものへとモジュールを構築し、単純なものから複雑なものへとプログラムを組み立てることができます。モジュール思考を採用することで、複雑さを管理し、無関係な情報を捨てることができます。たとえば、最後にパーサーを定義したとき、コンビネータを使用して構文をほぼ1対1で複製し、パーサーの具体的な実装を気にする必要はありませんでした。このようにしてのみ、より大きく、保守可能で、バグの少ないプログラムを構築できます。