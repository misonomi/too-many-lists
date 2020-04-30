# Pop

`push`同様，`pop`もリストに変更を加えますが，`push`と違い返り値があります．
そしてリストが空の場合という厄介な状態を考えなくてはいけません．この
ケースに対処するために，信頼できる`Option`型を使います：

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    // TODO
}
```

`Option<T>`は値が存在するかもしれないことを表すのに使われ，`Some(T)`か`None`
の状態を取ることができます．Linkのときやったように似たような型を自作することも
できますが，このリストを使う人が戻り値の型が一体全体何なのか考えなくていいように，
Optionという知らない人がいないほどありふれてる型を使います．実際あまりにも
基礎的なので何も書かなくても`Some`と`None`と一緒にすべての.rsファイルに
インポートされています（なので`Option::None`とか書かなくてもいいのです）．

`Option<T>`のとげとげはOptionがTの*ジェネリック型*であることを表しています．
どんな型のOptionも作ることができるのです！

はい，で，この`Link`とかいうのがあるわけですが，どうやってこれがEmptyかMoreか
判断するのでしょうか？`match`を使ったパターンマッチングです！

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
}
```

```text
> cargo build

error[E0308]: mismatched types
  --> src/first.rs:27:30
   |
27 |     pub fn pop(&mut self) -> Option<i32> {
   |            ---               ^^^^^^^^^^^ expected enum `std::option::Option`, found ()
   |            |
   |            this function's body doesn't return
   |
   = note: expected type `std::option::Option<i32>`
              found type `()`
```

おおおっと．`pop`は値を返さなくてはいけませんが，まだ実装してませんでした．
`None`を返すこともできますが，このような場合は関数がまだ未実装であることを
表す`unimplemented!()`を返すのがよさそうです．`unimplemented!()`はマクロで
（`!`がマクロであることを表しています），実行するとプログラムがパニックします
（制御下でクラッシュさせます）．

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
    unimplemented!()
}
```

無条件でパニックするような関数は[発散する関数][diverging]と呼ばれます．
発散する関数は値を返さず，したがってどんな型の値としても使うことが
できます．ここでは`unimplemented!()`の返り値を`Option<T>`として使っています．

`return`を書かなくていいことにも注目してください．最後の表現（基本的には行）
が暗黙的に関数の返り値になります．これでシンプルな関数をより簡潔に書くことが
できるのです．ほかのCライクな言語のように，明示的に`return`を使って
早期リターンすることもできます．

```text
> cargo build

error[E0507]: cannot move out of borrowed content
  --> src/first.rs:28:15
   |
28 |         match self.head {
   |               ^^^^^^^^^
   |               |
   |               cannot move out of borrowed content
   |               help: consider borrowing here: `&self.head`
...
32 |             Link::More(node) => {
   |                        ---- data moved here
   |
note: move occurs because `node` has type `std::boxed::Box<first::Node>`, which does not implement the `Copy` trait
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^
```

頼むぞRust，邪魔しないでくれ！いつもどおりRustは激おこです．ありがたいことに
今回は何がいけないのか全部言ってくれています！デフォルトでは，パターンマッチを
するときmatch節への値のムーブが発生します．しかし今回はself.headの所有権を
持っていないためそれはできません．

```text
help: consider borrowing here: `&self.head`
```

Rustは`match`節で参照を使えと言ってます．🤷‍♀️まあ試してみましょう：

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match &self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(ref node) => {
            // TODO
        }
    };
    unimplemented!()
}
```

```text
> cargo build

warning: unused variable: `node`
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^ help: consider prefixing with an underscore: `_node`
   |
   = note: #[warn(unused_variables)] on by default

warning: field is never used: `elem`
  --> src/first.rs:13:5
   |
13 |     elem: i32,
   |     ^^^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `next`
  --> src/first.rs:14:5
   |
14 |     next: Link,
   |     ^^^^^^^^^^
```

イエーイ，コンパイルが通りました！ではTODOに入る処理を考えていきましょう．
Optionを返したいので，そのための変数を作りましょう．EmptyのときにはNoneを
返せばいいですね．Moreのときは`Some(i32)`を返し，Listのheadを更新します．
じゃあとりあえずはこんな感じでどうでしょうか？

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match &self.head {
        Link::Empty => {
            result = None;
        }
        Link::More(ref node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
> cargo build
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:35:29
   |
35 |                 self.head = node.next;
   |                             ^^^^^^^^^ cannot move out of borrowed content

```

*(頭を机に伏せる)*

共有参照しか持ってない`node`から値を取ろうとしているのがよくないようです．

ここは一旦私達が何をしようとしているのか考え直すべきでしょう．私達がやりたいのは
こういうことです：

* リストが空か確認する
* もし空ならNoneを返す
* もし空*でない*なら...
    * リストのheadを消す
    * headの`elem`を消す
    * リストのheadを古いheadの`next`に更新する
    * `Some(elem)`を返す

重要な点は私達がなにかを*消そう*としている点です．つまり，リストのheadの*値*を
持っている必要があります．これは明らかに`&self.head`の共有参照を使っていては
達成できません．そもそもこの関数は`self`の可変参照*しか*持っておらず，
*入れ替える*ことでしか値を取ることができません．これはまたEmptyの出番のよう
ですね！

試してみましょう：


```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => {
            result = None;
        }
        Link::More(node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

や　っ　た　ぜ

*一つの*警告もなくコンパイルしました！！！！！

ちょっとここで個人的なlintを行いたいと思います．私達は`result`を返り値に
していますが，実はそんなことをする必要はないのです！関数がその最後の行を
返すように，全ての節もその最後の行を返すのです．普通はセミコロンをつけることで
空タプル`()`を返すようにします．返り値を定義していない関数（`push`みたいな）も
空タプルを返します．

そんなわけで，`pop`はこんなふうに書けます：

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => None,
        Link::More(node) => {
            self.head = node.next;
            Some(node.elem)
        }
    }
}
```

この方が簡潔で慣用的です．Link::Emptyのところにカッコがないことに注目してください．
一つしか表現がないシンプルな場合にはこのように書くことができます．

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

よしよし，これでも動きますね！



[ownership]: first-ownership.html
[diverging]: https://doc.rust-jp.rs/book/second-edition/ch19-04-advanced-types.html#never%E5%9E%8B%E3%81%AF%E7%B5%B6%E5%AF%BE%E3%81%AB%E8%BF%94%E3%82%89%E3%81%AA%E3%81%84
