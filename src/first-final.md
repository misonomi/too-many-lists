# 最終コード

はい，このコードを書くまでに6000語（翻訳前）かかりました：

```rust
use std::mem;

pub struct List {
    head: Link,
}

enum Link {
    Empty,
    More(Box<Node>),
}

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: Link::Empty }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: mem::replace(&mut self.head, Link::Empty),
        });

        self.head = Link::More(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match mem::replace(&mut self.head, Link::Empty) {
            Link::Empty => None,
            Link::More(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, Link::Empty);

        while let Link::More(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, Link::Empty);
        }
    }
}

#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let mut list = List::new();

        // 空のリストが動くことを確認
        assert_eq!(list.pop(), None);

        // リストの要素をつめる
        list.push(1);
        list.push(2);
        list.push(3);

        // 普通に要素を削除してみる
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(2));

        // 何も壊れてないことを確認するためにもう一回push
        list.push(4);
        list.push(5);

        // 普通に要素を削除してみる
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), Some(4));

        // リストを出し切ったとき
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), None);
    }
}
```

マジかよ，80行しかないし半分はテストだぞ！最初に長い章になりますと言ったとおりに
なりましたね．
