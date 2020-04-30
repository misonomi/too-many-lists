# 日本語版について
日本語話者の方にRustおよびこの本に触れていただくために頑張って訳しました．
勝手にforkして訳しただけであり，元repoとのそれ以上の関係はありません．

ビルドしたものは https://misonomi.github.io/too-many-lists/ から読んでください．

誤字・脱字・誤訳・超訳・そもそも意味がわかってないなどたくさんあると思われるのでPRください．

# Learn Rust by writing Entirely Too Many Linked Lists
[![Build Status](https://travis-ci.org/rust-unofficial/too-many-lists.svg?branch=master)](https://travis-ci.org/rust-unofficial/too-many-lists)

Read the pretty version at https://rust-unofficial.github.io/too-many-lists/.

# Building

Building requires mdbook, which can be installed from crates.io:

```sh
cargo install mdbook
```

Assuming you've placed the install directory `~/.cargo/bin` into your system PATH, then run from the root of your local copy:

```sh
mdbook build
```

---

If you'd prefer, this project can also be built with
[GitBook](https://github.com/GitbookIO/gitbook), although GitBook
is not officially supported and compatibility is therefore
uncertain and incidental.
