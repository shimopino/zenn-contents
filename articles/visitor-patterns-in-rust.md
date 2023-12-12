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

enum とパターンマッチで十分そうに感じ、Visitor パターンを使う理由は特に思い至りませんでした...
なぜ `serde` では `Visitor` パターンを採用しているのかはわかりません...

## 解決したい課題は何か？

### 背景

まずは言語実装の背景を理解するために、以下のような構文を有する言語を実装する場合には、どのような処理を実現しなければならないかを考えてみます。ここはすでに内容を理解している方はスキップしてしまって問題ありません。

```rust
2 + 3;
```

大抵の言語では上記のようなソースコードが入力された際に、3 つのステップにわけて処理を進めていきます。

![](visitor-patterns-in-rust/three-step.drawio.png)

入力されたソースコードは言語の構文を表現する構文解析木として解析され、木をただる形で最終的な値の計算を行います。

### 課題

ではもう少し複雑な構文である下記のサンプルに、構文木に変換することを考えてみます。

```rust
- 1 * ( 1 + 4 )
```

この構文の構成要素は以下のとおりです。

![](visitor-patterns-in-rust/expressions.drawio.png)

ここから「Lox」言語と同じ式の文法を採用すると、以下のような構文木に変換されます。

![](visitor-patterns-in-rust/ast.drawio.png)

このような構文木に解析できたことを検証するために、標準出力に構文木を出力しようと思うと、このデータ構造の走査と標準出力への表示という 2 つの処理を実行しなければなりません。

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

これらのクラスを使って構文木が正しく構築されたのかどうかを標準出力に表示するための処理を追加しようとすると、以下のように対象の式に応じてそれぞれ呼び出すメソッドを変更しながら、再帰的に子側の式を辿っていく方法を取ることになる。

```java
if (expr instanceof Expr.Binary) {
  // Binary に所属する、標準出力に表示するためのメソッドを呼び出す
} else if (expr instanceof Expr.Unary) {
  // Unary に所属する、標準出力に表示するためのメソッドを呼び出す
} else if // ...
```

今回のような単純な処理であれば上記の実装にしておくと、追加の式を実装する場合には対象クラスのインスタンスメソッドに標準出力に表示するためのメソッドの追加と、 `if` 文の分岐を追加することで対応できるかもしれない。

ただし、今度は標準出力への表示ではなく、実際に構文木を辿って式を評価しようとした場合、各式を表すクラスのすべてに専用のインスタンスメソッドを追加し、同じように大量の `if` 文を記述する必要が出てきてしまう。

## Visitor パターンとは何か？

Visitor パターンは、上記の課題を解決するための一つの効果的な方法です。このパターンは、オブジェクトの構造とその操作を分離することにより、新しい操作を追加することが容易になります。これにより、既存のコードを変更することなく、新しい動作を追加することが可能になります。

Java の場合には、各クラスに対応する専用のインタフェース（ `Expr` の `accept` メソッド ）と、そのメソッドの具体的な処理内容を記述するクラス（ `ExprVisitor` ）を用意することで、元々のクラスに変更を加えることなく、同じような処理を実現することができるようになります。

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
    expr.left.accept(this);
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

## Rust における課題解決

ここまでは Java での実装例でしたが、このパターンを Rust ではどのように実装すれば良いでしょうか？

### Java における実装パターンを採用する場合

まずは愚直に Java を再現するように、トレイトと構造体を使って無理やり再現してみます。

```rust
trait Expr {
  fn accept(&self, visitor: &dyn Visitor);
}

struct BinaryExpr {
  left: Box<dyn Expr>,
  operator: Token,
  right: Box<dyn Expr>,
}

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
    Ok(())
  }

  // ...
}
```

はい。あまりにも面倒ですね。

Rust には `enum` という柔軟な型を定義できる機能が備わっているため、今度はこちらを活用してみましょう

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

Visitor パターンの一部を活用し、 `Expr` の各式に対して統一的なメソッドを定義し、パターンマッチを活用することで、それぞれの式にアクセスできるようにすることが可能である。

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

これで標準出力に表示したい場合には、 `Expr` の実装は一切変更することなく、 `ExprVisitor` の具体実装に標準出力に表示する処理を実装すれば良いだけとなった。

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

あとは以下のように `AstPrinter` に対して検証したい表現を渡せば、標準出力に対して期待通り表示してくれることがわかる。

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

### Enum とパターンマッチング、再帰呼び出しを採用する場合

1 つ前の実装では以下のような呼び出し順序になっていた。

1. `AstPrinter` の `print` メソッドが実行
2. `Expr` の `accept` メソッドを実行
3. `Expr` 側でパターンマッチングを行い、 `ExprVisitor` のどのメソッドを呼び出すのかを決定する
4. `ExprVisitor` の具体的な実装が呼び出される

しかしよく考えると、 `print` メソッドの引数で渡される `Expr` からパターンマッチングを行うことは可能であり、その際に各式のパラメータから内部の `Expr` を取り出して再帰呼び出しを行えば、 `Visitor` パターン自体使う必要はなくなる。

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

トレイトを利用せずに、Visitor パターンで実現していたことをかなりシンプルに記述することができた。

### まとめ

ここまでの感触では、

また、特定の言語ではプラクティスであるとみなされていたデザインパターンも、別の言語ではその言語が提供する機能を活用することでよりシンプルに解決できることがわかったと思います。

## 実際の crate での適用例

## おわりに
