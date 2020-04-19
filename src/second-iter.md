# Iter

さて、ではIterを実装していきましょう。今回はすでにある実装に頼るわけには
いかないので、自らの手で実装しましょう。基本的な処理は、次に返したいノード
の一つ前のノードのポインタを握り続けることです。というのも、次に返したい
ノードが存在しないかもしれないからです（リストが空か、最後の要素を返した後
そういうことになります）。参照はOptionである必要がありますね。要素を返したら
そのノードの`next`にポインタを移す処理が要ります。

よし、やってみましょう：

```rust ,ignore
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

```text
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:72:18
   |
72 |     next: Option<&Node<T>>,
   |                  ^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:82:17
   |
82 |     type Item = &T;
   |                 ^ expected lifetime parameter
```

ああ、寿命ね…。ライフタイムについては聞いたことがあります。悪夢のような代物
だと言う話ですが。
Oh god. Lifetimes. I've heard of these things. I hear they're a nightmare.

ちょっと新しいことをやってみましょう。`error[E0106]`っていうのがありますね？
これはコンパイラエラーコードです。これが何なのかrustcに聞くことができるオプション
が、えっと、`--explain`です：

```text
> rustc --explain E0106
This error indicates that a lifetime is missing from a type. If it is an error
inside a function signature, the problem may be with failing to adhere to the
lifetime elision rules (see below).

Here are some simple examples of where you'll run into this error:

struct Foo { x: &bool }        // error
struct Foo<'a> { x: &'a bool } // correct

enum Bar { A(u8), B(&bool), }        // error
enum Bar<'a> { A(u8), B(&'a bool), } // correct

type MyStr = &str;        // error
type MyStr<'a> = &'a str; //correct
...

```

これは、その…あんまり助けになりませんね（このドキュメントは私達がいま持っている
より深いRustへの理解を要求しています）でもこの`'a`っていうのをstructにつければ
良さそうに見えますね？やってみましょう。

```
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}
```

```text
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:83:22
   |
83 | impl<T> Iterator for Iter<T> {
   |                      ^^^^^^^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:84:17
   |
84 |     type Item = &T;
   |                 ^ expected lifetime parameter

error: aborting due to 2 previous errors
```

なるほど、だんだんパターンが見えてきました。このちっこいのをあらゆるところに
つけまくってみましょう：

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<'a, T> List<T> {
    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &'a node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&'a mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &'a node);
            &'a node.elem
        })
    }
}
```

```text
> cargo build

error: expected `:`, found `node`
  --> src/second.rs:77:47
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ---- while parsing this struct        ^^^^ expected `:`

error: expected `:`, found `node`
  --> src/second.rs:85:50
   |
85 |             self.next = node.next.map(|node| &'a node);
   |                                                  ^^^^ expected `:`

error[E0063]: missing field `next` in initializer of `second::Iter<'_, _>`
  --> src/second.rs:77:9
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ^^^^ missing `next`
```

あーっ。Rustが壊れてしまいました。

多分この`'a`が一体何を意味するのか知る必要がありそうです。

ライフタイムは沢山の人々を恐怖によって遠ざけてきました。プログラミングの
黎明から私達が慣れ親しんできた概念を変えてしまうからです。これまでは
なんとかライフタイムから逃れ続けてくることができましたが、実はこれまでもずっと
私達のプログラムに絡みついていたのです。

ライフタイムはガベージコレクションを持つ言語には必要のないものです。ガベージコレクタ―
が魔法のように全ての寿命を管理してくれるからです。Rustではほとんどのデータが
*手動で*管理されるため、別のやり方が必要でした。CとC++から、人間にポインタを
与えて管理させると制御不能のメモリ不安全性がはびこることがすでに分かっています。
この不安全性の原因はおおむね次の2種類の誤りに分類できます：

* スコープ外に出た物のポインタを持ち続ける
* 変更されてしまった物のポインタを持ち続ける

ライフタイムはどちらの問題も解決し、99%の場合透過的に処理してくれます。

で、ライフタイムとはなんでしょうか？

簡単に言うと、コードの中の特定の領域（ブロックやスコープ）のことです。終わり。
借用がライフタイムに紐付けられたとき、その借用はそのライフタイムの*あいだじゅう*有効
でなくてはならないことを表しています。様々な条件によって借用がどのくらい生存
しなくてはいけないか、また生存できるかが決まります。とどのつまりライフタイムという
仕組みは、それらの条件の中で借用の寿命を最小化する制約ソルバなのです。もし
すべての条件を満たすライフタイムが見つかれば、コンパイルが通ります！さもなければ、
何かの寿命が足りないというエラーが返ってくるでしょう。

一般に、関数内でのライフタイムについて言及する必要はありませんし、*何があっても*
言及したくないはずです。コンパイラは全ての借用の寿命を知っていて、ちゃんと
最小のライフタイムを見つけることができます。しかしAPIレベルではコンパイラは
十分な情報を持って*いません*。なので人間がライフタイムどうしの関係を手動で教えて
あげる必要があるのです。

原則としては、依存パッケージも含めてライフタイムをチェックすれば人間が教える必要も
ない*はず*です。しかしそのようなチェックは膨大な処理が必要なうえ、別パッケージ
からのエラーを生み出して精神を破壊します。Rustは全ての借用を関数ごとに独立に
チェックし、全てのエラーをローカルからのものにとどめています（さもなければ
あなたが宣言した型のシグネチャが誤っています）。

そういえば過去のコードで、借用を関数のシグネチャに書いたのにライフタイムを指定して
いないことがありましたね。実はあれはOKです！これはRustがライフタイムを自動で割り当てて
くれるよくあるケースで、*ライフタイムの省略*と呼ばれています。

具体的にいうとこうです：

```rust ,ignore
// Only one reference in input, so the output must be derived from that input
fn foo(&A) -> &B; // sugar for:
fn foo<'a>(&'a A) -> &'a B;

// Many inputs, assume they're all independent
fn foo(&A, &B, &C); // sugar for:
fn foo<'a, 'b, 'c>(&'a A, &'b B, &'c C);

// Methods, assume all output lifetimes are derived from `self`
fn foo(&self, &B, &C) -> &D; // sugar for:
fn foo<'a, 'b, 'c>(&'a self, &'b B, &'c C) -> &'a D;
```

で`fn foo<'a>(&'a A) -> &'a B`は何を*意味する*のでしょうか？これは要するに
fooの引数は少なくとも返り値より長いライフタイムを持たなくてはいけないことを
意味しています。つまり、fooの返り値をずっと引き回した場合、引数に与えた
借用もずっと生存していなくてはいけません。返り値を使うのをやめたとき、
コンパイラは引数も解放して大丈夫と判断します。

この仕組みがあることで、なにかのメモリ割り当てが解放された後に使われることも、
なにかが借用されているのに変更されることも防ぐことができます。逆に言えば
この仕組みを守るだけでそれを達成できるのです！

さて、ではIterに話を戻しましょう。

ライフタイムがないバージョンに戻しましょう：

```rust ,ignore
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

ライフタイムをつけなくてはいけないのは関数と型のシグネチャです：

```rust ,ignore
// Iter is generic over *some* lifetime, it doesn't care
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

// No lifetime here, List doesn't have any associated lifetimes
impl<T> List<T> {
    // We declare a fresh lifetime here for the *exact* borrow that
    // creates the iter. Now &self needs to be valid as long as the
    // Iter is around.
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

// We *do* have a lifetime here, because Iter has one that we need to define
impl<'a, T> Iterator for Iter<'a, T> {
    // Need it here too, this is a type declaration
    type Item = &'a T;

    // None of this needs to change, handled by the above.
    // Self continues to be incredibly hype and amazing
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

おし、今回はうまく行くと思います。

```text
cargo build

error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`
```

(╯°□°)╯︵ ┻━┻

OK。ライフタイムのエラーは直りましたが別の型エラーが出てきました。

`&Node`を保存したいのに`&Box<Node>`が返っていると。OK。簡単です。借用の前にBox
を外せばいいわけですね：

```rust ,ignore
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &*node);
            &node.elem
        })
    }
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:77:43
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                                           ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                      ^^^^^^^^^ cannot move out of borrowed content

error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:85:46
   |
85 |             self.next = node.next.map(|node| &*node);
   |                                              ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &*node);
   |                         ^^^^^^^^^ cannot move out of borrowed content
```

(ﾉಥ益ಥ）ﾉ﻿ ┻━┻

`as_ref`を忘れてました。そのせいでBoxを`map`にムーブしてしまい、値がドロップされ、
借用がもとの値を失ってしまったのです：

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_ref().map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_ref().map(|node| &*node);
            &node.elem
        })
    }
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.as_ref().map(|node| &*node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.as_ref().map(|node| &*node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

```

😭

`as_ref`を使ったのでもう一段参照外しが必要になりました：


```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_ref().map(|node| &**node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_ref().map(|node| &**node);
            &node.elem
        })
    }
}
```

```text
cargo build

```

🎉 🎉 🎉

きっと「うわ、この`&**`とかいうのはマジでカスだな」と思ってることでしょう。あなたの
感性は正しいです。通常Rustは*参照外し型強制*という機能でこういった操作を暗黙に
やってくれます。参照外し型強制がすることは、基本的には\*をコード中に入れて型が
合うようにすることです。借用チェッカがあるため、ポインタがぐちゃぐちゃになって
いないことが分かっているのでこういうことができるのです！

しかし、今回の場合`&T`ではなく`Option<&T>`を使っていますね。これは参照外し型強制
が適用されるには複雑すぎるのです。私の経験上、このような形は幸運にもあまりありません。

完璧を期すために、*ターボフィッシュ*を使って*さらに*型のヒントを与えることができます：

```rust ,ignore
    self.next = node.next.as_ref().map::<&Node<T>, _>(|node| &node);
```

mapの実装を見ると、このようにジェネリックになっています：

```rust ,ignore
pub fn map<U, F>(self, f: F) -> Option<U>
```

ターボフィッシュ（`::<>`）によって、コンパイラにmapがどの型を使えばいいか
教えることができます。この場合`::<&Node<T>, _>`によって、「`&Node<T>`を返してね、
でも他の型はどうでもいい」ということを言っています。

すると今度はコンパイラは`&node`に参照外し型強制が適用できることがわかるので
いちいち\*をつける必要がなくなります！

でもこれはあんまり大きな進歩とは言えなそうですね。参照外し型強制とたまには使える
ターボフィッシュを使いたい言い訳です😅

テストを書いて、動かなくなったりとにかく悪いことが起こったりしていないことを確認
しましょう：

```rust ,ignore
#[test]
fn iter() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&1));
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 5 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured

```

イエ〜ィ。

最後に、ここのライフタイムを省略できることに触れておきます：

```rust ,ignore
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_ref().map(|node| &**node) }
    }
}
```

これは次のコードと同じです：

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.as_ref().map(|node| &**node) }
    }
}
```

やった―！ライフタイムが減りました！

もしライフタイムが実際にはあるのに省略することが気になるのであれば、
Rust2018の新機能、「ライフタイム省略明示表記」、`'_`を使うこともできます：

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_ref().map(|node| &**node) }
    }
}
```
