# Unsafe Rust

これは深刻で，大きく，複雑で，危険な話題です．あまりに深刻なので
私はもう一冊[これについての本][nom]を書きました．

とにもかくにも，*どんな*言語も他の言語を呼び出した途端不安全になります．呼び出したCが適当に
悪いことをするのを許さなくてはなりませんから．Javaも，Pythonも，Rubyも，Haskellも...
FFI（Foreign Function Interface）の前ではあらゆる言語が不安全になります．

Rustはこの事実を受け止めるために自分自身を2つの言語に分けました．安全なRustと，不安全な
Rustです．ここまで私達は安全なRustだけを扱ってきました．安全なRustは100%完全に安全です...
不安全なRustにFFIできることを除いて．

不安全なRustは安全なRustの機能をすべて持っており，それに加えていくつか*追加の*，C言語に
取り付いている恐ろしい未定義動作を引き起こすような，野蛮かつ不安全な操作が行なえます．

繰り返しになりますが，この話題はたくさんの興味深い内容を含みます．私は本当にこの話題に
深入りしたくありません（実はしたいです．というかしました．[こちらを読んでください][nom]）．
でも大丈夫です．私達のリストを実装する上では深入りする必要はありません．

主に使う不安全アイテムは*生ポインタ*です．生ポインタとは，基本的にはCのポインタです．
エイリアスルールを持たず，ライフタイムを持たず，nullポインタにもダングリングポインタにも
なり得る，未初期化のメモリ領域を指すこともでき，整数型と互換性があり，キャストして別の型を
指すようにもできます．可変性がほしい？キャストしましょう．あまりにもなんでもできるため，
あまりにもよくぶっ壊れます．

生ポインタは悪いやつであり，正直関わらないほうが幸せな人生を送れます．しかし不幸にも
私達はおぞましい連結リストを書きたいのです．つまり私達は不安全な生ポインタを使わなくては
いけません．

生ポインタには二種類あります．`*const T`と`*mut T`です．これはそれぞれCでいうところの
`const T*`と`T*`ですが，実はCでどう扱われているかはそれほど重要ではありません．
`*const T`は参照外しによって`&T`にしか変換できませんが，これは変数の可変性と同じ感じで
誤った使い方を防ぐためです．要はたいていの場合，まず`*const`を`*mut`にキャストする
必要があるということです．たとえポインタの指す値を変える権限がなくてもよくないことが
起こりうるのです．

まあなんにせよ書いていくうちに慣れてくるでしょう．とりあえず`*mut T == &unchecked mut T`
と考えてください！

[nom]: https://doc.rust-jp.rs/rust-nomicon-ja/
