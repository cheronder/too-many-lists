# Push 方法

> 原文链接：[first-push.md](https://github.com/rust-unofficial/too-many-lists/blob/master/src/first-push.md) <br>
> 翻译基准：[commit b57202a](https://github.com/rust-unofficial/too-many-lists/blob/b57202a5e01b50e4217b85af3d89f49f612dcbae/src/first-push.md)

下面开始写把数据压入链表的代码。由于 `push` 方法**改变**了链表，所以应该使用 `&mut self` 形式的方法。我们还需要一个 `i32` 的整数作为要压入的数据。

``` rust, ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        // TODO
    }
}
```

首先，我们要创建一个存储数据的结点：

``` rust, ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: ?????
        };
    }
}
```

`next` 是什么呢？没错，就是原来的整个链表！那我们……可以这么做吗？

``` rust, ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: self.head,
        }
    };
}
```

``` text
$ cargo build
   Compiling lists v0.1.0
error[E0507]: cannot move out of `self.head` which is behind a mutable reference
  --> src/first.rs:22:19
   |
22 |             next: self.head,
   |                   ^^^^^^^^^ move occurs because `self.head` has type
`first::Link`, which does not implement the `Copy` trait
```

不！Rust 确实输出了正确的信息，但问题的确切含义和解决方法仍旧云里雾里。

> 【已翻译】
>
> error[E0507]: 不能移动 `self.head` 变量，因为它是可变引用。[^1]

我们试图将 `self.head` 字段赋给 `next`，但 Rust 并不同意。当 `self` 借用结束并「归还」给真正的所有者时，我们会发现 `self` 少了一部份。我们之前说过，这样粗鲁地使用 `&mut` 是被文雅的 Rust 所禁止的。

如果我们把移走的东西补充回来会怎样？具体来说，就用新节点来补充。

``` rust, ignore
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: self.head,
    });

    self.head = Link::More(new_node);
}
```

``` text
$ cargo build
   Compiling lists v0.1.0
error[E0507]: cannot move out of `self.head` which is behind a mutable reference
  --> src/first.rs:22:19
   |
22 |             next: self.head,
   |                   ^^^^^^^^^ move occurs because `self.head` has type
`first::Link`, which does not implement the `Copy` trait
```

又搞砸了。从原则上来讲，这下 Rust 应该能接受这种写法，但它没有。有许多原因导致 Rust 报错，最主要的是[异常安全][es]。我们需要有一种方法能获取 `head`，同时不让 Rust 发现。我们向非著名 Rust 黑客 Indiana Jones 寻求建议：

![Indy Prepares to mem::replace][indy]

Indiana Jones 建议我们使用 `mem::replace`。这个函数能让我们偷走借来的值，并用另一个值取而代之。在文件开头引入 `std::mem`, 就能使 `mem` 存在于局部作用域中了：

``` rust, ignore
use std::mem;
```

并正确地使用：

``` rust, ignore
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: mem::replace(&mut self.head, Link::Empty),
    });
    self.head = Link::More(new_node);
}
```

这里我们在将新链表头赋给 `self.head` 之前，暂时用`Link::Empty`来取代它。我没有开玩笑：这件事令人难受却又不得不做，至少现在没错。

但我们已经搞定 `push` 操作了！应该说是大概搞定了，我们还得测试一下。现在看来最简单的方式是把 `pop` 方法写出来，然后验证是否能产生正确的结果。

[^1]: 由于译者使用了较新版本的 Rust，错误提示与原文不一致。但错误代码以及含义都是一样的。这里采用较新版本的错误提示。

[es]: https://doc.rust-lang.org/nightly/nomicon/exception-safety.html
[indy]: https://s3.ax1x.com/2020/12/03/DTI7QA.gif
