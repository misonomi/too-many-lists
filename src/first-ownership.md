# 所有権

リストを作ることができるようになったので，次はリストで何か*できる*ようになると
いいですね．それを実現するために普通の（スタティックでない）メソッドを使います．
メソッドは`self`という型のない引数を持つ，関数の特別な形です：

```rust ,ignore
fn foo(self, arg2: Type2) -> ReturnType {
    // body
}
```

selfは主に3つの形をとります：`self`，`&mut self`，そして `&self`です．
これらはRustにおける所有権の3つの形に対応しています：

* `self` - 値
* `&mut self` - 可変な参照[^1]
* `&self` - 共有参照[^2]

値は*本物の*所有権を表します．この値を煮るなり焼くなり動かすなり破壊するなり変更するなり
参照を貸し出すなり自由です．
A value represents *true* ownership. You can do whatever you want with a value:
move it, destroy it, mutate it, or loan it out via a reference. When you pass
something by value, it's *moved* to the new location. The new location now
owns the value, and the old location can no longer access it. For this reason
most methods don't want `self` -- it would be pretty lame if trying to work with
a list made it go away!

A mutable reference represents temporary *exclusive access* to a value that you
don't own. You're allowed to do absolutely anything you want to a value you
have a mutable reference to as long you leave it in a valid state when you're
done (it would be rude to the owner otherwise!). This means you can actually completely
overwrite the value. A really useful special case of this is *swapping* a value
out for another, which we'll be using a lot. The only thing you can't do with an
`&mut` is move the value out with no replacement. `&mut self` is great for
methods that want to mutate `self`.

A shared reference represents temporary *shared access* to a value that you
don't own. Because you have shared access, you're generally not allowed to
mutate anything. Think of `&` as putting the value out on display in a museum.
`&` is great for methods that only want to observe `self`.

Later we'll see that the rule about mutation can be bypassed in certain cases.
This is why shared references aren't called *immutable* references. Really,
mutable references could be called *unique* references, but we've found that
relating ownership to mutability gives the right intuition 99% of the time.

[^1]: mutable reference．定訳がないような気がする

[^2]: shared reference．定訳が以下略