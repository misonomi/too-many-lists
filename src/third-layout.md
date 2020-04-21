# 設計

では再び設計作業に入りましょう．

永続リストで最も重要なことはリストの末尾をほぼノーコストで操作できることです．

例えばこのような操作は永続リストではよくあることです：

```text
list1 = A -> B -> C -> D
list2 = tail(list1) = B -> C -> D
list3 = push(list2, X) = X -> B -> C -> D
```

そして最終的にはメモリ上でこのような構造になっていてほしいのです：

```text
list1 -> A ---+
              |
              v
list2 ------> B -> C -> D
              ^
              |
list3 -> X ---+
```

これはBoxでは実現できません．なぜなら`B`が*共有参照*だからです．Bのメモリ割り当て
が解放されるときはどんなときでしょうか？今list2を解放したらBは解放されるべきでしょうか？
Boxを使うならそうなって然るべきです！

関数型言語では--ほぼすべての言語でそうですが--これをガベージコレクションで解決
しています．ガベージコレクションの魔法のおかげでBが開放されるのはBを参照している
全てが解放された後です．うほほーい！

Rustにはガベージコレクションがない代わりに，*参照カウンタ*というものがあります．
参照カウンタは極めてシンプルなGCみたいなものです．大抵の場合トレーシングGCよりも
性能は著しく低いうえ，参照のループがあるとうまく動きません．でもこれしか
ないんだからしょうがないですね！幸い私達はループを作ることはないので（証明
してみてください．自信あります）大丈夫です．

ではどうすれば参照カウンタを使えるのでしょう？`Rc`です！RcはBoxのようなものですが，
複製することができ，そのメモリは参照が*全て*外されて*はじめて*解放されます．
不幸にもこの柔軟性の代償は高くつきます．というのは，Rcの中身の共有参照しか
得ることができないのです．つまり，Rcを使ったリストはデータを取り出すことも
変更することもできないのです．

ではどういう設計になりでしょうか？前回はこうでした：

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

単にBoxをRcに変えるとどうでしょうか？

```rust ,ignore
// in third.rs

pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Rc<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

```text
cargo build

error[E0412]: cannot find type `Rc` in this scope
 --> src/third.rs:5:23
  |
5 | type Link<T> = Option<Rc<Node<T>>>;
  |                       ^^ not found in this scope
help: possible candidate is found in another module, you can import it into scope
  |
1 | use std::rc::Rc;
  |
```

はいクソ．私達が今まで使ってきたものと違い，RcはRustのプログラムにデフォルトで
インポートされていないのでした．*雑魚が*．

```rust ,ignore
use std::rc::Rc;
```

```text
cargo build

warning: field is never used: `head`
 --> src/third.rs:4:5
  |
4 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `elem`
  --> src/third.rs:10:5
   |
10 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `next`
  --> src/third.rs:11:5
   |
11 |     next: Link<T>,
   |     ^^^^^^^^^^^^^
```

大丈夫そうですね．Rustは相変わらずどうでもいいことをいちいち指摘していますが．
でもこれはBoxをRcに書き換えて全部終わりにできそうですね！

...

だめです．そうは問屋がおろしません．
