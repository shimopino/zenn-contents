---
title: "Rustにおけるエラーハンドリング"
emoji: "🦀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
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

エラーメッセージでは、`From` トレイトが実装されていないと言われています。実は、シンタックスシュガーである `?` を使うと、型推論に基づいて暗黙的に [`From`]((https://doc.rust-lang.org/std/convert/trait.From.html)) トレイトの実装を呼び出しています。

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

```rs
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
