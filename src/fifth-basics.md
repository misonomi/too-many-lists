# 基本

はい，では基本に立ち返りましょう．どうやってリストを初期化したら
よいでしょうか？

前回はこうしました：

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }
}
```

しかしもはや`tail`にOptionを使っていません：

```text
> cargo build

error[E0308]: mismatched types
  --> src/fifth.rs:15:34
   |
15 |         List { head: None, tail: None }
   |                                  ^^^^ expected *-ptr, found enum `std::option::Option`
   |
   = note: expected type `*mut fifth::Node<T>`
              found type `std::option::Option<_>`
```

Optionを使うこともできるにはできますが，Boxと違って`*mut`はnullになり得ます．つまり
ヌルポインタ最適化の恩恵を受けられないのです．なので，Optionを使う代わりに`null`で
Noneを表すことにしましょう．

ではヌルポインタを代入するにはどうすればいいでしょうか？いくつかやり方はありますが，
私は`std::ptr::null_mut()`を使うのが好きです．`0 as *mut _`を使う手もありますが
ちょっと汚く見えます．

```rust ,ignore
use std::ptr;

// defns...

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: ptr::null_mut() }
    }
}
```

```text
cargo build

warning: field is never used: `head`
 --> src/fifth.rs:4:5
  |
4 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `tail`
 --> src/fifth.rs:5:5
  |
5 |     tail: *mut Node<T>,
  |     ^^^^^^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/fifth.rs:11:5
   |
11 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `head`
  --> src/fifth.rs:12:5
   |
12 |     head: Link<T>,
   |     ^^^^^^^^^^^^^
```

黙ってろコンパイラ，今から使うから．

はい，ではもう一度`push`を実装していきましょう．今回は挿入したあとに`Option<&mut Node<T>>`
を取るのではなく，Boxの中にある`*mut Node<T>`をそのまま取ります．これがうまくいくのは
Boxを移動しても中身のメモリアドレスは変わらないからですね．もちろんこれは不安全な操作です．
もしBoxをdropしてしまったら私達は解放されたメモリを指すポインタを持つことになります．

どうすれば普通のポインタから生ポインタを作れるのでしょうか？Coercionです！
もし変数が生ポインタとして宣言されていれば，普通の参照は生ポインタになることを強制
（Coerce）されます：

```rust ,ignore
let raw_tail: *mut _ = &mut *new_tail;
```

これで必要なものは揃いました．前回の参照を使ったバージョンとだいたい同じ感じに
書けます：

```rust ,ignore
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(Node {
        elem: elem,
        next: None,
    });

    let raw_tail: *mut _ = &mut *new_tail;

    // .is_null checks for null, equivalent to checking for None
    if !self.tail.is_null() {
        // If the old tail existed, update it to point to the new tail
        self.tail.next = Some(new_tail);
    } else {
        // Otherwise, update the head to point to it
        self.head = Some(new_tail);
    }

    self.tail = raw_tail;
}
```

```text
> cargo build

error[E0609]: no field `next` on type `*mut fifth::Node<T>`
  --> src/fifth.rs:31:23
   |
31 |             self.tail.next = Some(new_tail);
   |             ----------^^^^
   |             |
   |             help: `self.tail` is a raw pointer; try dereferencing it: `(*self.tail).next`
```

は？Nodeのポインタを持ってるのに`next`をとれないの？

Rustは生ポインタを使い出すとちょっと嫌なヤツになります．生ポインタの中身にアクセス
するとき，不安全な操作だと言って手動で参照外しをすることを強制してくるのです．では
そのようにやってみましょう：

```rust ,ignore
*self.tail.next = Some(new_tail);
```

```text
> cargo build

error[E0609]: no field `next` on type `*mut fifth::Node<T>`
  --> src/fifth.rs:31:23
   |
31 |             *self.tail.next = Some(new_tail);
   |             -----------^^^^
   |             |
   |             help: `self.tail` is a raw pointer; try dereferencing it: `(*self.tail).next`
```

うううううオペレータの優先順を考えなくてはいけませんでした．

```rust ,ignore
(*self.tail).next = Some(new_tail);
```

```text
> cargo build

error[E0133]: dereference of raw pointer is unsafe and requires unsafe function or block
  --> src/fifth.rs:31:13
   |
31 |             (*self.tail).next = Some(new_tail);
   |             ^^^^^^^^^^^^^^^^^ dereference of raw pointer
   |
   = note: raw pointers may be NULL, dangling or unaligned; they can violate aliasing rules and cause data races: all of these are undefined behavior
```

こんな難しいわけないだろ．

不安全なRustは安全なRustにとってFFIみたいなものだと言ったことを覚えていますか？えっと，
私達はコンパイラにどこでFFIしているか明示する必要があります．方法は二つです．一つ目は
関数全体を不安全にすることで，そうすると関数は不安全なRustに属し，`unsafe`なところ
からしか呼べなくなります．これはあまり良くありません．私達はリストを使う分には安全に
したいのですから．二つ目は関数内に`unsafe`ブロックを書き，FFIの境界を切ることです．
この場合関数自体は安全なままです．後者でいってみましょう：


```rust ,ignore
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(Node {
        elem: elem,
        next: None,
    });

    let raw_tail: *mut _ = &mut *new_tail;

    // Put the box in the right place, and then grab a reference to its Node
    if !self.tail.is_null() {
        // If the old tail existed, update it to point to the new tail
        unsafe {
            (*self.tail).next = Some(new_tail);
        }
    } else {
        // Otherwise, update the head to point to it
        self.head = Some(new_tail);
    }

    self.tail = raw_tail;
}
```

```text
> cargo build
warning: field is never used: `elem`
  --> src/fifth.rs:11:5
   |
11 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default
```

イエーイ！

他のいろんなところでも生ポインタを扱っているのにここ*しか*unsafeにしなくていいのは
ちょっとおもしろいですね．何が起こっているのでしょうか？

`unsafe`のことになるとRustはかなり仕切りたがり屋であるようです．私達は安全なRustのほうが
自信があるので当然できるだけそっちを使いたいわけですが，Rustはそれを達成するために
不安全な部分が最小になるような境界を注意深く引いてくれるのです．私達が生ポインタを
使っている他の部分はポインタに*代入している*か，nullかどうかチェックしているだけである
ことに注目してください．

もし参照を外さないのであれば，*生ポインタは完全に安全です*．ただ整数を読んだり書いたり
しているだけです！生ポインタで問題が起きるのは参照を外すときだけなので，Rustはその
操作*だけ*を不安全だと言い，ほかは安全な操作として扱うのです．

超．衒学的．でも技術的には正しいですね．

しかしこれは興味深い問題を生みます．もし`unsafe`ブロックを区切っても，その中の
状態はunsafeブロックの外部に依存します．もしかしたら関数の外にすら依存するかも
しれません！

わたしはこれをunsafe*汚染*と呼んでいます．`unsafe`を使うや否やモジュール全体が
不安全に汚染されてしまうのです．コードの不変性が不安全さを支えて持ちこたえられる
ためには，全てが正しく実装されている必要があります．

この汚染は*プライバシー*によって食い止められます．私達のモジュール外からはstructの
フィールドは操作できないので私達以外の誰もそれらの内部状態を好き勝手することはできません．
私達のAPIをどんなふうに組み合わせも安全であり，かつ外部から操作できる範囲がちゃんと
正しく決められている限り，私達のコードは安全です！そして本当にこれはFFIと何も
変わりません．PythonのライブラリがCを呼び出していようと，安全なインターフェースを
提供している限り誰も気にしません．

ともかく`pop`にいきましょう．参照をつかうバージョンとほぼ一緒です：

```rust ,ignore
pub fn pop(&mut self) -> Option<T> {
    self.head.take().map(|head| {
        let head = *head;
        self.head = head.next;

        if self.head.is_none() {
            self.tail = ptr::null_mut();
        }

        head.elem
    })
}
```

またしても安全とはステートフルであることを示すケースに遭遇しました．もしこの関数で
tailのポインタをnullにするのを忘れても，すぐにはなんの問題も現れません．しかし
そのあと`push`するとダングリングポインタになっているtailに書き込んでしまいます！

テストしていきましょう：

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop(), None);

        // Populate list
        list.push(1);
        list.push(2);
        list.push(3);

        // Check normal removal
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), None);

        // Check the exhaustion case fixed the pointer right
        list.push(6);
        list.push(7);

        // Check normal removal
        assert_eq!(list.pop(), Some(6));
        assert_eq!(list.pop(), Some(7));
        assert_eq!(list.pop(), None);
    }
}
```

これはスタックのテストそのままですが，`pop`が逆順に出てくることを期待している点が違います．
また，最後に`pop`が空振りしたときもポインタが壊れていないか確認するケースを追加しています．

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 12 tests
test fifth::test::basics ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured
```

大金星！
