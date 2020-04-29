# イテレート

この悪い子をイテレートしていきましょう．

## IntoIter

いつものようにIntoIterが一番楽です．スタックでラップして`pop`を呼びましょう：

```rust ,ignore
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<T> {
        self.0.pop_front()
    }
}
```

でもちょっとおもしろいことが起こっています．前はリストをイテレートする「自然な」方向が
一意に決まっていましたが，両端キューには2つあります．もし逆方向にイテレートしたい人が
いたらどうしたらいいでしょうか？

じつはRustにはそのための`DoubleEndedIterator`トレイトがあります．DoubleEndedIterator
はIteratorを*継承*し（これは全てのDoubleEndedIteratorがIteratorsであることを意味します），
`next_back`というメソッドの実装を必要とします．このメソッドは`next`と全く同じシグネチャ
を持ちますが，逆側の要素を返すようになっています．DoubleEndedIteratorを使えば簡単に
イテレータを両端キューにして，イテレータが空になるまで前からでも後ろからでも要素を
取り出すことができます．

`next_back`はDoubleEndedIteratorを使う人にとってそれほど重要なメソッドではありません．
それより，イテレータを逆順にして返す`rev`メソッドがあることのほうが重要です．この
メソッドのやっていることはとてもシンプルです．逆順になったイテレータで呼ばれる`next`は
代わりに`next_back`が呼ばれるというだけです．

なんにせよ私達のリストはすでに両端キューなので，これの実装は超簡単です：

```rust ,ignore
impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<T> {
        self.0.pop_back()
    }
}
```

そしてテストします：

```rust ,ignore
#[test]
fn into_iter() {
    let mut list = List::new();
    list.push_front(1); list.push_front(2); list.push_front(3);

    let mut iter = list.into_iter();
    assert_eq!(iter.next(), Some(3));
    assert_eq!(iter.next_back(), Some(1));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next_back(), None);
    assert_eq!(iter.next(), None);
}
```


```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 11 tests
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test fourth::test::into_iter ... ok
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test third::test::iter ... ok
test third::test::basics ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok

test result: ok. 11 passed; 0 failed; 0 ignored; 0 measured

```

イエーイ．

## Iter

Iterはもうすこし厳しいです．またあの恐ろしい`Ref`と向き合わなくてはいけません！
Refのせいでこれまでのように`&Node`を保持するわけにはいきません．代わりに`Ref<Node>`
を保持します：

```rust ,ignore
pub struct Iter<'a, T>(Option<Ref<'a, Node<T>>>);

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter(self.head.as_ref().map(|head| head.borrow()))
    }
}
```

```text
> cargo build

```

ここまでは順調です．`next`の実装はちょっと難しいですが，これまでのIterMutの実装に
RefCellの狂気を添えた感じで行けると思います：

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = Ref<'a, T>;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.take().map(|node_ref| {
            self.0 = node_ref.next.as_ref().map(|head| head.borrow());
            Ref::map(node_ref, |node| &node.elem)
        })
    }
}
```

```text
cargo build

error[E0521]: borrowed data escapes outside of closure
   --> src/fourth.rs:155:13
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- `self` is declared here, outside of the closure body
154 |         self.0.take().map(|node_ref| {
155 |             self.0 = node_ref.next.as_ref().map(|head| head.borrow());
    |             ^^^^^^   -------- borrow is only valid in the closure body
    |             |
    |             reference to `node_ref` escapes the closure body here

error[E0505]: cannot move out of `node_ref` because it is borrowed
   --> src/fourth.rs:156:22
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- lifetime `'1` appears in the type of `self`
154 |         self.0.take().map(|node_ref| {
155 |             self.0 = node_ref.next.as_ref().map(|head| head.borrow());
    |             ------   -------- borrow of `node_ref` occurs here
    |             |
    |             assignment requires that `node_ref` is borrowed for `'1`
156 |             Ref::map(node_ref, |node| &node.elem)
    |                      ^^^^^^^^ move out of `node_ref` occurs here
```

クソが．

`node_ref`のライフタイムが十分ではないようです．普通の参照と違って，Refをこんなふうに
分割することはできないようです．`head.borrow()`でとったRefは`node_ref`が生きている
間しか生きられませんが，私達は`Ref::map`で`node_ref`を捨てています．

偶然にも私がこれを書いているとき，私達が使いたい関数が2日前に安定化されました．つまり
あと数ヶ月でstableリリースに含まれることになります．なので最新のnightlyビルドを
使っていきましょう[^1]：

```rust ,ignore
pub fn map_split<U, V, F>(orig: Ref<'b, T>, f: F) -> (Ref<'b, U>, Ref<'b, V>) where
    F: FnOnce(&T) -> (&U, &V),
    U: ?Sized,
    V: ?Sized,
```

ウーッ．やってみましょう...

```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.0.take().map(|node_ref| {
        let (next, elem) = Ref::map_split(node_ref, |node| {
            (&node.next, &node.elem)
        });

        self.0 = next.as_ref().map(|head| head.borrow());

        elem
    })
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
error[E0521]: borrowed data escapes outside of closure
   --> src/fourth.rs:159:13
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- `self` is declared here, outside of the closure body
...
159 |             self.0 = next.as_ref().map(|head| head.borrow());
    |             ^^^^^^   ---- borrow is only valid in the closure body
    |             |
    |             reference to `next` escapes the closure body here
```

えー．ライフタイムのつじつまを合わせるためにもう1回`Ref::Map`する必要がありますが，
私達が欲しいのは`Option<Ref>`であって`Ref::Map`から返る`Ref`ではありません．しかし
OptionをmapするにはRefの中を見る必要があります...

**（しばらく虚空を見つめる）**

??????

```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.0.take().map(|node_ref| {
        let (next, elem) = Ref::map_split(node_ref, |node| {
            (&node.next, &node.elem)
        });

        self.0 = if next.is_some() {
            Some(Ref::map(next, |next| &**next.as_ref().unwrap()))
        } else {
            None
        };

        elem
    })
}
```

```text
error[E0308]: mismatched types
   --> src/fourth.rs:162:22
    |
162 |                 Some(Ref::map(next, |next| &**next.as_ref().unwrap()))
    |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `fourth::Node`, found struct `std::cell::RefCell`
    |
    = note: expected type `std::cell::Ref<'_, fourth::Node<_>>`
               found type `std::cell::Ref<'_, std::cell::RefCell<fourth::Node<_>>>`
```

あーそうですね．RefCellが二重になっています．リストを深く探索すればするほどRefCellが
ネストしていってしまいます．複数の参照を一気にカウントしてくれるRefみたいなものが必要
です．リストの要素を解放したとき，それまでネストしてきた参照カウントを一気に1つづつ
デクリメントしてくれるような何か.............

もうなにか手が残っているとは思えません．終わりです．RefCellから離れて考えてみましょう．

Rcを使うというのはどうでしょう．参照を保持する必要なんかありませんよね？単にRcを
Cloneして所有権を管理させるのではダメでしょうか？

```rust
pub struct Iter<T>(Option<Rc<Node<T>>>);

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter(self.head.as_ref().map(|head| head.clone()))
    }
}

impl<T> Iterator for Iter<T> {
    type Item =
```

えっと...待ってください，ここの型は何でしょう？`&T`？`Ref<T>`？

だめです，どっちもうまくいきません...私達のIterはもはやライフタイムを持っていないのです！
`&T`も`Ref<T>`も`next`する際にライフタイムが必要です．でもRcから出そうとするものは
Iteratorを借用します...ウッ，頭が...痛い...ぐあああああああああ

多分Rcを...mapして...`Rc<T>`にする？本当か？Rcのドキュメントを見る限りそういうことを
するメソッドはなさそうです．実際これをできるようにする[クレート][own-ref]を作った人がいます．

でも待ってください．かりに*それ*をやったとしてもさらに大きな問題に直面します．イテレータ
破壊という恐ろしい現象です．これまで私達はイテレータ破壊に対して完全に無敵でした．
なぜならIterはリストを借用するだけなので，それに変更を加えないからです．しかし，もし
IterがRcを返すならそれは借用に留まりません！これによってリストの利用者はポインタを持ち
続ける限り`push`や`pop`を呼べてしまえます！

おお神よ！いったいどうすればいいのですか！？

えっと，実はpushは無問題です．私達の見えている範囲よりリストの全体が大きくなる
だけです．大したことありません．

でも`pop`は話が別です．もし私達の知らないところでpopが行われても*まだ*大丈夫です．
もともとそのノードは私達の知らない所にあるので何も起こりません．しかし，もし私達
に見えているノードがpopされれば...全てがぶっ壊れます！具体的には`try_unwrap`の
結果を`unwrap`しようとしたときにpanicするでしょう．

これは実際かなりイケてます．リストを指すポインタを持つイテレータをいくらでも生成
できて，他が指しているノードを削除しようとしない限り*ともかく正常に動作する*のですから．
しかもそのような削除が発生したときにも，ダングリングポインタを生むこともなくしっかり
パニックしてくれるのです！

でもRcのマップをするためにイテレータ破壊に対処しなくてはいけないのはなんというか...
よくありません．私達は`Rc<RefCell>`に完膚無きまでに絶望させられました．面白いことに
いま私達は永続スタックのときの逆を体験しています．永続スタックのとき，私達はデータの
所有権を得ようと苦闘しましたが参照はほぼいつでも得ることができました．今回は所有権
を得るのは問題ありませんでしたが参照を貸し出すのに苦労しています．

とはいえ，公平に言って，私達が苦労している理由は実装の詳細を隠蔽し洗練されたAPIを
提供しようとしているからと言えるでしょう．もしNodeをどこにでも渡すようにすれば
すべてうまくやれたでしょう．

そうすればランタイムに同時に同じ要素が変更されないことを保証する並列IterMutすら
実装することができます！

本当にこの設計は内部のデータ構造を隠蔽するAPIに向いてませんね．内部可変性は安全な
*アプリケーション*を書くには適していますが，安全な*ライブラリ*を書くためには
それほど有用ではありません．

ともかくIterとIterMutは諦めます．実装できることはできますが，やりたくありません．

[own-ref]: https://crates.io/crates/owning_ref

[^1]: 訳注：これは原文が書かれた当時の話で，当該関数はとっくにstableに入っています．したがってこのためにnightlyを使う必要はありません