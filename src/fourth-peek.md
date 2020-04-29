# Peek

さて，`push`と`pop`を実装しました．私はちょっと感動してます．マジで．コンパイル時
の正確性があるというのは病みつきになりそうです．

簡単な`peek_front`でも実装して落ち着きましょう．いままでこのメソッドは簡単でしたし
今回も簡単なはずです．そうですよね？

そうですよね？

実際コピペで済みそうです！

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

ちょっと待った．今回はこうです．

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        // BORROW!!!!
        &node.borrow().elem
    })
}
```

どうよ．

```text
cargo build

error[E0515]: cannot return value referencing temporary value
  --> src/fourth.rs:66:13
   |
66 |             &node.borrow().elem
   |             ^   ----------^^^^^
   |             |   |
   |             |   temporary value created here
   |             |
   |             returns a value referencing data owned by the current function
```

PC燃やしました．

片方向リストのときと同じロジックだろ．なんで違うんだ．なんで...

その答えはこの章から得られる教訓そのものです．その教訓とは，RefCellはあらゆるものに
悲しみをもたらす存在であるということです．これまでRefCellはただの困ったちゃんでしたが，
悪夢と化しつつあります．

実際のところ何がどうなってるのでしょうか？それを理解するために`borrow`の定義をもう一度
見てみましょう：

```rust ,ignore
fn borrow<'a>(&'a self) -> Ref<'a, T>
fn borrow_mut<'a>(&'a self) -> RefMut<'a, T>
```

設計の節でこのように書きました：

> RefCellはこの条件をコンパイルタイムではなくランタイムにチェックします。
> もしルールが守られなければRefCellはパニックを起こし、プログラムは
> クラッシュします。ところで、このRefとかRefMutとかいう型はなんでしょう？
> これは基本的には借用のために使われるRcみたいなもので、これがスコープ外
> に出るまでRefCellは借用されたままになります。**これについては後で触れます。**

今がその「後」です．

`Ref`と`RefMut`はそれぞれ`Deref`と`DerefMut`を実装しています．これらは`&T`，`&mut T`
と*全く*同じ動作をするようになっているのですが，トレイトの実装上，戻り値のライフタイムは
RefCellではなくRefに紐付けられるようになっています．つまり戻り値の参照を使い続ける
間ずっとRefを生かしておかなくてはいけないということになります．

実はこれは整合性を取るためには必要なことです．Refがdropされれば，RefCellはもうそれを
借用しているものはないと判断してしまします．なのでもしRefより長く中の参照を持ち続け
ることができてしまったら，RefMutの排他性が損なわれRustの型システムを半壊させてしまいます．

結局どうしたらいいのでしょうか？ただ参照を返したいだけなのですが，参照を持ち続ける限り
Refを持ち続けなくてはいけません．そして`peek`がreturnしたとき`Ref`はスコープ外に
行ってしまいます．

😖

私が知る限りこれは手詰まりです．今回のような場合RefCellをカプセル化することはできないのです．

でも...もし実装の隠蔽を諦めたらどうでしょうか？Refを返したらどうなるのでしょう？

```rust ,ignore
pub fn peek_front(&self) -> Option<Ref<T>> {
    self.head.as_ref().map(|node| {
        node.borrow()
    })
}
```

```text
> cargo build

error[E0412]: cannot find type `Ref` in this scope
  --> src/fourth.rs:63:40
   |
63 |     pub fn peek_front(&self) -> Option<Ref<T>> {
   |                                        ^^^ not found in this scope
help: possible candidates are found in other modules, you can import them into scope
   |
1  | use core::cell::Ref;
   |
1  | use std::cell::Ref;
   |
```

ぶっは．importしなきゃいけませんね．


```rust ,ignore
use std::cell::{Ref, RefCell};
```

```text
> cargo build

error[E0308]: mismatched types
  --> src/fourth.rs:64:9
   |
64 | /         self.head.as_ref().map(|node| {
65 | |             node.borrow()
66 | |         })
   | |__________^ expected type parameter, found struct `fourth::Node`
   |
   = note: expected type `std::option::Option<std::cell::Ref<'_, T>>`
              found type `std::option::Option<std::cell::Ref<'_, fourth::Node<T>>>`
```

うーん...確かに．`Ref<Node<T>>`を返していますが欲しいのは`Ref<T>`です．全てを諦めて
`Ref<Node<T>>`を返すというのも手ですし，`&T`にだけアクセスできるような型で`Ref<Node<T>>`
をラップし事態を更に複雑化させるという手もあります．

どちらも*まあまあ*ダサいですね．

かわりにさらなる深淵を覗きましょう．*楽しもう*じゃないですか．楽しみの種はこの
*やべーやつ*です：

```rust ,ignore
map<U, F>(orig: Ref<'b, T>, f: F) -> Ref<'b, U>
    where F: FnOnce(&T) -> &U,
          U: ?Sized
```

> Make a new Ref for a component of the borrowed data.

そう，Option同様Refもmapできるのです．

どこかの誰かは*モナド*とか何とか言って興奮しているかと思いますが，私にはどうでもいい
ことです．あとこのメソッドはNothingのケースを持たないので厳密にはモナドではないと
思います．話がそれました．

このメソッドはイカすということだけが重要です．*使わせろ*．
It's cool and that's all that matters to me. *I need this*.

```rust ,ignore
pub fn peek_front(&self) -> Option<Ref<T>> {
    self.head.as_ref().map(|node| {
        Ref::map(node.borrow(), |node| &node.elem)
    })
}
```

```text
> cargo build
```

やっっっった．

ちゃんと動いていることを，スタックのときに実装したテストを修正して確認しましょう．Ref
同士の比較はできないのでちょっと修正が必要です．

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert!(list.peek_front().is_none());
    list.push_front(1); list.push_front(2); list.push_front(3);

    assert_eq!(&*list.peek_front().unwrap(), &3);
}
```


```
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 10 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::basics ... ok
test fourth::test::peek ... ok
test second::test::iter_mut ... ok
test second::test::into_iter ... ok
test third::test::basics ... ok
test second::test::peek ... ok
test second::test::iter ... ok
test third::test::iter ... ok

test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured

```

やりました！
