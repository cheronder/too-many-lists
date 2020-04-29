# 差的栈实现

> 原文链接：[first.md](https://github.com/rust-unofficial/too-many-lists/blob/master/src/first.md) <br>
> 翻译基准：[commit b57202a](https://github.com/rust-unofficial/too-many-lists/blob/b57202a5e01b50e4217b85af3d89f49f612dcbae/src/first.md)

这一章可能在本书中是最长的，因为要讲 Rust 的基础知识，甚至会故意绕弯路以便能更好理解这门语言。

第一种链表实现方式会在 *src/first.rs* 中编写。为了让我们要写的库能调用 *first.rs*，需要在 Cargo 自动生成的 *src/lib.rs* 开头写上一行：

文件名：src/lib.rs

```rust, ignore
pub mod first;
```
