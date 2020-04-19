# あらゆるものをジェネリックにする

OptionとBoxですこしだけジェネリクスには触れました．しかし，これまでは
なんとかジェネリックな要素を持つ型を宣言することは避けることができていました．

実はジェネリック型の宣言はとても簡単です．とりあえず全部ジェネリックにしましょう：

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

コードに若干とげとげを追加することで唐突にジェネリックになりました．もちろん
これ*だけ*ではだめです．コンパイラにバチクソ怒られるでしょう．


```text
> cargo test

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:14:6
   |
14 | impl List {
   |      ^^^^ expected 1 type argument

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:36:15
   |
36 | impl Drop for List {
   |               ^^^^ expected 1 type argument

```

ここでの問題は明らかです．私達はこの`List`とかいうのを扱っていますが，そんなものは
もう存在しませｎ．OptionやBoxのように，常に`List<何か>`を扱わなくてはいけないのです．

しかしこのimplに渡す何かとは何でしょうか？List同様，*全て*のコードがTに対して
動いてほしいですよね．ではListと同じく`impl`もとげとげにしましょう：


```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<T> {
        self.head.take().map(|node| {
            self.head = node.next;
            node.elem
        })
    }
}

impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

…そしてこれで終わりです！


```
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

私達のコードはこれで完全に任意の型Tについてジェネリックになりました．いやマジ
Rust*楽勝*だな．ここで一切の変更が加えられていない`new`に注目してみましょう：

```rust ,ignore
pub fn new() -> Self {
    List { head: None }
}
```

リファクタリングとコピペの守護神，Selfの栄光を讃えましょう．Listを生成する際
`List<T>`と書いていないことにも注目してください．`List<T>`を返す関数の戻り値
であることから，関数を読んだ側からTの型が推論されます．

さて，次は全く新しい*処理*を書いていきましょう！
