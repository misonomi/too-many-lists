# Push

ではリストに値を追加（push）する操作を書いていきましょう．`push`は
リストの変更を伴うので`&mut self`を引数に取ります．pushするi32の値も
引数に取る必要があります：

```rust ,ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        // TODO
    }
}
```

まずは要素をいれるノードを作りましょう．

```rust ,ignore
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: ?????
        };
    }
```

`next`には何が入るでしょうか？えーっと，リスト全部です！たぶん…こう？

```rust ,ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: self.head,
        };
    }
}
```

```text
> cargo build
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:19:19
   |
19 |             next: self.head,
   |                   ^^^^^^^^^ cannot move out of borrowed content
```

うえええええん．Rustは私達に正論を突きつけています．でもこれが具体的に何を
意味するのか，あるいはどうすればいいかはよくわかりません：

> cannot move out of borrowed content

私達は`self.head`を`next`に移そうとしましたが，Rustはそういうことをやってほしく
ないようです．これをすると`self`は一部が欠けた状態でこのメソッドから返される
ことになります．すでに見たとおり，これは`&mut`で唯一許されていないことです．
借りたものをちゃんと返さないのは超失礼なことで，Rustはとても真面目なのです
（超危険なことでもあるのですが，今回はそういう問題ではありません）．

では何かを戻したらどうでしょう？つまり，今作ったノードとか：


```rust ,ignore
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: self.head,
    });

    self.head = Link::More(new_node);
}
```

```text
> cargo build
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:19:19
   |
19 |             next: self.head,
   |                   ^^^^^^^^^ cannot move out of borrowed content
```

だめかー．原則的にはこれはRust的にはOKなのですが，実際にはだめです（理由はいくつか
ありますが，[例外に対する安全性][exception safety]が一番深刻な問題です）．Rustに値を取ったと
気づかれないように取る方法が必要です．ここは悪名高いRustのハッカー，
インディ・ジョーンズに教えを請うことにしましょう：

![mem::replaceの構えを取るインディ](img/indy.gif)

ああ！インディは`mem::replace`を使うことを提案しています．この超有能な関数を使うと，
借りた値を代わりのものと入れ替えることで盗んでしまうことができます．さしあたり
`std::mem`をファイルの先頭に持ってきて`mem`をローカルスコープで使えるようにしましょう：

```rust ,ignore
use std::mem;
```

そしていい感じに使います：

```rust ,ignore
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: mem::replace(&mut self.head, Link::Empty),
    });

    self.head = Link::More(new_node);
}
```

ここで，新しいノードをいれる前，一時的にself.headをLink::Emptyにしています．
今から嘘をつきます．こういうことをしなきゃいけないのは不幸なことです．こうする
他に方法はないのです（いまのところは）．

でもこれで`push`は完成です！多分．多分テストしたほうが良さそうですね，正直．
とりあえず一番簡単なやり方は`pop`を書いて，予期したとおりに動くことを確認する
ことでしょう．





[exception safety]: https://doc.rust-lang.org/nightly/nomicon/exception-safety.html
