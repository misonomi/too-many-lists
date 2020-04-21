# 基本

Rustの基本についてはすでにだいぶ分かっていますので、簡単なコードを書き直す
ことは造作もありません。

コンストラクタについてはコピペで十分です：

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }
}
```

今回は`push`と`pop`は意味をなしません。代わりにだいたい同じ機能の`append`
と`tail`を実装しましょう。

リストに要素を追加する`append`から始めましょう。この関数はリストと要素を
一つずつとり、それらをくっつけたリストを返します。可変なリストのとき同様
新しいノードを作りたいわけですが、そのノードの`next`は元のリストを指している
必要があります。目新しいことは、どうやってもとのリストに変更を加えずに
`next`を取得するかということくらいです。

その答えはCloneトレイトです。Cloneはほぼ全ての型に対して実装されており、
元の参照から切り離された「元オブジェクトとだいたい同じもの」を得る手段を
提供しています。C++のコピーコンストラクタに似ていますが、暗黙的に呼ばれること
がない点が異なります。

特にRcはCloneを、参照カウントを増やすために使用しています。なのでRcを使う場合、
Boxをサブリストに移動させる代わりに、単に古いリストのheadをCloneすることになります。
headをmatchで取り出すことも必要ありません。なぜならOptionに実装されているCloneが
まさに私達がやりたいことをやってくれるからです。

よし、じゃあやってみましょう:

```rust ,ignore
pub fn append(&self, elem: T) -> List<T> {
    List { head: Some(Rc::new(Node {
        elem: elem,
        next: self.head.clone(),
    }))}
}
```

```text
> cargo build

warning: field is never used: `elem`
  --> src/third.rs:10:5
   |
10 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `next`
  --> src/third.rs:11:5
   |
11 |     next: Link<T>,
   |     ^^^^^^^^^^^^^
```

うわ。Rustはフィールドを使っているかどうかにめちゃくちゃこだわりますね。この
リスを使う人が

Wow, Rust is really hard-nosed about actually using fields. It can tell no
consumer can ever actually observe the use of these fields! Still, we seem good
so far.

`tail` is the logical inverse of this operation. It takes a list and returns the
whole list with the first element removed. All that is is cloning the *second*
element in the list (if it exists). Let's try this:

```rust ,ignore
pub fn tail(&self) -> List<T> {
    List { head: self.head.as_ref().map(|node| node.next.clone()) }
}
```

```text
cargo build

error[E0308]: mismatched types
  --> src/third.rs:27:22
   |
27 |         List { head: self.head.as_ref().map(|node| node.next.clone()) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `std::rc::Rc`, found enum `std::option::Option`
   |
   = note: expected type `std::option::Option<std::rc::Rc<_>>`
              found type `std::option::Option<std::option::Option<std::rc::Rc<_>>>`
```

Hrm, we messed up. `map` expects us to return a Y, but here we're returning an
`Option<Y>`. Thankfully, this is another common Option pattern, and we can just
use `and_then` to let us return an Option.

```rust ,ignore
pub fn tail(&self) -> List<T> {
    List { head: self.head.as_ref().and_then(|node| node.next.clone()) }
}
```

```text
> cargo build

```

Great.

Now that we have `tail`, we should probably provide `head`, which returns a
reference to the first element. That's just `peek` from the mutable list:

```rust ,ignore
pub fn head(&self) -> Option<&T> {
    self.head.as_ref().map(|node| &node.elem )
}
```

```text
> cargo build

```

Nice.

That's enough functionality that we can test it:


```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let list = List::new();
        assert_eq!(list.head(), None);

        let list = list.append(1).append(2).append(3);
        assert_eq!(list.head(), Some(&3));

        let list = list.tail();
        assert_eq!(list.head(), Some(&2));

        let list = list.tail();
        assert_eq!(list.head(), Some(&1));

        let list = list.tail();
        assert_eq!(list.head(), None);

        // Make sure empty tail works
        let list = list.tail();
        assert_eq!(list.head(), None);

    }
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 5 tests
test first::test::basics ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured

```

Perfect!

Iter is also identical to how it was for our mutable list:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_ref().map(|node| &**node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_ref().map(|node| &**node);
            &node.elem
        })
    }
}
```

```rust ,ignore
#[test]
fn iter() {
    let list = List::new().append(1).append(2).append(3);

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&1));
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 7 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured

```

Who ever said dynamic typing was easier?

(chumps did)

Note that we can't implement IntoIter or IterMut for this type. We only have
shared access to elements.
