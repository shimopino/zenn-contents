---
title: "RustにおけるVisitorパターン"
emoji: "🦀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
---

## はじめに

最近出版された [インタプリタの作り方](https://amzn.asia/d/9cDKgmF) で紹介されたプログラミング言語「Lox」を Rust での再実装に取り組んでいます。本記事では、その過程で遭遇した Visitor パターンの Rust での表現方法について探ります。

### 結論

enum とパターンマッチで十分そうに感じ、Visitor パターンをあえて採用するケースは少ないかもしれません。
なぜ `serde` では `Visitor` パターンを採用しているのかまでは調査しきれませんでした。

## 解決したい課題は何か？

### 背景

まずは言語実装の背景を理解するために、以下のような構文を有する言語を実装する場合には、どのような処理を実現しなければならないかを考えてみます。ここはすでに内容を理解している方はスキップしてしまって問題ありません。

```rust
2 + 3;
```

大抵の言語では上記のようなソースコードが入力された際に、3 つのステップにわけて処理を進めていきます。

![](visitor-patterns-in-rust/three-step.drawio.png)

入力されたソースコードは、言語の構文を表現する構文解析木として解析され、木をただる形で最終的な値の計算を行います。

ソースコードが解析されるとき、まず最初に字句解析を行い、言語の構文に従ったトークンの列として解析します。次に構文解析を行い、ソースコードは構文解析木（AST: Abstract Syntax Tree）と呼ばれるデータ構造に変換されます。この構文解析木は、ソースコードの構造を表現しています。

構文解析木は、ソースコードの各部分（例えば、変数宣言、関数呼び出し、演算子など）をノードとして表現します。これらのノードは、ソースコードの構造を反映した形で木構造を形成します。

次に、この構文解析木を「ただる」（つまり、全てのノードを訪れる）形で処理が行われます。この処理の中で、各ノードが表現する部分の意味に基づいた計算が行われ、最終的な値が得られます。「Lox」言語でもこの「ただる」処理は Visitor パターンを用いて実装されています。

### 課題

ではもう少し複雑な構文である下記のサンプルに、構文解析木に変換することを考えてみます。

```rust
- 1 * ( 1 + 4 )
```

この構文の構成要素は以下のとおりです。

![](visitor-patterns-in-rust/expressions.drawio.png)

ここから「Lox」言語と同じ式の文法を採用すると、以下のような構文解析木に変換されます。

![](visitor-patterns-in-rust/ast.drawio.png)

このような構文解析木に解析できたことを検証するために、標準出力に構文解析木を出力しようと思うと、このデータ構造の走査と標準出力への表示という 2 つの処理を実行しなければなりません。

Java の記法に従うと書籍上では以下のように各式を表現するクラスを定義しています。

```java
abstract class Expr {
  static class Binary extends Expr {
    // ...
    Binary(Expr left, Token operator, Expr right) { /* ... */ }
  }

  static class Unary extends Expr {
    // ...
    Unary(Token operator, Expr right) { /* ... */ }
  }

  static class Literal extends Expr {
    // ...
    Literal(Object value) { /* ... */ }
  }

  static class Grouping extends Expr {
    // ...
    Grouping(Expr expression) { /* ... */ }
  }
}
```

これらのクラスを使って構文解析木が正しく構築されたのかどうかを標準出力に表示するための処理を追加しようとすると、以下のように対象の式に応じてそれぞれ呼び出すメソッドを変更しながら、再帰的に子側の式を辿っていく方法を取ることになります。

```java
if (expr instanceof Expr.Binary) {
  // Binary に所属する、標準出力に表示するためのメソッドを呼び出す
} else if (expr instanceof Expr.Unary) {
  // Unary に所属する、標準出力に表示するためのメソッドを呼び出す
} else if // ...
```

単純な処理の場合、上記のような直接的な実装でも問題ないかもしれません。新しい式を追加する際には、対象となるクラスに新たなインスタンスメソッドを追加し、`if`文の分岐を増やすことで対応可能です。

しかし、このアプローチにはいくつかの問題点があります。例えば、標準出力への表示から、構文解析木を辿って式を評価する処理に変更したいとします。この場合、すべての式を表すクラスに新たなインスタンスメソッドを追加する必要があります。さらに、それぞれの式の評価に対応するために、大量の `if` 文を書く必要が出てきます。

このような実装は、コードの冗長性を増加させ、保守性を低下させます。また、新しい機能の追加が難しくなり、型チェックの漏れによるランタイムエラーを引き起こす可能性があります。さらに、各クラスが自身の状態を管理するだけでなく、表示や評価などの責任も負うことになってしまいます。

## Visitor パターンとは何か？

Visitor パターンは、新しい操作を追加する際の課題を解決するための効果的な手段です。このパターンの主な特徴は、オブジェクトの構造とその操作を分離することです。これにより、既存のコードを変更せずに新しい動作を追加することが可能になります。

具体的には、Java の場合、各クラスに対応する専用のインタフェース（例えば、 `Expr` クラスの `accept` メソッド）と、そのメソッドの具体的な処理内容を記述するクラス（例えば、 `ExprVisitor` クラス）を用意します。

```java
interface Expr {
  void accept(ExprVisitor visitor);
}

class Binary implements Expr {
  Binary(Expr left, Token operator, Expr right) { /* ... */ }

  @Override
  void accept(ExprVisitor visitor) {
    visitor.visitBinary(this);
  }
}

class Unary implements Expr {
  Unary(Token operator, Expr right) { /* ... */ }

  @Override
  void accept(ExprVisitor visitor) {
    visitor.visitUnary(this);
  }
}

// ...
```

各クラス自体には具体的なメソッドは実装せず、標準出力に表示するなどの具体的な実装はこれらのクラスを受け取る側が実装することで、他の処理でも既存のクラスを変更することなく実現することが可能になります。

```java
interface ExprVisitor {
  void visitBinary(Binary expr);
  void visitUnary(Unary expr);
  // ...
}

class Astprinter implements ExprVisitor {
  @Override
  void visitBinary(Binary expr) {
    System.out.print("(");
    expr.left.accept(this); // ここで実際の expr の型に応じて visit_XXX メソッドを呼び分けることができる
    System.out.print(" ");
    System.out.print(expr.operator);
    System.out.print(" ");
    expr.right.accept(this);
    System.out.print(")");
  }

  @Override
  void visitUnary(Binary expr) {
    System.out.print(expr.operator);
    expr.right.accept(this);
  }

  // ...
}
```

Java では、オーバーライドされたメソッドの呼び出しは動的ディスパッチによって行われます。つまり、Visitor パターンでは、この動的ディスパッチの機能を利用しています。具体的には、 `ExprVisitor` の各 `visit` メソッドが、対応する `Expr` の実装クラスのインスタンスに対して呼び出されます。これにより、各 `Expr` の実装クラスは自身の型に応じた処理を `ExprVisitor` に委譲することができます。

## Rust における課題解決

ここまでは Java での実装例でしたが、このパターンを Rust ではどのように実装すれば良いでしょうか？

### Java における実装パターンを採用する場合

まずは愚直に Java を再現するように、トレイトと構造体を使って無理やり再現してみます。

```rust
trait Expr {
  fn accept(&self, visitor: &dyn Visitor);
}

struct BinaryExpr {
  // 他の式の表現をトレイトオブジェクトとして受け取る
  left: Box<dyn Expr>,
  operator: Token,
  right: Box<dyn Expr>,
}

// Expr を実装することで、対応する処理を Visitor 側に移譲する
impl Expr for BinaryExpr {
  fn accept(&self, visitor: &dyn Visitor) {
    visitor.visit_binary_expr(self)
  }
}

// ...

trait Visitor {
  fn visit_binary_expr(&self, expr: &BinaryExpr);
  fn visit_unary_expr(&self, expr: &UnaryExpr);
  fn visit_literal_expr(&self, expr: &LiteralExpr);
  fn visit_grouping_expr(&self, expr: &GroupingExpr);
}

struct AstPrinter;

impl Visitor for PrintVisitor {
  fn visit_binary_expr(&self, expr: &BinaryExpr) {
    print!("(");
    expr.left.accept(self);
    print!(" {} ", expr.operator);
    expr.right.accept(self);
    print!(")");
  }

  // ...
}
```

トレイトオブジェクトを通じて動的ディスパッチを行うことでなんとか再現はできますが、なかなか直観的には理解が難しいコードだとは思います。

Rust には `enum` という柔軟に型を定義できる機能が備わっているため、今度はこちらを活用してみましょう

### Enum と Visitor パターンの両者を活用する場合

`enum` を活用すれば今までは 1 つ 1 つ独立していた各式の表現を 1 つにまとめることができ、また各式で必要とされるパラメータを個別に持たせることも可能です。

```rust
#[derive(Debug)]
enum Expr {
    Binary(Box<Expr>, BinaryOp, Box<Expr>),
    Unary(UnaryOp, Box<Expr>),
    Literal(Literal),
    Grouping(Box<Expr>),
}

#[derive(Debug)]
enum Literal {
    Number(f64),
}

#[derive(Debug)]
enum UnaryOp {
    Minus,
}

#[derive(Debug)]
enum BinaryOp {
    Plus,
    Star,
}
```

Visitor パターンの一部を活用し、 `Expr` の各式に対して統一的なメソッドを定義しパターンマッチを活用することで、 Visitor がそれぞれの式にアクセスできるようにすることが可能です。

```rust
impl Expr {
    fn accept(&self, visitor: &mut dyn ExprVisitor) {
        match self {
            Expr::Literal(literal) => visitor.visit_literal(literal),
            Expr::Unary(unary_op, expr) => visitor.visit_unary(unary_op, expr),
            Expr::Binary(left_expr, binary_op, right_expr) => {
                visitor.visit_binary(left_expr, binary_op, right_expr)
            }
            Expr::Grouping(expr) => visitor.visit_grouping(expr),
        }
    }
}

trait ExprVisitor {
    fn visit_literal(&mut self, literal: &Literal);
    fn visit_unary(&mut self, unary_op: &UnaryOp, expr: &Expr);
    fn visit_binary(&mut self, left_expr: &Expr, binary_op: &BinaryOp, right_expr: &Expr);
    fn visit_grouping(&mut self, expr: &Expr);
}
```

これで標準出力に表示したい場合には、 `Expr` の実装は一切変更することなく、 `ExprVisitor` の具体実装に標準出力に表示する処理を実装すれば良いだけとなりました。

```rust
struct AstPrinter;

impl AstPrinter {
    fn print(&mut self, expr: &Expr) {
        expr.accept(self);
    }
}

impl ExprVisitor for AstPrinter {
  fn visit_binary(&mut self, left_expr: &Expr, binary_op: &BinaryOp, right_expr: &Expr) {
        print!("(");
        left_expr.accept(self);
        match binary_op {
            BinaryOp::Plus => print!(" + "),
            BinaryOp::Star => print!(" * "),
        }
        right_expr.accept(self);
        print!(")");
    }

    fn visit_unary(&mut self, unary_op: &UnaryOp, expr: &Expr) {
      print!("(");
        match unary_op {
          UnaryOp::Minus => print!("-"),
        }
        expr.accept(self);
        print!(")");
    }

    fn visit_literal(&mut self, literal: &Literal) {
        match literal {
            Literal::Number(n) => print!("{}", n),
        }
    }

    fn visit_grouping(&mut self, expr: &Expr) {
        print!("(");
        expr.accept(self);
        print!(")");
    }
}
```

あとは以下のように `AstPrinter` に対して検証したい表現を渡せば、標準出力に対して期待通り表示してくれることがわかります。

```rust
fn main() {
    // -2 * (1 + 4)
    let tokens = Expr::Binary(
        Box::new(Expr::Unary(
            UnaryOp::Minus,
            Box::new(Expr::Literal(Literal::Number(2.0))),
        )),
        BinaryOp::Star,
        Box::new(Expr::Grouping(Box::new(Expr::Binary(
            Box::new(Expr::Literal(Literal::Number(1.0))),
            BinaryOp::Plus,
            Box::new(Expr::Literal(Literal::Number(4.0))),
        )))),
    );

    let mut printer = AstPrinter;
    printer.print(&tokens);
}
```

[code](https://gist.github.com/rust-play/11a155c7e491726989aa8b8141c5f0ea)

これで最初のパターンと同じように Visitor パターンによる実装コードの移譲を行いながら、 `enum` を使って言語の構文の表現力を向上させることができました。

### Enum とパターンマッチング、再帰呼び出しを採用する場合

1 つ前の実装では以下のような呼び出し順序になっていました。

1. `AstPrinter` の `print` メソッドを実行
2. `Expr` の `accept` メソッドを実行
3. `Expr` 側でパターンマッチングを行い、 `ExprVisitor` のどのメソッドを呼び出すのかを決定
4. `ExprVisitor` の具体的な実装を呼び出し

しかしよく考えると、 `print` メソッドの引数で渡される `Expr` は既に `enum` で表現されているためパターンマッチングを行うことが可能であり、その際に各式のパラメータから内部の `Expr` を取り出して再帰呼び出しを行えば、 `Visitor` パターン自体使う必要はなくなります。

```rust
struct AstPrinter;

impl AstPrinter {
    fn print(&mut self, e: &Expr) {
        use crate::Expr::*;
        use crate::Literal::*;

        match e {
            Literal(literal) => match literal {
                Number(n) => print!("{}", n),
            },
            Unary(unary_op, expr) => {
                print!("(");
                match unary_op {
                    UnaryOp::Minus => print!("-"),
                }
                self.print(expr);
                print!(")");
            }
            Binary(left_expr, binary_op, right_op) => {
                print!("(");
                self.print(left_expr);
                match binary_op {
                    BinaryOp::Plus => print!(" + "),
                    BinaryOp::Star => print!(" * "),
                }
                self.print(right_op);
                print!(")");
            }
            Grouping(expr) => {
                print!("(");
                self.print(expr);
                print!(")");
            }
        }
    }
}
```

これでトレイトを一切使用せずに、Visitor パターンで実現していたことをかなりシンプルな形に記述することができました。

## まとめ

Java などのオブジェクト指向言語などの文脈で登場していた Visitor パターンなどを、Rust で実装するとどうなるのかをみてきました。Rust の強力な特性である `enum` とパターンマッチングを活用することで Visitor パターンで実現していた内容をよりシンプルに記述できることもわかりました。

ただし、これは Rust では Visitor パターンが使われていないことを意味しません。実際に有名なライブラリである [serde](https://github.com/serde-rs/serde) では、独自のデシリアライズ処理を記述する際に、以下のように [Visitor パターンに従った実装](https://docs.rs/serde/latest/serde/de/trait.Visitor.html) を行う必要があります。

ただ、時間が足りずなぜ `serde` では Visitor パターンが採用されたかまでは調査し切ることができませんでした。下記の記事ではパフォーマンスが向上したとあったため、時間があるときにさらに調査してみようと思います。

- [Rewriting Rust Serialization, Part 3: Introducing Serde](https://erickt.github.io/blog/2014/12/13/rewriting-rust-serialization/)
- [Rewriting Rust Serialization: Part 4: Serde2 Is Ready!](https://erickt.github.io/blog/2015/02/16/rewriting-rust-serialization-there-can-be-only-one-serde/)
