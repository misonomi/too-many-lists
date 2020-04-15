# Drop

スタックを作りましたし，pushとpopもできるようになりましたし，テストまで実装して
ちゃんと動くこともわかりました！

リストの後片付けをするコードを書く必要があるでしょうか？技術的には
ノーです．全く必要ありません！C++のように，Rustはデストラクタを使って使い終わった
リソースを自動的に解放してくれます．Dropという名前の*トレイト*を実装することで
デストラクタを型に与えることができます．Rustではインターフェースをトレイト
と呼びます．Dropはこんな感じのインターフェースです：

```rust ,ignore
pub trait Drop {
    fn drop(&mut self);
}
```

基本的には，スコープの外に出たときに後片付けのためにdropが実行されます．

実はDropを実装してある型を含む型に対してはDropを実装しなくてもよく，その
Dropが実装してある型のデストラクタを呼べばいいだけです．私達が実装したListの場合
デストラクタではheadを消せばいいだけですから，何も実装しなくても`Box<Node>`を
消してくれる*はず*です．全ては自動的に処理されます…が，一つだけ問題があります．

その自動処理はクソです．

こんな感じの簡単なリストを考えてみましょう：


```text
list -> A -> B -> C
```

`list`がdropされるとき，`list`はAをdropしようとし，AはBを，BはCをdropしようとします．
不安になってきた人もいますよね？これは再帰なので，スタックを消費し尽くしてしまう
可能性があります！

「これは末尾再帰だから，ちゃんとした言語ならスタックが尽きないようにしてくれるだろ」
と思った人もいるでしょう．それは，実は，間違いです！理由を知るために，コンパイラが
実際にするであろう処理をdropとして実装してみましょう：


```rust ,ignore
impl Drop for List {
    fn drop(&mut self) {
        // NOTE: you can't actually explicitly call `drop` in real Rust code;
        // we're pretending to be the compiler!
        self.head.drop(); // tail recursive - good!
    }
}

impl Drop for Link {
    fn drop(&mut self) {
        match *self {
            Link::Empty => {} // Done!
            Link::More(ref mut boxed_node) => {
                boxed_node.drop(); // tail recursive - good!
            }
        }
    }
}

impl Drop for Box<Node> {
    fn drop(&mut self) {
        self.ptr.drop(); // uh oh, not tail recursive!
        deallocate(self.ptr);
    }
}

impl Drop for Node {
    fn drop(&mut self) {
        self.next.drop();
    }
}
```

Boxを`deallocate`した*後に*中身をdropすることは*できません*．したがって末尾再帰で
このListをdropすることはできないのです！なのでBoxからNodeを取り出してdropする
反復処理を書かなくてはいけません．


```rust ,ignore
impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, Link::Empty);
        // `while let` == "do this thing until this pattern doesn't match"
        while let Link::More(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, Link::Empty);
            // boxed_node goes out of scope and gets dropped here;
            // but its Node's `next` field has been set to Link::Empty
            // so no unbounded recursion occurs.
        }
    }
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

```

いいですね！

----------------------

<span style="float:left">![Bonus](img/profbee.gif)</span>

## おまけ：早期最適化

私達が実装した`drop`は`while let Some(_) = self.pop() { }`に*とても*よく
似ていますし，そう書いたほうが簡潔です．これらの違いはなんでしょうか？
また，リストにint以外を入れた場合パフォーマンスにどのように影響する
でしょうか？

<details>
  <summary>クリックして答えを見る</summary>

`pop`は`Option<i32>`を返しますが，私達の実装ではLink（`Box<Node>`）に対する操作を行います．つまり，私達の実装はNodeのポインタを
動かすだけであるのに対し，popはNodeの値をムーブするのです．もしListを一般化してDrop実装済みめちゃデカ型（VeryBigThingWithADropImpl
　略して　VBTWADI）も入れられるようにしたとき，これは超コストのかかる操作になるおそれがあります．しかし，Boxは内容物のDropをそのまま
呼べるのでこの問題を回避することができます．VBTWADIを入れることこそが*まさしく*配列ではなく連結リストを使うメリットなので，VBTWADIを
使うときパフォーマンスが良くなかったらちょっと残念ですよね．

両方の実装のいいとこ取りをしたいなら，新しく`fn pop_node(&mut self) -> Link`というメソッドを
作り，これを使って`pop`と`drop`を実装するのがいいでしょう．

</details>
