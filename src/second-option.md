# Optionを使う

特別目ざとい人は私達が劣化Optionを再発明したことに気づいたかもしれません：

```rust ,ignore
enum Link {
    Empty,
    More(Box<Node>),
}
```

Linkは`Option<Box<Node>>`と同じことです。かといって`Option<Box<Node>>`を
どこにでも書くのは気が引けますし、`pop`の戻り値とは違いLinkは外部から
見ることはできないので、これをLinkと呼び続けることは問題なさそうです。
しかし、Optionには*超いい*メソッドがいくつもあり、Optionを使えば
（私達がやったように）それらを自力で実装することもありません。なので
全部Optionを使いましょう。まずは愚直に全部のMoreとEmptyをSomeとNoneに
書き換えていきましょう：

```rust ,ignore
use std::mem;

pub struct List {
    head: Link,
}

// yay type aliases!
type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: mem::replace(&mut self.head, None),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match mem::replace(&mut self.head, None) {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, None);
        while let Some(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, None);
        }
    }
}
```

これは若干良くなった程度ですが、さらなる良さがOptionのメソッドを使うことで
得られます。

まず`mem::replace(&mut option, None)`はめっちゃよくある形で、
あまりにもよくあるのでOptionにはそのための`take`というメソッドがあります。

```rust ,ignore
pub struct List {
    head: Link,
}

type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match self.head.take() {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

次に`match option { None => None, Some(x) => Some(y) }`もめっちゃよくある形で、
あまりにもよくあるので`map`というメソッドがあります。`map`には`Some(x)`の`x`
から`Some(y)`の`y`を作る関数を渡す必要があります。`fn`で関数を宣言して渡す
のもいいですが、*インラインで*書くほうがいいでしょう。

そのためには*クロージャ*を使います。クロージャは、クロージャ外のローカル変数を
参照することができる無名関数です。この強力な機能によって、条件に依存する処理を
超簡単に書くことができます。`match`を使っているのは`pop`だけですね。書き換えて
みましょう：

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    self.head.take().map(|node| {
        self.head = node.next;
        node.elem
    })
}
```

アーイイ。今回の変更で何もぶっ壊れてないことを確認しましょう：

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

いいですね！次はコードの*挙動*を改善していきましょう！
