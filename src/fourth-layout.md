# 設計

今回のキモは`RefCell`型です．RefCellの重要な部分はこの2つのメソッドです：

```rust ,ignore
fn borrow(&self) -> Ref<'_, T>;
fn borrow_mut(&self) -> RefMut<'_, T>;
```
`borrow`と`borrow_mut`の関係は`&`と`&mut`の関係と全く同じです． `borrow`は
好きなだけ呼ぶことができますが`borrow_mut`は排他的にしか呼べません．

RefCellはこの条件をコンパイルタイムではなくランタイムにチェックします．
もしルールが守られなければRefCellはパニックを起こし，プログラムは
クラッシュします．ところで，このRefとかRefMutとかいう型はなんでしょう？
これは基本的には借用のために使われるRcみたいなもので，これがスコープ外
に出るまでRefCellは借用されたままになります．これについては後で触れます．

そしてRcとRefCellを使えば私達は...ビビるほど冗長であちこち可変だしサイクルを
解放できないガベージコレクタを作れます！イエェェェェィ...

はい．今回は*双方向連結*でやっていきたいんでした．これはそれぞれのノードが
一つ前と一つ次のノードを指すポインタを持っていることを意味しています．更に
リスト自身ははじめのノードと最後のノードのポインタを持ちます．これによって
はじめの要素と最後の要素の*両方*に対して高速な挿入と削除が行えます．

なので，多分私達が作りたいのはこんな感じのものでしょう：

```rust ,ignore
use std::rc::Rc;
use std::cell::RefCell;

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>,
}

type Link<T> = Option<Rc<RefCell<Node<T>>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
    prev: Link<T>,
}
```

```text
> cargo build

warning: field is never used: `head`
 --> src/fourth.rs:5:5
  |
5 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `tail`
 --> src/fourth.rs:6:5
  |
6 |     tail: Link<T>,
  |     ^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/fourth.rs:12:5
   |
12 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `next`
  --> src/fourth.rs:13:5
   |
13 |     next: Link<T>,
   |     ^^^^^^^^^^^^^

warning: field is never used: `prev`
  --> src/fourth.rs:14:5
   |
14 |     prev: Link<T>,
   |     ^^^^^^^^^^^^^
```

見てください！ビルドしました！未使用コードの警告はありますが
ビルドは通っています！ではこのコードを使っていきましょう．
