---
title: "Rustにおけるエラーハンドリング"
emoji: "🦀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: true
---

Rustは、強力な型システムを採用しており、そのおかげでバグの少ない堅牢なコードを記述することのできる言語です。この型システムのサポートを活用して、型安全なエラーハンドリングの仕組みが提供されています。

本記事では、Rustにおけるエラーハンドリングの概要を把握し、よく利用されているクレートがどのような問題を解決するものなのかを理解することを目的にしています。

## Rustにおけるエラーハンドリングの基本

Rustでは、エラーハンドリングを行う方法として主に2つのアプローチがあります。

|panic|Result|
|:--|:--|
|復帰不可能なエラー|復帰可能なエラー|
|プログラムの実行を中断し、スタックを巻き戻すことでエラーを報告する。プログラム自身のバグに起因すると思われる問題が発生した時に起こる。|問題が発生した際にプログラムの実行を継続することが可能であり、エラーに応じて適切な処理を行うことができる。|

本記事では `Result` 型に焦点を当てて解説します。

### Result型

エラー処理を行う際に標準ライブラリから提供される `Result` 型を利用できます。

`Result` 型は、標準ライブラリの `std::result` モジュールで定義されており、成功時の戻り値と失敗時の戻り値の両方を表現することができます。以下に示すように、 `Result` 型は成功時に `Ok` 値を、失敗時に` Err` 値を保持します。

```rust
// https://doc.rust-lang.org/std/result/index.html
enum Result<T, E> {
   Ok(T),
   Err(E),
}
```

例えば、整数の割り算を行う関数を考えます。0で割り算を行おうとした場合には失敗を表現し、それ以外の場合には成功を表現することで、この関数が返しうるすべての範囲を表現できます。

```rust
fn divide(numerator: i32, denominator: i32) -> Result<i32, String> {
    if denominator == 0 {
        Err("Divide by 0".to_string())
    } else {
        Ok(numerator / denominator)
    }
}
```

この関数を実際に使用すると、以下のように成功時と失敗時でそれぞれ異なる値を取得していることがわかります。

```rust
fn main() {
    // 推論される型はどちらも Result<i32, String>
    let success = divide(10, 2);
    assert_eq!(success, Ok(5));

    let failure = divide(10, 0);
    assert_eq!(failure, Err("Divide by 0".to_string()));
}
```

[再現コード](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&code=fn+main%28%29+%7B%0A++++%2F%2F+%E6%8E%A8%E8%AB%96%E3%81%95%E3%82%8C%E3%82%8B%E5%9E%8B%E3%81%AF%E3%81%A9%E3%81%A1%E3%82%89%E3%82%82+Result%3Ci32%2C+String%3E%0A++++let+success+%3D+divide%2810%2C+2%29%3B%0A++++assert_eq%21%28success%2C+Ok%285%29%29%3B%0A%0A++++let+failure+%3D+divide%2810%2C+0%29%3B%0A++++assert_eq%21%28failure%2C+Err%28%22Divide+by+0%22.to_string%28%29%29%29%3B%0A%7D%0A%0Afn+divide%28numerator%3A+i32%2C+denominator%3A+i32%29+-%3E+Result%3Ci32%2C+String%3E+%7B%0A++++if+denominator+%3D%3D+0+%7B%0A++++++++Err%28%22Divide+by+0%22.to_string%28%29%29%0A++++%7D+else+%7B%0A++++++++Ok%28numerator+%2F+denominator%29%0A++++%7D%0A%7D)

Rustのパターンマッチング機能と早期return機能を利用すれば、以下のように成功時には値の中身を取り出して変数に代入し、失敗時には即座に結果を関数から返却することができます。

```rust
fn early_return() -> Result<(), String> {
    let value = match divide(10, 5) {
        // 成功時には中身を取り出して変数に代入する
        Ok(value) => value,
        // 失敗時にはこの時点で、結果を関数から返却する
        Err(e) => return Err(e),
    };

    // 値は 2 であり中身が取り出されている
    println!("値は {} であり中身が取り出されている", value);

    Ok(())
}
```

[再現コード](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&code=fn+main%28%29+%7B%0A++++early_return%28%29%3B%0A%7D%0A%0Afn+early_return%28%29+-%3E+Result%3C%28%29%2C+String%3E+%7B%0A++++let+value+%3D+match+divide%2810%2C+5%29+%7B%0A++++++++%2F%2F+%E6%88%90%E5%8A%9F%E6%99%82%E3%81%AB%E3%81%AF%E4%B8%AD%E8%BA%AB%E3%82%92%E5%8F%96%E3%82%8A%E5%87%BA%E3%81%97%E3%81%A6%E5%A4%89%E6%95%B0%E3%81%AB%E4%BB%A3%E5%85%A5%E3%81%99%E3%82%8B%0A++++++++Ok%28value%29+%3D%3E+value%2C%0A++++++++%2F%2F+%E5%A4%B1%E6%95%97%E6%99%82%E3%81%AB%E3%81%AF%E3%81%93%E3%81%AE%E6%99%82%E7%82%B9%E3%81%A7%E3%80%81%E7%B5%90%E6%9E%9C%E3%82%92%E9%96%A2%E6%95%B0%E3%81%8B%E3%82%89%E8%BF%94%E5%8D%B4%E3%81%99%E3%82%8B%0A++++++++Err%28e%29+%3D%3E+return+Err%28e%29%2C%0A++++%7D%3B%0A%0A++++%2F%2F+%E5%80%A4%E3%81%AF+2+%E3%81%A7%E3%81%82%E3%82%8A%E4%B8%AD%E8%BA%AB%E3%81%8C%E5%8F%96%E3%82%8A%E5%87%BA%E3%81%95%E3%82%8C%E3%81%A6%E3%81%84%E3%82%8B%0A++++println%21%28%22%E5%80%A4%E3%81%AF+%7B%7D+%E3%81%A7%E3%81%82%E3%82%8A%E4%B8%AD%E8%BA%AB%E3%81%8C%E5%8F%96%E3%82%8A%E5%87%BA%E3%81%95%E3%82%8C%E3%81%A6%E3%81%84%E3%82%8B%22%2C+value%29%3B%0A%0A++++Ok%28%28%29%29%0A%7D%0A%0Afn+divide%28numerator%3A+i32%2C+denominator%3A+i32%29+-%3E+Result%3Ci32%2C+String%3E+%7B%0A++++if+denominator+%3D%3D+0+%7B%0A++++++++Err%28%22Divide+by+0%22.to_string%28%29%29%0A++++%7D+else+%7B%0A++++++++Ok%28numerator+%2F+denominator%29%0A++++%7D%0A%7D)

Rustには [`?`](https://doc.rust-lang.org/std/result/index.html#the-question-mark-operator-) というシンタックスシュガーが用意されており、上記で実行した内容をよりシンプルな構文で再現することができます。

```rust
fn early_return() -> Result<(), String> {
    // 成功時には中身を取り出して変数に代入する
    // 失敗時にはこの時点で、結果を関数から返却する
    let value = divide(10, 5)?;

    // 値は 2 であり中身が取り出されている
    println!("値は {} であり中身が取り出されている", value);

    Ok(())
}
```

これでRustが提供している `Result` 型がどのようなものなのか概要を掴むことができました。

### Err 型で自作型を返却する

作成した `divide` 関数の返却値の型は `Result<i32, String>` となっていますが、すべての関数の返り値をこのように設計した場合、呼び出し元では型を見てもどのようなエラーが発生する可能性があるのか把握することができません。

そのため、以下のような失敗時の専用の型を用意して、明確に他の返り値を分離させることができます。

```rust
struct DivideByZero;
```

この型を使用すれば以下のように返り値の型を明確に表現することが可能になります。

```rust
// 呼び出し元は DivideByZero という型からどのようなエラーが発生する可能性があるのか把握できる
fn divide(numerator: i32, denominator: i32) -> Result<i32, DivideByZero> {
    if denominator == 0 {
        Err(DivideByZero)
    } else {
        Ok(numerator / denominator)
    }
}
```

しかし元々この関数を呼び出していた `early_return` 関数は、返り値の型と関数が返す型が合わない状態になってしまうためコンパイルエラーが発生してしまいます。

```rust
fn early_return() -> Result<(), String> {
    // 型が合わない
    let value = divide(10, 5)?;
    
    // 値は 2 であり中身が取り出されている
    println!("値は {} であり中身が取り出されている", value);

    Ok(())
}
```

実際にコンパイルエラーを確認すると、以下のように `DivideByZero` 型から `String` 型に型変換することができないことがわかります。

```bash
error[E0277]: `?` couldn't convert the error to `String`
 --> src/main.rs:8:30
  |
5 | fn early_return() -> Result<(), String> {
  |                      ------------------ expected `String` because of this
...
8 |     let value = divide(10, 5)?;
  |                              ^ the trait `From<DivideByZero>` is not implemented for `String`
  |
  = note: the question mark operation (`?`) implicitly performs a conversion on the error value using the `From` trait
  = help: the following other types implement trait `From<T>`:
            <String as From<&String>>
            <String as From<&mut str>>
            <String as From<&str>>
            <String as From<Box<str>>>
            <String as From<Cow<'a, str>>>
            <String as From<char>>
  = note: required for `Result<(), String>` to implement `FromResidual<Result<Infallible, DivideByZero>>`
```

[再現コード](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&code=fn+main%28%29+%7B%0A++++early_return%28%29%3B%0A%7D%0A%0Afn+early_return%28%29+-%3E+Result%3C%28%29%2C+String%3E+%7B%0A++++%2F%2F+%E6%88%90%E5%8A%9F%E6%99%82%E3%81%AB%E3%81%AF%E4%B8%AD%E8%BA%AB%E3%82%92%E5%8F%96%E3%82%8A%E5%87%BA%E3%81%97%E3%81%A6%E5%A4%89%E6%95%B0%E3%81%AB%E4%BB%A3%E5%85%A5%E3%81%99%E3%82%8B%0A++++%2F%2F+%E5%A4%B1%E6%95%97%E6%99%82%E3%81%AB%E3%81%AF%E3%81%93%E3%81%AE%E6%99%82%E7%82%B9%E3%81%A7%E3%80%81%E7%B5%90%E6%9E%9C%E3%82%92%E9%96%A2%E6%95%B0%E3%81%8B%E3%82%89%E8%BF%94%E5%8D%B4%E3%81%99%E3%82%8B%0A++++let+value+%3D+divide%2810%2C+5%29%3F%3B%0A%0A++++%2F%2F+%E5%80%A4%E3%81%AF+2+%E3%81%A7%E3%81%82%E3%82%8A%E4%B8%AD%E8%BA%AB%E3%81%8C%E5%8F%96%E3%82%8A%E5%87%BA%E3%81%95%E3%82%8C%E3%81%A6%E3%81%84%E3%82%8B%0A++++println%21%28%22%E5%80%A4%E3%81%AF+%7B%7D+%E3%81%A7%E3%81%82%E3%82%8A%E4%B8%AD%E8%BA%AB%E3%81%8C%E5%8F%96%E3%82%8A%E5%87%BA%E3%81%95%E3%82%8C%E3%81%A6%E3%81%84%E3%82%8B%22%2C+value%29%3B%0A%0A++++Ok%28%28%29%29%0A%7D%0A%0Astruct+DivideByZero%3B%0A%0A%2F%2F+%E5%91%BC%E3%81%B3%E5%87%BA%E3%81%97%E5%85%83%E3%81%AF+DivideByZero+%E3%81%A8%E3%81%84%E3%81%86%E5%9E%8B%E3%81%8B%E3%82%89%E3%81%A9%E3%81%AE%E3%82%88%E3%81%86%E3%81%AA%E3%82%A8%E3%83%A9%E3%83%BC%E3%81%8C%E7%99%BA%E7%94%9F%E3%81%99%E3%82%8B%E5%8F%AF%E8%83%BD%E6%80%A7%E3%81%8C%E3%81%82%E3%82%8B%E3%81%AE%E3%81%8B%E6%8A%8A%E6%8F%A1%E3%81%A7%E3%81%8D%E3%82%8B%0Afn+divide%28numerator%3A+i32%2C+denominator%3A+i32%29+-%3E+Result%3Ci32%2C+DivideByZero%3E+%7B%0A++++if+denominator+%3D%3D+0+%7B%0A++++++++Err%28DivideByZero%29%0A++++%7D+else+%7B%0A++++++++Ok%28numerator+%2F+denominator%29%0A++++%7D%0A%7D)

ここでは `early_return` 関数の返り値の型を修正することも一つの対応方法ですが、今回はこのコンパイルエラーを解決する別の方法を詳しく見てみましょう。

エラーメッセージでは、`From` トレイトが実装されていないと言われています。実は、シンタックスシュガーである `?` を使うと、型推論に基づいて暗黙的に `From` トレイトの実装を呼び出しています。

https://doc.rust-lang.org/std/convert/trait.From.html

この `From` トレイトを使って型変換を行うことで、さまざまな関数を組み合わせることができます。

今回は `DivideByZero` という独自の型を `String` 型に変換するための実装を次のように追加しましょう。

```rust
impl From<DivideByZero> for String {
    // 値を消費する
    fn from(_value: DivideByZero) -> Self {
        println!("convert DivideByZero to 'Divide by 0' String");
        "Divide By 0".to_string()
    }
}
```

[再現コード](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&code=fn+main%28%29+%7B%0A++++early_return%28%29%3B%0A%7D%0A%0Afn+early_return%28%29+-%3E+Result%3C%28%29%2C+String%3E+%7B%0A++++%2F%2F+%E6%88%90%E5%8A%9F%E6%99%82%E3%81%AB%E3%81%AF%E4%B8%AD%E8%BA%AB%E3%82%92%E5%8F%96%E3%82%8A%E5%87%BA%E3%81%97%E3%81%A6%E5%A4%89%E6%95%B0%E3%81%AB%E4%BB%A3%E5%85%A5%E3%81%99%E3%82%8B%0A++++%2F%2F+%E5%A4%B1%E6%95%97%E6%99%82%E3%81%AB%E3%81%AF%E3%81%93%E3%81%AE%E6%99%82%E7%82%B9%E3%81%A7%E3%80%81%E7%B5%90%E6%9E%9C%E3%82%92%E9%96%A2%E6%95%B0%E3%81%8B%E3%82%89%E8%BF%94%E5%8D%B4%E3%81%99%E3%82%8B%0A++++let+value+%3D+divide%2810%2C+5%29%3F%3B%0A%0A++++%2F%2F+%E5%80%A4%E3%81%AF+2+%E3%81%A7%E3%81%82%E3%82%8A%E4%B8%AD%E8%BA%AB%E3%81%8C%E5%8F%96%E3%82%8A%E5%87%BA%E3%81%95%E3%82%8C%E3%81%A6%E3%81%84%E3%82%8B%0A++++println%21%28%22%E5%80%A4%E3%81%AF+%7B%7D+%E3%81%A7%E3%81%82%E3%82%8A%E4%B8%AD%E8%BA%AB%E3%81%8C%E5%8F%96%E3%82%8A%E5%87%BA%E3%81%95%E3%82%8C%E3%81%A6%E3%81%84%E3%82%8B%22%2C+value%29%3B%0A%0A++++Ok%28%28%29%29%0A%7D%0A%0Astruct+DivideByZero%3B%0A%0Aimpl+From%3CDivideByZero%3E+for+String+%7B%0A++++%2F%2F+%E5%80%A4%E3%82%92%E6%B6%88%E8%B2%BB%E3%81%99%E3%82%8B%0A++++fn+from%28_value%3A+DivideByZero%29+-%3E+Self+%7B%0A++++++++println%21%28%22convert+DivideByZero+to+%27Divide+by+0%27+String%22%29%3B%0A++++++++%22Divide+By+0%22.to_string%28%29%0A++++%7D%0A%7D%0A%0A%2F%2F+%E5%91%BC%E3%81%B3%E5%87%BA%E3%81%97%E5%85%83%E3%81%AF+DivideByZero+%E3%81%A8%E3%81%84%E3%81%86%E5%9E%8B%E3%81%8B%E3%82%89%E3%81%A9%E3%81%AE%E3%82%88%E3%81%86%E3%81%AA%E3%82%A8%E3%83%A9%E3%83%BC%E3%81%8C%E7%99%BA%E7%94%9F%E3%81%99%E3%82%8B%E5%8F%AF%E8%83%BD%E6%80%A7%E3%81%8C%E3%81%82%E3%82%8B%E3%81%AE%E3%81%8B%E6%8A%8A%E6%8F%A1%E3%81%A7%E3%81%8D%E3%82%8B%0Afn+divide%28numerator%3A+i32%2C+denominator%3A+i32%29+-%3E+Result%3Ci32%2C+DivideByZero%3E+%7B%0A++++if+denominator+%3D%3D+0+%7B%0A++++++++Err%28DivideByZero%29%0A++++%7D+else+%7B%0A++++++++Ok%28numerator+%2F+denominator%29%0A++++%7D%0A%7D)

こうすることで暗黙的に `from` メソッドが実行され、以下のように呼び出しもとに変換されたエラーが返却されていることがわかります。

```rust
fn main() {
    let result = early_return();
    // from によって変換された値が返ってきていることがわかる
    assert_eq!(result, Err("Divide By 0".to_string()));
}

fn early_return() -> Result<(), String> {
    // Err が返却される条件で引数を指定する
    // 暗黙的に DivideByZero -> String に変換するための from メソッドが呼ばれる
    let value = divide(10, 0)?;

    // ,,,

    Ok(())
}
```

これは以下のように明示的に `from` を読んだ時と同じ挙動になります。

```rust
fn early_return() -> Result<(), String> {
    let value = match divide(10, 0) {
        Ok(value) => value,
        // e: DivideByZero と型推論される
        // そのため自動的に DivideByZero の from 実装が呼び出される
        Err(e) => return Err(From::from(e)),
    };

    // ...

    Ok(())
}
```

なお `From` トレイトを実装することで自動的に `Into` トレイトも実装されるため、以下のように型変換を行うことも可能です。

```rust
let sample: String = DivideByZero.into();
```

これで自作した型を `Result` 型に適用したり、異なる型同士で型変換を行う方法がわかりました。

### Error トレイトを実装する

`Result` 型の `E` に指定する型として、文字列や独自の型を使うこともできますが、標準ライブラリが提供している `Error` トレイトを実装したものを使用することが一般的な慣習です。

https://doc.rust-lang.org/std/error/trait.Error.html

このトレイトは次のように定義されており、 `Debug` トレイトや `Display` トレイトが境界として指定されているため、これらの実装が必要になります。

```rust
pub trait Error: Debug + Display {
    // Provided methods
    fn source(&self) -> Option<&(dyn Error + 'static)> { ... }
    fn description(&self) -> &str { ... }
    fn cause(&self) -> Option<&dyn Error> { ... }
    fn provide<'a>(&'a self, demand: &mut Demand<'a>) { ... }
}
```

これらのトレイトが設定されているおかげで、エラーの詳細な情報を `"{:?}"` を使用して デバッグ出力できるようになったり、エラー情報を人間が理解しやすい形式で `"{}"` を使用して出力できるようになります。

また、 `source` メソッドが定義されており、このメソッドを使ってエラーの原因を追跡することができます。デフォルト実装が提供されているため、実装する必要はありませんが、内部エラーをラップしている場合にはオーバーライドすることが推奨されています。

先ほど作成した `DivideByZero` に対しては、 `Debug` 属性などのアトリビュートも利用して以下のようにトレイトを実装することができます。

```rust
#[derive(Debug)]
struct DivideByZero;

impl std::error::Error for DivideByZero {}

impl std::fmt::Display for DivideByZero {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Divided by 0")
    }
}
```

この変更により、以下のように `From` トレイトの実装を修正し、実際に標準出力でエラーを表示してみることで、実装した `Display` トレイトの内容が正しく反映されていることが確認できます。

```rust
impl From<DivideByZero> for String {
    fn from(value: DivideByZero) -> Self {
        // これは以下のように出力されます:
        // Display: [DividedByZero] Divided by 0, Debug: DivideByZero
        println!("Display: {}, Debug: {:?}", value, value);

        "Divide By 0".to_string()
    }
}
```

[再現コード](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&code=fn+main%28%29+%7B%0A++++let+failure+%3D+early_return%28%29%3B%0A++++assert_eq%21%28failure%2C+Err%28%22Divide+By+0%22.to_string%28%29%29%29%3B%0A%7D%0A%0Afn+early_return%28%29+-%3E+Result%3C%28%29%2C+String%3E+%7B%0A++++%2F%2F+%E6%88%90%E5%8A%9F%E6%99%82%E3%81%AB%E3%81%AF%E4%B8%AD%E8%BA%AB%E3%82%92%E5%8F%96%E3%82%8A%E5%87%BA%E3%81%97%E3%81%A6%E5%A4%89%E6%95%B0%E3%81%AB%E4%BB%A3%E5%85%A5%E3%81%99%E3%82%8B%0A++++%2F%2F+%E5%A4%B1%E6%95%97%E6%99%82%E3%81%AB%E3%81%AF%E3%81%93%E3%81%AE%E6%99%82%E7%82%B9%E3%81%A7%E3%80%81%E7%B5%90%E6%9E%9C%E3%82%92%E9%96%A2%E6%95%B0%E3%81%8B%E3%82%89%E8%BF%94%E5%8D%B4%E3%81%99%E3%82%8B%0A++++let+value+%3D+divide%2810%2C+0%29%3F%3B%0A%0A++++%2F%2F+%E5%80%A4%E3%81%AF+2+%E3%81%A7%E3%81%82%E3%82%8A%E4%B8%AD%E8%BA%AB%E3%81%8C%E5%8F%96%E3%82%8A%E5%87%BA%E3%81%95%E3%82%8C%E3%81%A6%E3%81%84%E3%82%8B%0A++++println%21%28%22%E5%80%A4%E3%81%AF+%7B%7D+%E3%81%A7%E3%81%82%E3%82%8A%E4%B8%AD%E8%BA%AB%E3%81%8C%E5%8F%96%E3%82%8A%E5%87%BA%E3%81%95%E3%82%8C%E3%81%A6%E3%81%84%E3%82%8B%22%2C+value%29%3B%0A%0A++++Ok%28%28%29%29%0A%7D%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+DivideByZero%3B%0A%0Aimpl+std%3A%3Aerror%3A%3AError+for+DivideByZero+%7B%7D%0A%0Aimpl+std%3A%3Afmt%3A%3ADisplay+for+DivideByZero+%7B%0A++++fn+fmt%28%26self%2C+f%3A+%26mut+std%3A%3Afmt%3A%3AFormatter%3C%27_%3E%29+-%3E+std%3A%3Afmt%3A%3AResult+%7B%0A++++++++write%21%28f%2C+%22Divided+by+0%22%29%0A++++%7D%0A%7D%0A%0Aimpl+From%3CDivideByZero%3E+for+String+%7B%0A++++fn+from%28value%3A+DivideByZero%29+-%3E+Self+%7B%0A++++++++%2F%2F+%E3%81%93%E3%82%8C%E3%81%AF%E4%BB%A5%E4%B8%8B%E3%81%AE%E3%82%88%E3%81%86%E3%81%AB%E5%87%BA%E5%8A%9B%E3%81%95%E3%82%8C%E3%81%BE%E3%81%99%3A%0A++++++++%2F%2F+Display%3A+%5BDividedByZero%5D+Divided+by+0%2C+Debug%3A+DivideByZero%0A++++++++println%21%28%22Display%3A+%7B%7D%2C+Debug%3A+%7B%3A%3F%7D%22%2C+value%2C+value%29%3B%0A%0A++++++++%22Divide+By+0%22.to_string%28%29%0A++++%7D%0A%7D%0A%0A%2F%2F+%E5%91%BC%E3%81%B3%E5%87%BA%E3%81%97%E5%85%83%E3%81%AF+DivideByZero+%E3%81%A8%E3%81%84%E3%81%86%E5%9E%8B%E3%81%8B%E3%82%89%E3%81%A9%E3%81%AE%E3%82%88%E3%81%86%E3%81%AA%E3%82%A8%E3%83%A9%E3%83%BC%E3%81%8C%E7%99%BA%E7%94%9F%E3%81%99%E3%82%8B%E5%8F%AF%E8%83%BD%E6%80%A7%E3%81%8C%E3%81%82%E3%82%8B%E3%81%AE%E3%81%8B%E6%8A%8A%E6%8F%A1%E3%81%A7%E3%81%8D%E3%82%8B%0Afn+divide%28numerator%3A+i32%2C+denominator%3A+i32%29+-%3E+Result%3Ci32%2C+DivideByZero%3E+%7B%0A++++if+denominator+%3D%3D+0+%7B%0A++++++++Err%28DivideByZero%29%0A++++%7D+else+%7B%0A++++++++Ok%28numerator+%2F+denominator%29%0A++++%7D%0A%7D)

### 複数のエラー型を組み合わせる

アプリケーション全体でエラー型を作成する際には、サードパーティのクレートで定義されているエラー型を含め、　 `enum` を使って複数のエラーを表現することがあります。

そのような場合でも、 `From` トレイトを利用してアプリケーション全体の型に変換することが可能です。

```rust
// 例えば、以下で定義されているErrorが、sqlx::Error や reqwest::Error などのサードパーティエラー型でも適用可能
#[derive(Debug)]
struct CustomErrorType1;

#[derive(Debug)]
struct CustomErrorType2;

#[derive(Debug)]
enum ApplicationError {
    Type1(CustomErrorType1),
    Type2(CustomErrorType2),
}

impl std::error::Error for ApplicationError {}

impl std::fmt::Display for ApplicationError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            ApplicationError::Type1(_) => write!(f, "Error type 1"),
            ApplicationError::Type2(_) => write!(f, "Error type 2"),
        }
    }
}
```

例えばアプリケーションを構成する関数の中に、以下のように `CustomErrorType1` を返すようなものが定義されているとします。

```rust
fn some_function_custom_error_1() -> Result<i32, CustomErrorType1> {
    // ...
}
```

この関数を以下のように利用してもそのままでは型変換できずにコンパイルエラーになってしまいます。

```rust
fn main() -> Result<(), ApplicationError> {
    // 以下の関数では CustomErrorType1 がエラーとして返却される
    let result = some_function_custom_error_1()?;

    // ...
}
```

このような場合にはそれぞれの型に対して `From` トレイトを実装して型推論から暗黙的に型変換のための関数を呼び出すようにすればコンパイルエラーが発生することはありません。

```rust
impl From<CustomErrorType1> for ApplicationError {
    fn from(error: CustomErrorType1) -> Self {
        ApplicationError::Type1(error)
    }
}

impl From<CustomErrorType2> for ApplicationError {
    fn from(error: CustomErrorType2) -> Self {
        ApplicationError::Type2(error)
    }
}
```

複数のエラーが存在していたとしても、 `enum` を利用してアプリケーション内で発生する可能性のあるエラーをまとめて、 `From` トレイトを実装することでスムーズに型変換を行うことが可能です。

```rust
fn main() -> Result<(), ApplicationError> {
    // それぞれ異なるErr型だが、Fromトレイトによる型変換によりApplicationErrorに変換される
    let result1 = some_function_custom_error1(5)?;
    let result2 = some_function_custom_error2(5)?;
    
    println!("result1: {}, result2: {}", result1, result2);
    
    Ok(())
}
```

[再現コード](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&code=%23%5Bderive%28Debug%29%5D%0Astruct+CustomErrorType1%3B%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+CustomErrorType2%3B%0A%0A%23%5Bderive%28Debug%29%5D%0Aenum+ApplicationError+%7B%0A++++Type1%28CustomErrorType1%29%2C%0A++++Type2%28CustomErrorType2%29%2C%0A%7D%0A%0Aimpl+std%3A%3Aerror%3A%3AError+for+ApplicationError+%7B%7D%0A%0Aimpl+std%3A%3Afmt%3A%3ADisplay+for+ApplicationError+%7B%0A++++fn+fmt%28%26self%2C+f%3A+%26mut+std%3A%3Afmt%3A%3AFormatter%3C%27_%3E%29+-%3E+std%3A%3Afmt%3A%3AResult+%7B%0A++++++++match+self+%7B%0A++++++++++++ApplicationError%3A%3AType1%28_%29+%3D%3E+write%21%28f%2C+%22Error+type+1%22%29%2C%0A++++++++++++ApplicationError%3A%3AType2%28_%29+%3D%3E+write%21%28f%2C+%22Error+type+2%22%29%2C%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Aimpl+From%3CCustomErrorType1%3E+for+ApplicationError+%7B%0A++++fn+from%28error%3A+CustomErrorType1%29+-%3E+Self+%7B%0A++++++++ApplicationError%3A%3AType1%28error%29%0A++++%7D%0A%7D%0A%0Aimpl+From%3CCustomErrorType2%3E+for+ApplicationError+%7B%0A++++fn+from%28error%3A+CustomErrorType2%29+-%3E+Self+%7B%0A++++++++ApplicationError%3A%3AType2%28error%29%0A++++%7D%0A%7D%0A%0Afn+some_function_custom_error1%28a%3A+i32%29+-%3E+Result%3Ci32%2C+CustomErrorType1%3E+%7B%0A++++if+a+%3D%3D+0+%7B%0A++++++++Err%28CustomErrorType1%29%0A++++%7D+else+%7B%0A++++++++Ok%28a%29%0A++++%7D%0A%7D%0A%0Afn+some_function_custom_error2%28b%3A+i32%29+-%3E+Result%3Ci32%2C+CustomErrorType2%3E+%7B%0A++++if+b+%3E+10+%7B%0A++++++++Err%28CustomErrorType2%29%0A++++%7D+else+%7B%0A++++++++Ok%28b%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+-%3E+Result%3C%28%29%2C+ApplicationError%3E+%7B%0A++++let+result1+%3D+some_function_custom_error1%285%29%3F%3B%0A++++let+result2+%3D+some_function_custom_error2%285%29%3F%3B%0A++++%0A++++println%21%28%22result1%3A+%7B%7D%2C+result2%3A+%7B%7D%22%2C+result1%2C+result2%29%3B%0A++++%0A++++Ok%28%28%29%29%0A%7D%0A%0A)

ここまででRustが提供している標準ライブラリを使用してどのようにエラーハンドリングを行えば良いのか把握することができました。

## thiserror クレート

独自のエラー型を定義する際には、今まで見てきたように各種トレイトの実装など、多くのボイラープレートの記述が必要となります。アプリケーションが規模を拡大するにつれて、エラー型の管理が大変になることがあります。

`thiserror` クレートは、ボイラープレートの実装の手間を軽減し、失敗時に呼び出し元が選択した情報を正確に受け取ることを重視する際に利用できます。ライブラリなどの呼び出し元が多岐にわたり、失敗した原因をできるだけユーザーに伝えたい場合に適しています。

https://docs.rs/thiserror/latest/thiserror/

公式ページに掲載されている以下のサンプルコードをご覧ください。

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum DataStoreError {
    #[error("data store disconnected")]
    Disconnect(#[from] std::io::Error),
    #[error("the data for key `{0}` is not available")]
    Redaction(String),
    #[error("invalid header (expected {expected:?}, found {found:?})")]
    InvalidHeader {
        expected: String,
        found: String,
    },
    #[error("unknown data store error")]
    Unknown,
}
```

`thiserror` クレートが提供する各種アトリビュートを使用すれば、エラーを実装する際に必要であった `Debug` トレイトや `Display` トレイトの実装を自身で管理する必要がなく、上記の記述のみでエラーを定義できるようになります。

アトリビュートでさまざまな定義を行なっていますが、 [cargo-expand](https://github.com/dtolnay/cargo-expand) を利用してどのようなコードが展開されているのか確認してみましょう。

### #[error("...")]

`#[error("...")]` では `Display` トレイトに対してどのような実装を行うのかを指定することができ、今回では以下のようにタプルで指定した値を表示したり、指定した属性の値を `Debug` で出力するような設定が組み込まれていることがわかります。

```rust
impl std::fmt::Display for DataStoreError {
    fn fmt(&self, __formatter: &mut std::fmt::Formatter) -> std::fmt::Result {
        #[allow(unused_imports)]
        use thiserror::__private::{DisplayAsDisplay, PathAsDisplay};
        #[allow(unused_variables, deprecated, clippy::used_underscore_binding)]
        match self {
            DataStoreError::Disconnect(_0) => {
                __formatter.write_fmt(format_args!("data store disconnected"))
            }
            DataStoreError::Redaction(_0) => {
                __formatter
                    .write_fmt(
                        format_args!(
                            "the data for key `{0}` is not available", _0.as_display()
                        ),
                    )
            }
            DataStoreError::InvalidHeader { expected, found } => {
                __formatter
                    .write_fmt(
                        format_args!(
                            "invalid header (expected {0:?}, found {1:?})", expected,
                            found
                        ),
                    )
            }
            DataStoreError::Unknown {} => {
                __formatter.write_fmt(format_args!("unknown data store error"))
            }
        }
    }
}
```

このように `thiserror` クレートを利用することでエラー型を定義する時のボイラープレートを大幅に削減することができます。

### #[error(transparent)]

また `Display` の実装は他の型で既に実装されているものを `#[error(transparent)]` で再利用することが可能です。

通常は以下のように `#[error("...")]` を付与すると出力する文字列を調整することができます。

```rust
#[derive(Error, Debug)]
pub enum DataStoreError {
    #[error("data store disconnected")]
    Disconnect(#[from] std::io::Error),
}

impl std::fmt::Display for DataStoreError {
    fn fmt(&self, __formatter: &mut std::fmt::Formatter) -> std::fmt::Result {
        match self {
            DataStoreError::Disconnect(_0) => {
                __formatter.write_fmt(format_args!("data store disconnected"))
            },
            // ...
        }
    }
}
```

`#[error(transparent)]` を利用することで `Disconnect` が値として受け取ったものに対してそのまま `fmt` を呼び出してエラーメッセージの表示の機能を委譲していることがわかります。

```rust
#[derive(Error, Debug)]
pub enum DataStoreError {
    #[error(transparent)]
    Disconnect(#[from] std::io::Error),
}

impl std::fmt::Display for DataStoreError {
    fn fmt(&self, __formatter: &mut std::fmt::Formatter) -> std::fmt::Result {
        match self {
            DataStoreError::Disconnect(_0) => std::fmt::Display::fmt(_0, __formatter),
            // ...
        }
    }
}
```

このように既存の `Error` の実装ともうまく連携することが可能である。

### #[source] / #[from]

展開した内容をみると以下のように `Error` トレイトで定義されている `source` メソッドの実装が自動的に追加されていることがわかります。

```rust
impl std::error::Error for DataStoreError {
    fn source(&self) -> std::option::Option<&(dyn std::error::Error + 'static)> {
        use thiserror::__private::AsDynError;
        #[allow(deprecated)]
        match self {
            DataStoreError::Disconnect { 0: source, .. } => {
                std::option::Option::Some(source.as_dyn_error())
            }
            DataStoreError::Redaction { .. } => std::option::Option::None,
            DataStoreError::InvalidHeader { .. } => std::option::Option::None,
            DataStoreError::Unknown { .. } => std::option::Option::None,
        }
    }
}
```

`Error` トレイトで定義されている `source()` メソッドは `#[source]` 属性を有するフィールドを下位レベルのエラーとして指定し、エラーが発生した原因をより深ぼることが可能になります。

今回 `#[source]` 属性を指定していませんが、 `#[from]` 属性を付与すると `From` トレイトの実装だけではなく、暗黙的に `#[source]` と同じフィールドだと識別されます。

実際に以下のように指定した属性に対して `From` トレイトが実装されていることがわかります。

```rust
// #[derive(Error, Debug)]
// pub enum DataStoreError {
//    #[error("data store disconnected")]
//    Disconnect(#[from] std::io::Error), <- ここで定義したエラーを source で抽出する
//    // ... 
// }

impl std::convert::From<std::io::Error> for DataStoreError {
    #[allow(deprecated)]
    fn from(source: std::io::Error) -> Self {
        DataStoreError::Disconnect {
            0: source,
        }
    }
}
```

ここまでみてきたように `thiserror` クレートはRustの標準ライブラリの `Error` トレイトの実装を簡単に実装することが可能でき、ボイラープレート的な記述の手間を省くためのクレートです。

実際に `Error` を自作した時と比べると、以下の再現コードではかなりの行数が削減されていることがわかります。

[再現コード](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&code=use+thiserror%3A%3AError%3B+%2F%2F+1.0.40%0A%0A%23%5Bderive%28Error%2C+Debug%29%5D%0A%23%5Berror%28%22CustomErrorType1+Error%22%29%5D%0Apub+struct+CustomErrorType1%3B%0A%0A%23%5Bderive%28Error%2C+Debug%29%5D%0A%23%5Berror%28%22CustomErrorType2+Error%22%29%5D%0Apub+struct+CustomErrorType2%3B%0A%0A%23%5Bderive%28Error%2C+Debug%29%5D%0Apub+enum+ApplicationError+%7B%0A++++%23%5Berror%28transparent%29%5D%0A++++Type1%28%23%5Bfrom%5D+CustomErrorType1%29%2C%0A++++%23%5Berror%28transparent%29%5D%0A++++Type2%28%23%5Bfrom%5D+CustomErrorType2%29%2C%0A%7D%0A%0Afn+some_function_custom_error1%28a%3A+i32%29+-%3E+Result%3Ci32%2C+CustomErrorType1%3E+%7B%0A++++if+a+%3D%3D+0+%7B%0A++++++++Err%28CustomErrorType1%29%0A++++%7D+else+%7B%0A++++++++Ok%28a%29%0A++++%7D%0A%7D%0A%0Afn+some_function_custom_error2%28b%3A+i32%29+-%3E+Result%3Ci32%2C+CustomErrorType2%3E+%7B%0A++++if+b+%3E+10+%7B%0A++++++++Err%28CustomErrorType2%29%0A++++%7D+else+%7B%0A++++++++Ok%28b%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+-%3E+Result%3C%28%29%2C+ApplicationError%3E+%7B%0A++++%2F%2F+Display%E3%81%AE%E5%AE%9F%E8%A3%85%E3%82%92%E7%A2%BA%E8%AA%8D%E3%81%99%E3%82%8B%E3%81%9F%E3%82%81%E3%81%AB+map_err+%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%97%E3%81%A6%E6%A8%99%E6%BA%96%E5%87%BA%E5%8A%9B%E3%81%AB%E5%87%BA%E3%81%97%E3%81%A6%E3%81%84%E3%82%8B%0A++++%2F%2F+%E5%AE%9F%E9%9A%9B%E3%81%AB%E3%81%AF+some_function_custom_error1%280%29%3F%3B+%E3%81%A0%E3%81%91%E3%81%A7%E3%82%82%E5%8D%81%E5%88%86%0A++++let+result1+%3D+some_function_custom_error1%280%29.map_err%28%7Ce%7C+%7B%0A++++++++println%21%28%22%7B%7D%22%2C+e%29%3B%0A++++++++e%0A++++%7D%29%3F%3B%0A++++let+result2+%3D+some_function_custom_error2%285%29.map_err%28%7Ce%7C+%7B%0A++++++++println%21%28%22%7B%7D%22%2C+e%29%3B%0A++++++++e%0A++++%7D%29%3F%3B%0A++++%0A++++println%21%28%22result1%3A+%7B%7D%2C+result2%3A+%7B%7D%22%2C+result1%2C+result2%29%3B%0A++++%0A++++Ok%28%28%29%29%0A%7D)

## anyhow クレート

これまでの例でみてきたように、Rustの型安全性を利用することで、 `Result` 型を返却する関数などを作成する際にはコンパイルエラーが発生しないように型を定義する必要がありました。不特定多数のユーザーが利用するライブラリであれば、より厳密にエラーを管理することでユーザーに有用なフィードバックを提供することが可能ですが、自身が開発するアプリケーションでは厳密にエラーを管理することにかなりのコストが発生するかもしれません。

そういった状況の際には `anyhow` を利用することで `std::error::Error` トレイトを実装したそれぞれのエラーの違いを吸収することが可能です。

https://docs.rs/anyhow/latest/anyhow/

### 異なるエラー型の統一

先ほどまでのコードでは、以下のように関数をそれぞれ異なる `Err` を返却するように定義しており、関数の呼び出し元では `enum` で定義したエラーへの型変換を行うことでコンパイルエラーの発生を回避していました。

```rust
#[derive(Error, Debug)]
#[error("CustomErrorType1 Error")]
pub struct CustomErrorType1;

#[derive(Error, Debug)]
#[error("CustomErrorType2 Error")]
pub struct CustomErrorType2;

fn some_function_custom_error1(a: i32) -> Result<i32, CustomErrorType1> {
    if a == 0 { Err(CustomErrorType1) } else { Ok(a) }
}

fn some_function_custom_error2(b: i32) -> Result<i32, CustomErrorType2> {
    if b > 10 { Err(CustomErrorType2) } else { Ok(b) }
}
```

`anyhow` では以下のようなエラーを統一的に取り扱うための `Result` 型を提供しており、標準ライブラリの `Error` を実装している型の違いを吸収することが可能です。

```rust
// エラーの違いを吸収する
pub type Result<T, E = Error> = core::result::Result<T, E>;
```

実際に以下のように返却するエラーの型が異なる場合でもコンパイルエラーが発生することはありません。

```rust
// 以前は ApplicationError という全てのエラーの可能性を定義した Enum を指定していた
// anyhowがエラーの型の違いを吸収することで ? で伝播されるエラーの違いによるコンパイルエラーを防いでいる
fn main() -> anyhow::Result<()> {
    let result1 = some_function_custom_error1(0)?;
    let result2 = some_function_custom_error2(5)?;
    
    println!("result1: {}, result2: {}", result1, result2);
    
    Ok(())
}
```

[再現コード](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&code=use+thiserror%3A%3AError%3B+%2F%2F+1.0.40%0A%0A%23%5Bderive%28Error%2C+Debug%29%5D%0A%23%5Berror%28%22CustomErrorType1+Error%22%29%5D%0Apub+struct+CustomErrorType1%3B%0A%0A%23%5Bderive%28Error%2C+Debug%29%5D%0A%23%5Berror%28%22CustomErrorType2+Error%22%29%5D%0Apub+struct+CustomErrorType2%3B%0A%0Afn+some_function_custom_error1%28a%3A+i32%29+-%3E+Result%3Ci32%2C+CustomErrorType1%3E+%7B%0A++++if+a+%3D%3D+0+%7B+Err%28CustomErrorType1%29+%7D+else+%7B+Ok%28a%29+%7D%0A%7D%0A%0Afn+some_function_custom_error2%28b%3A+i32%29+-%3E+Result%3Ci32%2C+CustomErrorType2%3E+%7B%0A++++if+b+%3E+10+%7B+Err%28CustomErrorType2%29+%7D+else+%7B+Ok%28b%29+%7D%0A%7D%0A%0Afn+main%28%29+-%3E+anyhow%3A%3AResult%3C%28%29%3E+%7B%0A++++let+result1+%3D+some_function_custom_error1%280%29%3F%3B%0A++++let+result2+%3D+some_function_custom_error2%285%29%3F%3B%0A++++%0A++++println%21%28%22result1%3A+%7B%7D%2C+result2%3A+%7B%7D%22%2C+result1%2C+result2%29%3B%0A++++%0A++++Ok%28%28%29%29%0A%7D)

エラーの違いを吸収することができるようになりましたが、anyhowを多用すると、呼び出し元でどの種類のエラーが発生するか把握することが困難になり、型による明確な宣言の利点が失われてしまうことに注意が必要です。

実際のアプリケーション開発では、下層に定義されているドメインロジックなどでは、 `thiserror` を使用してより精密なエラーを返すようにすることが設計し、一方で、ドメインロジックの組み合わせにより表現される上層の部分、例えばユースケース層などでは、エラー型の違いを吸収できるように `anyhow` を利用するといった使い方が望ましいのではないかと思います。

具体的には [Domain Modeling Made Functional](https://amzn.asia/d/9EwPafU) の第10章で言及されているようなエラー設計のイメージです。

### 簡易的なエラーの定義

`anyhow` はエラー型の違いの吸収以外にもさまざまなことを行うことができるが、その1つとして簡易的にエラーを生成することができる。

例えば今までのサンプルコードでは以下のように各関数が返すエラーを厳密に定義していたが、プロジェクト初期段階であったりプロトタイプ開発ではそこでま厳密なはエラーの設計が必要ではないかもしれない。

```rust
use thiserror::Error;

#[derive(Error, Debug)]
#[error("CustomErrorType1 Error")]
pub struct CustomErrorType1;

#[derive(Error, Debug)]
#[error("CustomErrorType2 Error")]
pub struct CustomErrorType2;

fn some_function_custom_error1(a: i32) -> Result<i32, CustomErrorType1> {
    if a == 0 { Err(CustomErrorType1) } else { Ok(a) }
}

fn some_function_custom_error2(b: i32) -> Result<i32, CustomErrorType2> {
    if b > 10 { Err(CustomErrorType2) } else { Ok(b) }
}
```

そのような場合には `anyhow!` マクロを使用して個別にエラー型を定義することなく、以下のように成功と失敗の表現をすることが可能である。

https://docs.rs/anyhow/latest/anyhow/macro.anyhow.html

```rust
use anyhow::{Result, anyhow}; // 1.0.71

fn some_function_custom_error1(a: i32) -> Result<i32> {
    if a == 0 { 
        Err(anyhow!("Custom Error 1"))
    } else { 
        Ok(a)
    }
}

fn some_function_custom_error2(b: i32) -> Result<i32> {
    if b == 0 { 
        Err(anyhow!("Custom Error 2"))
    } else { 
        Ok(b)
    }
}

fn main() -> anyhow::Result<()> {
    let result1 = some_function_custom_error1(0)?;
    let result2 = some_function_custom_error2(5)?;
    
    println!("result1: {}, result2: {}", result1, result2);
    
    Ok(())
}
```

[再現コード](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&code=use+anyhow%3A%3A%7BResult%2C+anyhow%7D%3B+%2F%2F+1.0.71%0A%0Afn+some_function_custom_error1%28a%3A+i32%29+-%3E+Result%3Ci32%3E+%7B%0A++++if+a+%3D%3D+0+%7B+%0A++++++++Err%28anyhow%21%28%22Custom+Error+1%22%29%29%0A++++%7D+else+%7B+%0A++++++++Ok%28a%29%0A++++%7D%0A%7D%0A%0Afn+some_function_custom_error2%28b%3A+i32%29+-%3E+Result%3Ci32%3E+%7B%0A++++if+b+%3D%3D+0+%7B+%0A++++++++Err%28anyhow%21%28%22Custom+Error+2%22%29%29%0A++++%7D+else+%7B+%0A++++++++Ok%28b%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+-%3E+anyhow%3A%3AResult%3C%28%29%3E+%7B%0A++++let+result1+%3D+some_function_custom_error1%280%29%3F%3B%0A++++let+result2+%3D+some_function_custom_error2%285%29%3F%3B%0A++++%0A++++println%21%28%22result1%3A+%7B%7D%2C+result2%3A+%7B%7D%22%2C+result1%2C+result2%29%3B%0A++++%0A++++Ok%28%28%29%29%0A%7D)

この `anyhow!` マクロ内では下記の実装が呼び出されおり、メソッド内部で `anyhow` クレートが自身で定義しているエラー型を生成して返却していることがわかります。

https://github.com/dtolnay/anyhow/blob/8b4fc43429fd9a034649e0f919c646ec6626c4c7/src/lib.rs#L658-L674

`anyhow!` マクロ以外にも `bail!` マクロや `ensure!` マクロが定義されており、より簡易的にエラーを生成することができます。

```rust
fn some_function_custom_error1(a: i32) -> Result<i32> {
    if a == 0 { 
        Err(anyhow!("Custom Error 1"))
    } else { 
        Ok(a)
    }
}

fn some_function_custom_error2(b: i32) -> Result<i32> {
    if b == 0 { 
        // bail! マクロを使用すれば文字列だけを指定すれば良い
        // Err(anyhow!("Custom Error 2"))
        bail!("Custom Error 2")
    } else { 
        Ok(b)
    }
}

fn some_function_custom_error3(c: i32) -> Result<i32> {
    // ensure! マクロでは条件も一緒に指定することが可能である
    // assert! マクロに近い感覚
    ensure!(c > 0, "Custom Error 3");
    
    Ok(c)
}
```

[再現コード](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&code=use+anyhow%3A%3A%7BResult%2C+anyhow%2C+bail%2C+ensure%7D%3B+%2F%2F+1.0.71%0A%0Afn+some_function_custom_error1%28a%3A+i32%29+-%3E+Result%3Ci32%3E+%7B%0A++++if+a+%3D%3D+0+%7B+%0A++++++++Err%28anyhow%21%28%22Custom+Error+1%22%29%29%0A++++%7D+else+%7B+%0A++++++++Ok%28a%29%0A++++%7D%0A%7D%0A%0Afn+some_function_custom_error2%28b%3A+i32%29+-%3E+Result%3Ci32%3E+%7B%0A++++if+b+%3D%3D+0+%7B+%0A++++++++%2F%2F+bail%21+%E3%83%9E%E3%82%AF%E3%83%AD%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%99%E3%82%8B%E3%81%93%E3%81%A8%E3%81%A7%E6%96%87%E5%AD%97%E5%88%97%E3%81%A0%E3%81%91%E3%82%92%E6%8C%87%E5%AE%9A%E3%81%99%E3%82%8C%E3%81%B0%E8%89%AF%E3%81%84%E7%8A%B6%E6%85%8B%E3%81%A8%E3%81%AA%E3%82%8B%0A++++++++%2F%2F+Err%28anyhow%21%28%22Custom+Error+2%22%29%29%0A++++++++bail%21%28%22Custom+Error+2%22%29%0A++++%7D+else+%7B+%0A++++++++Ok%28b%29%0A++++%7D%0A%7D%0A%0Afn+some_function_custom_error3%28c%3A+i32%29+-%3E+Result%3Ci32%3E+%7B%0A++++%2F%2F+ensure%21+%E3%83%9E%E3%82%AF%E3%83%AD%E3%81%A7%E3%81%AF%E6%9D%A1%E4%BB%B6%E3%82%82%E4%B8%80%E7%B7%92%E3%81%AB%E6%8C%87%E5%AE%9A%E3%81%99%E3%82%8B%E3%81%93%E3%81%A8%E3%81%8C%E5%8F%AF%E8%83%BD%E3%81%A7%E3%81%82%E3%82%8B%0A++++%2F%2F+assert%21+%E3%83%9E%E3%82%AF%E3%83%AD%E3%81%AB%E8%BF%91%E3%81%84%E6%84%9F%E8%A6%9A%0A++++ensure%21%28c+%3E+0%2C+%22Custom+Error+3%22%29%3B%0A++++%0A++++Ok%28c%29%0A%7D%0A%0Afn+main%28%29+-%3E+anyhow%3A%3AResult%3C%28%29%3E+%7B%0A++++let+result1+%3D+some_function_custom_error1%282%29%3F%3B%0A++++let+result2+%3D+some_function_custom_error2%285%29%3F%3B%0A++++let+result3+%3D+some_function_custom_error3%28-2%29%3F%3B%0A++++%0A++++println%21%28%22%7Bresult1%7D%2C+%7Bresult2%7D%2C+%7Bresult3%7D%22%29%3B%0A++++%0A++++Ok%28%28%29%29%0A%7D)

### エラーコンテキスト情報の追加

`anyhow!` はエラーのコンテキスト情報を追加することが可能であり、エラーの原因をより特定しやすいように追加の情報を提供したり、追加したコンテキスト情報を伝播させることでエラーメッセージをより詳細にすることができます。

https://docs.rs/anyhow/latest/anyhow/trait.Context.html

例えば以下の実装でエラーのコンテキスト情報をどのように追加するのか考えます。

```rust
use thiserror::Error; // 1.0.40
use anyhow::Result; // 1.0.71

#[derive(Error, Debug)]
#[error("CustomErrorType1 Error")]
pub struct CustomErrorType1;

fn some_function_custom_error(a: i32) -> Result<i32, CustomErrorType1> {
    if a == 0 {
        Err(CustomErrorType1)
    } else {
        Ok(a)
    }
}

fn main() -> Result<()> {
    let input = 0;
    let result = some_function_custom_error(input)?;
    
    println!("result: {}", result);
    Ok(())
}
```

この関数を実行すると以下のようなメッセージが表示されますが、どのような引数を渡した結果、このメッセージが表示されてしまったのか把握することができません。

```bash
Error: CustomErrorType1 Error
```

自身で管理している関数であれば元のエラー型の定義を修正すれば解決できますが、外部クレートが提供しているエラー型などであればエラーメッセージを変更することは面倒になります。そのような場合に `anyhow::Context` を利用することで追加のメッセージを指定できます。

```rust
fn main() -> Result<()> {
    let input = 0;
    let result = some_function_custom_error(input)
        // 追加の情報を指定することができる
        .with_context(|| format!("Failed to execute with: {}", input))?;
    
    println!("result: {}", result);
    Ok(())
}
```

この関数を実行すると以下のメッセージが表示され、元のメッセージよりもさらに詳細な情報を追加できていることがわかります。

```bash
Error: Failed to execute with: 0

Caused by:
    CustomErrorType1 Error
```

[再現コード](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&code=use+thiserror%3A%3AError%3B+%2F%2F+1.0.40%0Ause+anyhow%3A%3A%7BResult%2C+Context%7D%3B+%2F%2F+1.0.71%0A%0A%23%5Bderive%28Error%2C+Debug%29%5D%0A%23%5Berror%28%22CustomErrorType1+Error%22%29%5D%0Apub+struct+CustomErrorType1%3B%0A%0Afn+some_function_custom_error%28a%3A+i32%29+-%3E+Result%3Ci32%2C+CustomErrorType1%3E+%7B%0A++++if+a+%3D%3D+0+%7B%0A++++++++Err%28CustomErrorType1%29%0A++++%7D+else+%7B%0A++++++++Ok%28a%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+-%3E+Result%3C%28%29%3E+%7B%0A++++let+input+%3D+0%3B%0A++++let+result+%3D+some_function_custom_error%28input%29%0A++++++++.with_context%28%7C%7C+format%21%28%22Failed+to+execute+with%3A+%7B%7D%22%2C+input%29%29%3F%3B%0A++++%0A++++println%21%28%22result%3A+%7B%7D%22%2C+result%29%3B%0A++++Ok%28%28%29%29%0A%7D)

ここまでみてきたように `anyhow` クレートは、Rustの型システムによる厳密なエラーハンドリングの要求を緩めることで、複数のエラー型が混在する状況をより柔軟に取り扱うことができ、またその柔軟性を活かしてより詳細なエラー情報の追加などが可能です。

ただし、どのようなエラー型も統一的に取り扱える都合上、型安全性は下がってしまうため導入は慎重に決めたほうが良さそうに感じます。

## 感想

私自身Rustの初学者であり、実際のプロジェクトでの使用経験もなく、エラーハンドリングに関するベストプラクティスが分からない状況でしたが、今回標準ライブラリを使ったエラー型の定義方法や各種クレートの利用方法を調査したことでかなり雰囲気を掴むことができました。

今回調査することができていない [error-stack](https://docs.rs/error-stack/latest/error_stack/) や [eyre](https://docs.rs/eyre/latest/eyre/) に関しても、時間があれば別記事でまとめてみようかなと思います。

本記事を執筆するにあたり、公式ドキュメントの確認とサンプルコードで理解度をチェックするというアプローチを取りましたが、ユーティリティトレイトの理解が曖昧だったり、ベストプラクティスに関する情報が不足していたため、ChatGPT に質問しながら進めることができました。おかげで、エラーハンドリングに関する理解が大幅に向上したと感じています。ChatGPT様様です！

## 参考資料

- [Rust/Anyhow の Tips](https://zenn.dev/yukinarit/articles/b39cd42820f29e)
- [Rust エラー処理 2020](https://cha-shu00.hatenablog.com/entry/2020/12/08/060000)
