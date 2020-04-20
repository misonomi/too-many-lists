# IterMut

正直に言います．IterMutは*治安が悪い*です．この言葉自体が治安悪いですが，
IterMutはIterと明らかに同じものです！

意味的にはそうですが，参照の基本に忠実に実装するとIterMutはガチの魔法になり，
それに比べればIterは児戯に等しいと言えます．

私達がIterのために実装したIteratorに注目してください：

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> { /* stuff */ }
}
```

これはこのように書きかえることができます：

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next<'b>(&'b mut self) -> Option<&'a T> { /* stuff */ }
}
```

`next`の入力のライフタイムと出力のライフタイムの間には*何の*関係もありません．
なぜそんなことを気にするのでしょう？これのおかげで`next`を無条件に呼びまくる
ことができるからです！


```rust ,ignore
let mut list = List::new();
list.push(1); list.push(2); list.push(3);

let mut iter = list.iter();
let x = iter.next().unwrap();
let y = iter.next().unwrap();
let z = iter.next().unwrap();
```

いいですね！

共有参照を使うなら*間違いなく*これでOKです．共有参照はいくつでも持つことができるからです．
しかし可変参照は同時に複数存在できません．

結局安全なコードを使う限り（この言葉の意味はまだ分かりませんが...）IterMutを実装するのは
とてつもなく困難なのです．ところが驚くべきことにIterMutは大抵のstructに対し全くもって
安全に実装することができます！

とりあえずIterのコードをコピーしてきて，全部可変にするところから始めましょう：

```rust ,ignore
pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn iter_mut(&self) -> IterMut<'_, T> {
        IterMut { next: self.head.as_mut().map(|node| &mut **node) }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_mut().map(|node| &mut **node);
            &mut node.elem
        })
    }
}
```

```text
> cargo build
error[E0596]: cannot borrow `self.head` as mutable, as it is behind a `&` reference
  --> src/second.rs:95:25
   |
94 |     pub fn iter_mut(&self) -> IterMut<'_, T> {
   |                     ----- help: consider changing this to be a mutable reference: `&mut self`
95 |         IterMut { next: self.head.as_mut().map(|node| &mut **node) }
   |                         ^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be borrowed as mutable

error[E0507]: cannot move out of borrowed content
   --> src/second.rs:103:9
    |
103 |         self.next.map(|node| {
    |         ^^^^^^^^^ cannot move out of borrowed content
```

この上下のエラーは別々のもののようです．1つ目はどうやって直すか書いてありますし簡単そうですね！
共有参照を可変参照に昇格させることはできないので`iter_mut`は`&mut self`を引数にとらなくては
いけません．ただのコピペミスですね．

```rust ,ignore
pub fn iter_mut(&mut self) -> IterMut<'_, T> {
    IterMut { next: self.head.as_mut().map(|node| &mut **node) }
}
```

もう一つのエラーはなんでしょうか？

おっと！前章の`iter`のコードにバグがあったものの，幸いにも動いていたようです！

私達はCopyの魔法を初めて目の当たりにしています．私達が[所有権][ownership]を使い始めたとき，
ムーブされてしまったものは使えないと言いました．幾つかの型に対しては，これは筋の通った
挙動です．かしこいBoxくんはムーブした変数のヒープの割当を2つに増やしたりはしません．

しかし他の型については，この挙動は*ゴミ*です．例えば整数はただの数であり，所有権もクソも
ありません．そこで，整数型はCopy型のひとつに入れられています．Copy型はビットごとのコピーに
よって元通りコピーできる型を指し，ムーブされたときでも元の変数を依然使用できるという強力な
特徴を持っています．同様にCopy型は，`mem::replace`などで代わりを用意しなくても参照から
取り出すことができるのです！

Rustの数値プリミティブ型（i32, u64, bool, f32, char, などなど）はCopy型です．
他の型も，構成要素が全てCopy型である限りCopy型にすることができます．

なぜIterのコードが動いていたかといえば，共有参照もCopy型だからです．そして`&`がCopy型なので
`Option<&>`もCopy型です．そして`self.next.map`したとき，Optionがコピーされるので
今回私達が出会ったエラーは出なかったというわけです．しかし今回はCopy型ではない`&mut`を
使っているため（もし&mutをコピーできたら同じメモリアドレスに2つの&mutを持つことになって
しまいます），`take`でOptionの中身を取らないといけません．


```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.next.take().map(|node| {
        self.next = node.next.as_mut().map(|node| &mut **node);
        &mut node.elem
    })
}
```

```text
> cargo build

```

えー...と，やりました！IterMutができました！

テストしてみましょう：


```rust ,ignore
#[test]
fn iter_mut() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.iter_mut();
    assert_eq!(iter.next(), Some(&mut 3));
    assert_eq!(iter.next(), Some(&mut 2));
    assert_eq!(iter.next(), Some(&mut 1));
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 6 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::peek ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured

```

やった．うごいてますね．

マジか．

どういうことなの...

まあこれが動く*ように*実装してはいました．でもたいてい何かに邪魔されるんです！
ひとつはっきりさせておきましょう：

片方向連結リストの要素をひとつずつ，その可変参照を取得するコードを実装しました．そして
それが動くことも確認しました．コードは安全です．なにも治安の悪いことをしてません．

私に言わせれば，これは結構すごいことです．うまく行った理由はいくつかあります：

* 私達は`Option<&mut>`を`take`したので，排他的な可変参照を得ることができました．
  これによって複数回参照される心配はなくなりました．
* Rustは，可変参照であるstructのフィールドを切り離しても大丈夫なことをわかっています．
  切り離されたフィールドから親をたどる手段がないからです．

これらの事実から，今回IterMutを実装した設計を流用して安全な配列やツリーも実装できる
事がわかります！イテレータを双方向にして前と後ろから同時にイテレートすることすらできます！すごい！

[ownership]: first-ownership.md
