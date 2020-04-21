# テスト

さてさて，`push`と`pop`が書けたので実際にスタックをテストしてみましょう！
RustとCargoはテストを第一級の機能としてサポートしているので，テストを書くのは
超カンタンです．関数を書いて`#[test]`アノテーションをつければよいだけです．

一般にRust界隈ではテストコードをテストされるコードの横に書く慣習があります．
しかし，通常は名前の衝突を避けるためテストのための名前空間を用意します．`first.rs`
を`lib.rs`で使えるように`mod`を使ったのと同じように，`mod`キーワードを使えば
新しいファイルを作るのと同じようなことを*インラインで*することができます：


```rust ,ignore
// in first.rs

mod test {
    #[test]
    fn basics() {
        // TODO
    }
}
```

`cargo test`で実行します．

```text
> cargo test
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
    Finished dev [unoptimized + debuginfo] target(s) in 1.00s
     Running /Users/ABeingessner/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```

イエーイ　私達の何もしないテストは無事通りました！これを何もしなくないように
していきましょう．そのためには`assert_eq!`マクロを使います．これは不思議な
魔法のたぐいではなく，与えた2つの値を比較してそれらが違えばパニックするだけです．
キョドることでテストの失敗を知らせるのです！

```rust ,ignore
mod test {
    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop(), None);

        // Populate list
        list.push(1);
        list.push(2);
        list.push(3);

        // Check normal removal
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), None);
    }
}
```

```text
> cargo test

error[E0433]: failed to resolve: use of undeclared type or module `List`
  --> src/first.rs:43:24
   |
43 |         let mut list = List::new();
   |                        ^^^^ use of undeclared type or module `List`


```

おおっと！`mod`で新しいモジュールを切ったのでListを明示的に宣言しなくてはいけません．

```rust ,ignore
mod test {
    use super::List;
    // everything else the same
}
```

```text
> cargo test

warning: unused import: `super::List`
  --> src/first.rs:45:9
   |
45 |     use super::List;
   |         ^^^^^^^^^^^
   |
   = note: #[warn(unused_imports)] on by default

    Finished dev [unoptimized + debuginfo] target(s) in 0.43s
     Running /Users/ABeingessner/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```

やりい！

でもこの警告は何でしょう...？Listはどう見ても使ってるだろ！

...でもテストのためだけに，ですね！コンパイラを黙らせるために（そしてこの
パッケージを使う人にわかりやすいように）`test`モジュール全体をテストのとき
だけにコンパイルされるようにしましょう．


```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    // everything else the same
}
```

これでテストは完成です！
