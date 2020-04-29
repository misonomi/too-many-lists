# Peek

前回実装する気もなかった機能はpeek[^1]です．これをやっていきましょう．やること
といえば，リストのheadがあるときその参照を返せばいいだけです．簡単そうですね．
やってみましょう：

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.map(|node| {
        &node.elem
    })
}
```


```text
> cargo build

error[E0515]: cannot return reference to local data `node.elem`
  --> src/second.rs:37:13
   |
37 |             &node.elem
   |             ^^^^^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:36:9
   |
36 |         self.head.map(|node| {
   |         ^^^^^^^^^ cannot move out of borrowed content


```

ハァ〜今度はなんだよ？

mapは`self`の値を取ってしまいますから，Optionの外に要素をムーブしてしまいます．
前回は`take`した直後だったのでそれでよかったのですが，今回は値を取っておきたいので
だめです．これに対処する*正しい*方法はOptionが持つ`as_ref`メソッドを使うことです．
`as_ref`のシグネチャはこんな感じです：

```rust ,ignore
impl<T> Option<T> {
    pub fn as_ref(&self) -> Option<&T>;
}
```

このメソッドはT型についてのOptionを，T型の参照についてのOptionに降格させます．
これをmatchで自前実装することもできますが*嫌です*ね．それをやろうとすると
Optionを剥がして詰め替えることをしなくてはいけません．幸いなことにそれは`.`
演算子がやってくれます．


```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

```text
cargo build

    Finished dev [unoptimized + debuginfo] target(s) in 0.32s
```

はいバッチリ

これの*可変*バージョンを`as_mut`で作ることもできます：

```rust ,ignore
pub fn peek_mut(&mut self) -> Option<&mut T> {
    self.head.as_mut().map(|node| {
        &mut node.elem
    })
}
```

```text
lists::cargo build

```

楽勝か？

テストも忘れず書きましょう：

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

いいですね．しかし，これでは`peek_mut`で取った値が本当に可変かどうか
わかりませんよね？値が可変でも誰も変更しなかったら，可変性をテストしたと
本当に言えるでしょうか？返り値の`Option<&mut T>`に`map`を使って全然違う値を
突っ込んでみましょう：

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
    list.peek_mut().map(|&mut value| {
        value = 42
    });

    assert_eq!(list.peek(), Some(&42));
    assert_eq!(list.pop(), Some(42));
}
```

```text
> cargo test

error[E0384]: cannot assign twice to immutable variable `value`
   --> src/second.rs:100:13
    |
99  |         list.peek_mut().map(|&mut value| {
    |                                   -----
    |                                   |
    |                                   first assignment to `value`
    |                                   help: make this binding mutable: `mut value`
100 |             value = 42
    |             ^^^^^^^^^^ cannot assign twice to immutable variable          ^~~~~
```

コンパイラは`value`が不変だと言って怒っています．でも明らかに`&mut value`って
書いてますよね．どゆこと？実はこの書き方はクロージャの引数が可変参照であることを
指定していることにならないのです．そうではなく，引数に対してマッチするパターンを
指定していることになります．`|&mut value|`は「この引数は可変参照だけど，こいつの
値をコピーして`value`に入れてね」という意味になります．`|value|`と書くことで
`value`の型を`&mut i32`にでき，headを変更できるようになります：

```rust ,ignore
    #[test]
    fn peek() {
        let mut list = List::new();
        assert_eq!(list.peek(), None);
        assert_eq!(list.peek_mut(), None);
        list.push(1); list.push(2); list.push(3);

        assert_eq!(list.peek(), Some(&3));
        assert_eq!(list.peek_mut(), Some(&mut 3));

        list.peek_mut().map(|value| {
            *value = 42
        });

        assert_eq!(list.peek(), Some(&42));
        assert_eq!(list.pop(), Some(42));
    }
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

かなり良くなりましたね！

[^1]: 訳注：スタックの一番上の要素をpopせずに取得する（覗き見る）機能
