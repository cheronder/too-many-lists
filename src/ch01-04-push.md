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

首先，我们要构建一个存储数据的结点：

``` rust, ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: ?????
        }
    }
}
```

`next` 是什么呢？没错，就是原来的整个链表！那我们……可以就这样做吗？

``` rust, ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: self.head,
        }
    }
}
```
