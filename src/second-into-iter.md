# IntoIter

Rustでは要素の集まりをイテレートするときには*Iterator*トレイトを使います．
このトレイトは`Drop`よりも若干複雑です：

```rust ,ignore
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

ここで`type Item`というブロックが新しく出てきました．これはIteratorを実装する
型には*関連付けられた型*があることを表しています．この場合`next`したときに
出てくる型のことです．

nextが`Option<Self::Item>`を返す理由は，このメソッドが`has_next`と`get_next`の
役割を併せ持つからです．もし次の値があれば`Some(value)`を返し，なければ`None`を
返すわけです．これによって，より人間工学的で安全なAPIを構築しつつ`has_next`と
`get_next`を別々に実装した際の冗長さを回避できるのです．うまい手ですね！

悲しいことにRustには（まだ）`yield`文のようなものはありません．なのでそれに
相当する処理を自力で実装する必要があります．さらに，実はイテレータには
3つの種類があり，それぞれを実装しなくてはいけません：

* IntoIter - `T`
* IterMut - `&mut T`
* Iter - `&T`

実はIntoIterを実装するために必要なものはすでに揃っています．ただ`pop`を
呼びまくればいいだけです．IntoIterをListのラッパーとして，こんな感じに
実装できます：


```rust ,ignore
// タプル構造体はstructの変化形の一つです
// 他の型のラッパーを作るときに便利
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        // タプル構造体のフィールドには数字でアクセス
        self.0.pop()
    }
}
```

そしてテストを書きましょう：

```rust ,ignore
#[test]
fn into_iter() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.into_iter();
    assert_eq!(iter.next(), Some(3));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next(), Some(1));
    assert_eq!(iter.next(), None);
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 4 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured

```

いい感じですね！
