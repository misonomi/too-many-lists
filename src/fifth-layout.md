# 設計

で，片方向キューとは何でしょうか？片方向スタックのとき，私達はリストの最後にpushして
同じ方向からpopしました．キューとスタックの違いはpopするのが逆側からであることだけです．
スタックのときはこんな感じでした：

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)

stack push X:
[Some(ptr)] -> (X, Some(ptr)) -> (A, Some(ptr)) -> (B, None)

stack pop:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)
```

キューを作るには，popとpushのどちらでリストの最後の要素を操作するか決めなくては
いけません．片方向リストなのでどっちにしろ同程度の計算量になります．

`push`を最後にするなら，リストを`None`が出るまでたどり新しい要素が入ったSomeに
入れ替えます．

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)

flipped push X:
[Some(ptr)] -> (A, Some(ptr)) -> (B, Some(ptr)) -> (X, None)
```

`pop`を最後にするなら，リストを`None`が出る*直前まで*たどり`take`します．

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, Some(ptr)) -> (X, None)

flipped pop:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)
```

これのどちらかをやって完成でもいいですが，これではお粗末です！どちらにせよリストの
*全体*をたどる必要があります．正しいインターフェースを提供しているのでキューの実装としては
これでいいという人もいるでしょう．ですが私はパフォーマンスを保証することもインターフェースの
一部であると考えます．漸近的境界がどうのという話ではなく*速い*か*遅い*かです．キューは
popとpushが速いことを保証するデータ構造であり，リスト全体をたどる操作は明らかに速くは
ありません．

ひとつの重要な視点は，同じことを何回も繰り返してリソースを無駄にしていることです．
この操作を記憶できないでしょうか？なんと，できます！リストの末尾へのポインタを持ち，
そこに直接行けばいいのです！

リストの末尾を使う操作は`push`と`pop`のどちらかで大丈夫です．`pop`で使う場合
末尾へのポインタを逆向きにたどる必要がありますが，片方向リストの性質上これを
効率的に行うのは困難です．`push`で使うようにすれば，popするときはリストの先頭の
ポインタを順方向にたどればいいので楽ですね．

試してみましょう：

```rust ,ignore
use std::mem;

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            next: None,
        });

        // swap the old tail to point to the new tail
        let old_tail = mem::replace(&mut self.tail, Some(new_tail));

        match old_tail {
            Some(mut old_tail) => {
                // If the old tail existed, update it to point to the new tail
                old_tail.next = Some(new_tail);
            }
            None => {
                // Otherwise, update the head to point to it
                self.head = Some(new_tail);
            }
        }
    }
}
```

この手の操作を書くのはかなり慣れてきていると思うので実装のペースをすこし上げていきます．
とはいえ上のようなコードを一発で書き上げる必要はありません．今まであったような
トライアンドエラーをとばして書いているだけです．実際私はこのコードを書くにあたって
めちゃくちゃ間違えてます．私が出したエラーを全部見せていたら`mut`や`;`の書き忘れが
あまりにも多く教本の体をなしていないでしょう．でも心配しないでください．これから
*別の*エラーメッセージを沢山見ることになります！

```text
> cargo build

error[E0382]: use of moved value: `new_tail`
  --> src/fifth.rs:38:38
   |
26 |         let new_tail = Box::new(Node {
   |             -------- move occurs because `new_tail` has type `std::boxed::Box<fifth::Node<T>>`, which does not implement the `Copy` trait
...
33 |         let old_tail = mem::replace(&mut self.tail, Some(new_tail));
   |                                                          -------- value moved here
...
38 |                 old_tail.next = Some(new_tail);
   |                                      ^^^^^^^^ value used here after move
```

クソ！

> use of moved value: `new_tail`

BoxはCopyを実装していないので2箇所に入れることはできません．更に重要なことはBoxは
中身の所有権を持つのでBoxがdropされたとき中身を解放しようとする点です．もしこの
実装がコンパイルしてしまったら`push`するたびにリストの末尾を2回解放することになります．
というか実際私達のコードはpushのたびにold_tailを解放しています！ギエー！🙀

OK，でも私達は所有権を持たないポインタがなにか知っています．ふつうの参照です！

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
    tail: Option<&mut Node<T>>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            next: None,
        });

        // Put the box in the right place, and then grab a reference to its Node
        let new_tail = match self.tail.take() {
            Some(old_tail) => {
                // If the old tail existed, update it to point to the new tail
                old_tail.next = Some(new_tail);
                old_tail.next.as_mut().map(|node| &mut **node)
            }
            None => {
                // Otherwise, update the head to point to it
                self.head = Some(new_tail);
                self.head.as_mut().map(|node| &mut **node)
            }
        };

        self.tail = new_tail;
    }
}
```

特にトリッキーなことはしていません．基本的な発想はさっきと同じで，違いは暗黙のリターン
を使ってnew_tailを作っていることくらいです．

```text
> cargo build

error[E0106]: missing lifetime specifier
 --> src/fifth.rs:3:18
  |
3 |     tail: Option<&mut Node<T>>, // NEW!
  |                  ^ expected lifetime parameter
```

ああそうでした．ライフタイムを与えなくてはいけません．うーん...ここのライフタイムは
何でしょう？えっと，これってIterMutに似てますよね？IterMutと同じことをして，
パラメータ名は`'a`を使いましょう：

```rust ,ignore
pub struct List<'a, T> {
    head: Link<T>,
    tail: Option<&'a mut Node<T>>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<'a, T> List<'a, T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            next: None,
        });

        // Put the box in the right place, and then grab a reference to its Node
        let new_tail = match self.tail.take() {
            Some(old_tail) => {
                // If the old tail existed, update it to point to the new tail
                old_tail.next = Some(new_tail);
                old_tail.next.as_mut().map(|node| &mut **node)
            }
            None => {
                // Otherwise, update the head to point to it
                self.head = Some(new_tail);
                self.head.as_mut().map(|node| &mut **node)
            }
        };

        self.tail = new_tail;
    }
}
```

```text
cargo build

error[E0495]: cannot infer an appropriate lifetime for autoref due to conflicting requirements
  --> src/fifth.rs:35:27
   |
35 |                 self.head.as_mut().map(|node| &mut **node)
   |                           ^^^^^^
   |
note: first, the lifetime cannot outlive the anonymous lifetime #1 defined on the method body at 18:5...
  --> src/fifth.rs:18:5
   |
18 | /     pub fn push(&mut self, elem: T) {
19 | |         let new_tail = Box::new(Node {
20 | |             elem: elem,
21 | |             // When you push onto the tail, your next is always None
...  |
39 | |         self.tail = new_tail;
40 | |     }
   | |_____^
note: ...so that reference does not outlive borrowed content
  --> src/fifth.rs:35:17
   |
35 |                 self.head.as_mut().map(|node| &mut **node)
   |                 ^^^^^^^^^
note: but, the lifetime must be valid for the lifetime 'a as defined on the impl at 13:6...
  --> src/fifth.rs:13:6
   |
13 | impl<'a, T> List<'a, T> {
   |      ^^
   = note: ...so that the expression is assignable:
           expected std::option::Option<&'a mut fifth::Node<T>>
              found std::option::Option<&mut fifth::Node<T>>


```

わー，これはずいぶん詳細なエラーメッセージですね．これはすこし憂慮すべきエラーです．
というのもこれによると私達はかなりめちゃくちゃなことをやっているからです．興味深いのは
部分です：

> the lifetime must be valid for the lifetime `'a` as defined on the impl

`self`を借用しているわけですが，コンパイラは少なくとも`'a`だけ生存して欲しがっています．
では`self`が実際`'a`だけ生存すると示せばどうでしょう...？

```rust ,ignore
    pub fn push(&'a mut self, elem: T) {
```

```text
cargo build

warning: field is never used: `elem`
 --> src/fifth.rs:9:5
  |
9 |     elem: T,
  |     ^^^^^^^
  |
  = note: #[warn(dead_code)] on by default
```

見てください，動きました！やった！

`pop`も同様に実装します：

```rust ,ignore
pub fn pop(&'a mut self) -> Option<T> {
    // Grab the list's current head
    self.head.take().map(|head| {
        let head = *head;
        self.head = head.next;

        // If we're out of `head`, make sure to set the tail to `None`.
        if self.head.is_none() {
            self.tail = None;
        }

        head.elem
    })
}
```

そしてちゃちゃっとテストを書いてしまいましょう：

```rust ,ignore
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
    }
}
```

```text
cargo test

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:68:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
68 |         list.push(1);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:69:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
69 |         list.push(2);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:70:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
70 |         list.push(3);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here


....

** まだまだあるエラー **

....

error: aborting due to 11 previous errors
```

🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀

何ということでしょう．

コンパイラは悪くありません．私達はたった今Rustの原罪[^1]を犯したのです．私達は
自分への参照を*自分自身の中に*持ってしまいました．`push`と`pop`の実装では
なんとかRustを言いくるめることができました（それができたのは衝撃的でしたが）．
私はRustが`push`と`pop`を見た時点では参照がそれ自身の中にあるかどうか
分からなかったんだと思いますが--というか，Rustにはそういう概念がありません．
自分自身への参照が動作しないのは

The compiler's not wrong for vomiting all over us. We just committed a
cardinal Rust sin: we stored a reference to ourselves *inside ourselves*.
Somehow, we managed to convince Rust that this totally made sense in our
`push` and `pop` implementations (I was legitimately shocked we did). I believe
the reason is that Rust can't yet tell that the reference is into ourselves
from just `push` and `pop` -- or rather, Rust doesn't really have that notion
at all. Reference-into-yourself failing to work is just an emergent behaviour.

私達のリストは使おうとした途端全てが空中分解します．`push`や`pop`を呼んだ瞬間
リストは自分への参照を持ち，*詰みます*．自分自身を借用しているのです．

`pop`の実装を見るとなぜこれが危険であるかわかります：

```rust ,ignore
// ...
if self.head.is_none() {
    self.tail = None;
}
```

もし私達がこれを忘れたらどうなっていたでしょう？リストのtailは*リストから除外された
ノード*を指すことになります．リストから除外されたノードのメモリは解放され，Rust
が防いでくれるはずのダングリングポインタが生まれてしまいます！

Rustはたしかにそういう危険から私達を守ってくれているのです．ただかなり...
**回りくどい**やりかたですが．

ではどうしたらいいでしょうか？また`Rc<RefCell>>`地獄に戻る？

だめです．勘弁してください．

かわりに敷かれたレールを外れて*生のポインタ*を使います．私達のリストはこんな感じに
なるでしょう：

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>, // DANGER DANGER
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

そしてこれが全てです．もうクソ雑魚参照カウント付動的借用チェックには頼りません！
本物の，過酷な，未検査の，ポインタを使います．

みなさん，C言語でいきましょう．いつも，いつでもCで．

私はここに帰ってきました．もう準備はできてます．

Hello, `unsafe`.

[^1]: 訳注：Lust（色欲）とかかっている
