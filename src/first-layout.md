# 基本データ設計

よし，で，連結リストってなんでしょう？えー，基本的にはヒープに割り当てられたデータの山で
（カーネル勢はちょっと黙っててください），それぞれが順番に互いのポインタを持っています．
連結リストは，手続き型言語のプログラマが10m以内に近づかないものであり関数型言語の
プログラマがあらゆる用途に使うものです．それなら，関数型言語のプログラマにどんなものか
説明してもらうのがよさそうですね．彼らはこんな感じの定義を寄越すでしょう：

```haskell
List a = Empty | Elem a (List a)
```

これはだいたい「リストは空か，リストにリンクした要素である」みたいな感じに読めます．
*直和型*を用いた再起的な定義ですね．直和型というのは「違う型の値を持つことができる型」
のことです．Rustでは直和型は`enum`に相当します！もしCライクな言語に慣れているなら，
Rustのenumはあなたが大好きなあのenumそのものです．じゃあ上の関数型の定義をRustっぽく
書き換えてみましょう！

とりあえず話を簡単にするためにジェネリクスは使わず，32bit intだけを要素として
持つことにします：

```rust ,ignore
// in first.rs

// pub says we want people outside this module to be able to use List
pub enum List {
    Empty,
    Elem(i32, List),
}
```

ぬわ疲．それじゃコンパイルしましょう：

```text
> cargo build

error[E0072]: recursive type `first::List` has infinite size
 --> src/first.rs:4:1
  |
4 | pub enum List {
  | ^^^^^^^^^^^^^ recursive type has infinite size
5 |     Empty,
6 |     Elem(i32, List),
  |               ---- recursive without indirection
  |
  = help: insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make `first::List` representable
```

えっと，あなたはどうか分かりませんが私は関数型界隈に裏切られた気分です．

エラーメッセージを見てみると（裏切られた絶望を乗り越えたあとで），rustcはまさに
エラーを解決する方法を示しています：

> insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make `first::List` representable

おっけー，`box`ね．何それ？ググってみましょう．`rust box`...

> [std::boxed::Box - Rust](https://doc.rust-lang.org/std/boxed/struct.Box.html)

これかな...

> `pub struct Box<T>(_);`
>
> A pointer type for heap allocation.
> See the [module-level documentation](https://doc.rust-lang.org/std/boxed/) for more.

*リンクをクリックする*

> `Box<T>`, casually referred to as a 'box', provides the simplest form of heap allocation in Rust. Boxes provide ownership for this allocation, and drop their contents when they go out of scope.
>
> Examples
>
> Creating a box:
>
> `let x = Box::new(5);`
>
> Creating a recursive data structure:
>
```
#[derive(Debug)]
enum List<T> {
    Cons(T, Box<List<T>>),
    Nil,
}
```
>
```
fn main() {
    let list: List<i32> = List::Cons(1, Box::new(List::Cons(2, Box::new(List::Nil))));
    println!("{:?}", list);
}
```
>
> This will print `Cons(1, Box(Cons(2, Box(Nil))))`.
>
> Recursive structures must be boxed, because if the definition of Cons looked like this:
>
> `Cons(T, List<T>),`
>
> It wouldn't work. This is because the size of a List depends on how many elements are in the list, and so we don't know how much memory to allocate for a Cons. By introducing a Box, which has a defined size, we know how big Cons needs to be.

わお，うん．多分今まで見た中で一番知りたいことが書いてあるドキュメントですね．
ドキュメントのまさに最初に書いてあるのは*私達がやろうとしていたことそのものと，
なぜそれが動かないのかと，どうやって直すか*です．

ドキュメントが強すぎる．

よし，じゃあやってみましょう：

```rust ,ignore
pub enum List {
    Empty,
    Elem(i32, Box<List>),
}
```

```text
> cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

やった，ビルドが通りました！

...でもちょっと考えてみるとこれはかなりバカっぽいリストです．

要素が2つあるリストを考えてみましょう．

```text
[] = スタック
() = ヒープ

[要素A, ポインタ] -> (要素B, ポインタ) -> (Empty *ゴミ*)
```

問題は2つあります：

* 最後のノードは「僕は実はノードじゃないヨ」を意味するデータを割り当てられている．
* 最初のノードは全くヒープのメモリ空間を割り当てられていない．

表面的には，これらは互いに打ち消し合っているように見えます．一方で追加のノードを
割当て，もう一方では割り当てていない...でも次のようなデータ構造を見てみましょう：

```text
[ポインタ] -> (要素A, ポインタ) -> (要素B, *null*)
```

この構成では要素数に関わらず各要素のためにヒープのメモリ空間を割り当てます．
重要な違いは1つ目の設計にあった*ゴミ*がないことです．このゴミはなんなのでしょうか？
それを知るために，enumがどのようにメモリに割り当てられるか見てみましょう．

一般にenumはこんな感じです：

```rust ,ignore
enum Foo {
    D1(T1),
    D2(T2),
    ...
    Dn(Tn),
}
```

Fooはどの*列挙子*（`D1`, `D2`, .. `Dn`）であるかを表す整数をもつ必要があります．これが
enumの*タグ*です．それに加え，`T1`, `T2`, .. `Tn`のうち*最大の*ものが入るメモリ空間が
必要になります（あとメモリの整列のためのパディング）．

ここで発生している大いなるムダは，`Empty`がたった1bitの情報でもポインタ1個と要素1個分の
メモリが必要であることです．`Elem`にいつ変換されてもいいようにしなくてはならないのです．
そんなわけで1つ目の設計ではゴミデータを割り当てざるを得ず，2つ目の構成より若干大きい
スペースが必要になります．

ノードの一つがヒープにないのも，多分ビビるほど，常に全ノードがヒープに割り当てられるより
*悪い*です．ノードの扱いが統一的でないと，pushやpopではそれほど困らないかもしれませんが
分割や結合の際に厄介なことになります．

それぞれの設計でリストを分割することを考えてみましょう：

```text
設計1:

[要素A, ポインタ] -> (要素B, ポインタ) -> (要素C, ポインタ) -> (Empty *ゴミ*)

Cを分割:

[要素A, ポインタ] -> (要素B, ポインタ) -> (Empty *ゴミ*)
[要素C, ポインタ] -> (Empty *ゴミ*)
```

```text
設計2:

[ポインタ] -> (要素A, ポインタ) -> (要素B, ポインタ) -> (要素C, *null*)

Cを分割:

[ポインタ] -> (要素A, ポインタ) -> (要素B, *null*)
[ポインタ] -> (要素C, *null*)
```

設計2では要素Bのポインタをスタックにコピーして，元あった場所をnullにするだけです．
設計1でも同じことをしていますが，要素Cをヒープからスタックにコピーする必要があります．
結合は同じことを逆にやるだけです．

連結リストの数少ない良いところのひとつに要素をノード自体の中で組み立て，メモリ空間を
移動させずに順番を入れ替える事ができる点があります．ポインタをいじるだけでリスト内の
順番が動くのです．設計1はこの特長を持ちません．

はい，設計1がクソなことが証明されました．ではどう書き直せばいいでしょうか？じゃあこんな
感じで：

```rust ,ignore
pub enum List {
    Empty,
    ElemThenEmpty(i32),
    ElemThenNotEmpty(i32, Box<List>),
}
```

悪化してるように見えたなら良かったです．特筆すべき点は，この設計は`ElemThenNotEmpty(0, Box(Empty))`
というまったく無意味な状態を持てるようになっており，ロジックを複雑にしていることです．
しかも相変わらずノードがスタックにあったりヒープにあったりします．

とはいえ，この設計は*ひとつ*興味深い特徴を持っています：Emptyの状態がなく，ヒープの
割当が1要素分小さいことです．不幸にもそれによって*更に大きい*メモリ空間を無駄にしている
んですけどね！なぜかというと，先程の設計が*ヌルポインタ最適化*を行っているからです．

さっきenumはどの列挙子を持つかを管理するタグを持っていると言いました．しかし，このような
特別な形のenumについては：

```rust,ignore
enum Foo {
    A,
    B(ContainsANonNullPtr),
}
```

ヌルポインタ最適化が行われ，*タグのためにはメモリ空間が使われなくなります*．列挙子Aのとき
enumはすべて`0`が割り当てられ，そうでないとき列挙子Bが割り当てられます．列挙子Bはnullでない
ポインタを含むため，中身が全て`0`ということはあり得ないのでこういう事ができるのです．
かっこいい！

他にこういう最適化ができるenumの形はどんなのがあるか思いつきますか？実はいっぱいあります！
これがRustがenumのメモリレイアウトを定義しない理由です．他にもいくつか複雑なRustの
enumメモリレイアウト最適化戦略がありますが，ヌルポインタ最適化が明らかに一番重要です！
ヌルポインタ最適化によって`&`, `&mut`, `Box`, `Rc`, `Arc`, `Vec`などを`Option`
に入れてもオーバーヘッドが発生しません（これらの型についてはおいおい触れます）．

で，どうすればゴミデータの割当を防ぎ，統一的にメモリを割り当たうえでヌルポインタ最適化
の恩恵を得ることができるでしょうか？要素を持つということとリストを新しく割り当てるという
ことを切り離して考えるほうがよさそうです．そのためにはCライクな構造体についてもう少し
考えなくてはいけません！

enumがいくつかある値の*ひとつ*を保持する型なら，構造体は*複数*を一度に保持する型です．
リストを2つの型に分割しましょう．ListとNodeです．

さっきと同様Listは空（Empty）かリストが後ろにくっついた要素です．「リストが後ろに
くっついた要素」を別の型で表現することでBoxをさらに最適化された状態にもっていく
ことができます：

```rust ,ignore
struct Node {
    elem: i32,
    next: List,
}

pub enum List {
    Empty,
    More(Box<Node>),
}
```

ちゃんとできてるか確認してみましょう：

* リストの末尾にゴミがない：OK!
* `enum`がヌルポインタ最適化されている：OK!
* ノードが統一的に割り当てられる：OK!

よし！これで問題があった（ことがRustの公式ドキュメントに指摘された）はじめの設計
の問題点を克服しました．

```text
> cargo build

warning: private type `first::Node` in public interface (error E0446)
 --> src/first.rs:8:10
  |
8 |     More(Box<Node>),
  |          ^^^^^^^^^
  |
  = note: #[warn(private_in_public)] on by default
  = warning: this was previously accepted by the compiler but
    is being phased out; it will become a hard error in a future release!
```

:(

またRustに怒られました．`List`を（外部から使ってほしいので）publicにしましたが`Node`
をpublicにはしていませんでした．問題は，publicな`enum`の中は全てpublicでなくてはならないことです．
`Node`をpublicにすることもできますが，一般にRustでは実装の詳細を隠蔽することが好まれます．
`List`を構造体にすることで実装の詳細を隠すことにしましょう：


```rust ,ignore
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
```

`List`はフィールドがひとつしかない構造体なので，サイズはフィールドと同じになります．
ゼロコスト抽象化です イエーイ！

```text
> cargo build

warning: field is never used: `head`
 --> src/first.rs:2:5
  |
2 |     head: Link,
  |     ^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: variant is never constructed: `Empty`
 --> src/first.rs:6:5
  |
6 |     Empty,
  |     ^^^^^

warning: variant is never constructed: `More`
 --> src/first.rs:7:5
  |
7 |     More(Box<Node>),
  |     ^^^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/first.rs:11:5
   |
11 |     elem: i32,
   |     ^^^^^^^^^

warning: field is never used: `next`
  --> src/first.rs:12:5
   |
12 |     next: Link,
   |     ^^^^^^^^^^

```

よっしゃ，コンパイルが通りました！Rustは私達が作ったものが完全に無駄だといて
めちゃくちゃ怒ってます．`head`をどこでも使ってないし，プライベートフィールドなので
外部からも参照できないからです．したがって`Link`と`Node`も同様に無意味というわけです．
というわけで次はこれを解決しましょう．このリストを使うコードを実装していきましょう！
