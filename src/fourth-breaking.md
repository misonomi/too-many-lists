# 崩す

`pop_front`は`push_front`と同じ処理を逆に行えばよさそうです。やってみましょう：

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    // need to take the old head, ensuring it's -2
    self.head.take().map(|old_head| {                         // -1 old
        match old_head.borrow_mut().next.take() {
            Some(new_head) => {                               // -1 new
                // not emptying list
                new_head.borrow_mut().prev.take();            // -1 old
                self.head = Some(new_head);                   // +1 new
                // total: -2 old, +0 new
            }
            None => {
                // emptying list
                self.tail.take();                             // -1 old
                // total: -2 old, (no new)
            }
        }
        old_head.elem
    })
}
```

```text
> cargo build

error[E0609]: no field `elem` on type `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>`
  --> src/fourth.rs:64:22
   |
64 |             old_head.elem
   |                      ^^^^ unknown field
```

アーッ。*RefCell*か。`borrow_mut`しなきゃいけないみたいですね...

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    self.head.take().map(|old_head| {
        match old_head.borrow_mut().next.take() {
            Some(new_head) => {
                new_head.borrow_mut().prev.take();
                self.head = Some(new_head);
            }
            None => {
                self.tail.take();
            }
        }
        old_head.borrow_mut().elem
    })
}
```

```text
cargo build

error[E0507]: cannot move out of borrowed content
  --> src/fourth.rs:64:13
   |
64 |             old_head.borrow_mut().elem
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^ cannot move out of borrowed content
```

*ため息*

> cannot move out of borrowed content

フム...私達はBoxに*めちゃくちゃ*甘えていたみたいですね。`borrow_mut`は
`&mut Node<T>`しか返さないのでムーブすることができません！

`RefCell<T>`から`T`を返す何かが必要です。[ドキュメント][refcell]を見てみましょう：

> `fn into_inner(self) -> T`
>
> Consumes the RefCell, returning the wrapped value.

これっぽいですね！

```rust ,ignore
old_head.into_inner().elem
```

```text
> cargo build

error[E0507]: cannot move out of an `Rc`
  --> src/fourth.rs:64:13
   |
64 |             old_head.into_inner().elem
   |             ^^^^^^^^ cannot move out of an `Rc`
```

あークソが。`into_inner`はRefCellをムーブしようとしますが`Rc`の中にあるので
それはできません。前章でみた通り`Rc<T>`からは中身の共有参照しか取れません。
でもそりゃそうですよね。参照カウンタがついた値は共有されるためのものなのですから！

この問題を解決するためには、前回と同様`Rc::try_unwrap`を使います。参照カウントが
1ならRcから中身を取り出すメソッドです。

```rust ,ignore
Rc::try_unwrap(old_head).unwrap().into_inner().elem
```

`Rc::try_unwrap`は`Result<T, Rc<T>>`を返します。Resultは`Option`を汎化したような
もので、`None`にデータを入れられるようになったものです。この場合`None`
の代わりに返されるものは剥こうとした`Rc`です。今回は失敗の場合を考えないので
（プログラムを正しく書けていれば成功する*はず*です）単に`unwrap`します。

とにかく、次はどんなコンパイルエラーが出るか見てみましょう（準備はよろしいですか？
1個はあるはずです）。

```text
> cargo build

error[E0599]: no method named `unwrap` found for type `std::result::Result<std::cell::RefCell<fourth::Node<T>>, std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>>` in the current scope
  --> src/fourth.rs:64:38
   |
64 |             Rc::try_unwrap(old_head).unwrap().into_inner().elem
   |                                      ^^^^^^
   |
   = note: the method `unwrap` exists but the following trait bounds were not satisfied:
           `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>> : std::fmt::Debug`
```

ぐえー。Resultを`unwrap`するにはエラーとして渡される型をデバッグ出力できる必要が
あります。`RefCell<T>`は`T`が`Debug`を実装する場合のみデバッグ出力できますが、
`Node`は`Debug`を実装していませんでした。

Debugを実装するより、`ok`でResultをOptionに変換してしまう方がいいでしょう：

```rust ,ignore
Rc::try_unwrap(old_head).ok().unwrap().into_inner().elem
```

頼む！！！

```text
cargo build

```

や　っ　た　ぜ

*フゥ*

やりました。

`push`と`pop`ができました。

はじめに作ったリストと実装した機能が同じなので、そっちからテストコードを
盗んできてテストしましょう：

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop_front(), None);

        // Populate list
        list.push_front(1);
        list.push_front(2);
        list.push_front(3);

        // Check normal removal
        assert_eq!(list.pop_front(), Some(3));
        assert_eq!(list.pop_front(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push_front(4);
        list.push_front(5);

        // Check normal removal
        assert_eq!(list.pop_front(), Some(5));
        assert_eq!(list.pop_front(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop_front(), Some(1));
        assert_eq!(list.pop_front(), None);
    }
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 9 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::basics ... ok
test fifth::test::iter_mut ... ok
test third::test::basics ... ok
test second::test::iter ... ok
test third::test::iter ... ok
test second::test::into_iter ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured

```

*はいバッチリ*

これでリストから要素を取り除けるのでDropの実装に移れます。今回のDropはすこし
おもしろいことになっています。というのも、これまでDropを実装するに際し
Now that we can properly remove things from the list, we can implement Drop.
Drop is a little more conceptually interesting this time around. Where
previously we bothered to implement Drop for our stacks just to avoid unbounded
recursion, now we need to implement Drop to get *anything* to happen at all.

`Rc` can't deal with cycles. If there's a cycle, everything will keep everything
else alive. A doubly-linked list, as it turns out, is just a big chain of tiny
cycles! So when we drop our list, the two end nodes will have their refcounts
decremented down to 1... and then nothing else will happen. Well, if our list
contains exactly one node we're good to go. But ideally a list should work right
if it contains multiple elements. Maybe that's just me.

As we saw, removing elements was a bit painful. So the easiest thing for us to
do is just `pop` until we get None:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        while self.pop_front().is_some() {}
    }
}
```

```text
cargo build

```

(We actually could have done this with our mutable stacks, but shortcuts are for
people who understand things!)

We could look at implementing the `_back` versions of `push` and `pop`, but
they're just copy-paste jobs which we'll defer to later in the chapter. For now
let's look at more interesting things!


[refcell]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
[multirust]: https://github.com/brson/multirust
[downloads]: https://www.rust-lang.org/install.html
