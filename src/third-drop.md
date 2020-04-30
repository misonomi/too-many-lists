# Drop

可変のリスト同様，このリストもdropが再帰する問題を抱えています．確かに
今回はそれほど大きな問題ではありません．リストの途中の*どこか*で他のリストに
参照されているノードがあればそこでdropが止まるからです．とはいえ，依然として
対処すべき問題であり，どうすればいいかは自明ではありません．前回はこうしました：

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

問題はループの中のところです：

```rust ,ignore
cur_link = boxed_node.next.take();
```

これはBoxの中のNodeを変更していますが，今回はRcを使っているのでそれはできません．
他のRcがこの同じノードを参照している可能性があり，私達は共有参照しか持っていません．

しかし，もし他の参照がないことがわかっていれば中のNodeをRcの外に出しても大丈夫な
*はず*です．そして，もし大丈夫でないとき，それはdropを止めるときであるという
ことを意味しています．

刮目してください．Rcにはまさにそのためのメソッド，`try_unwrap`があります：

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut head = self.head.take();
        while let Some(node) = head {
            if let Ok(mut node) = Rc::try_unwrap(node) {
                head = node.next.take();
            } else {
                break;
            }
        }
    }
}
```

```text
cargo test
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/too-many-lists/lists)
    Finished dev [unoptimized + debuginfo] target(s) in 1.10s
     Running /Users/ABeingessner/dev/too-many-lists/lists/target/debug/deps/lists-86544f1d97438f1f

running 8 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

やった！
勝ち．
