# 内存布局

> 原文链接：[first-layout.md](https://github.com/rust-unofficial/too-many-lists/blob/master/src/first-layout.md) <br>
> 翻译基准：[commit 0f4c26b](https://github.com/rust-unofficial/too-many-lists/blob/0f4c26bfe58f81d3667eb214398a3cf99f66a7f2/src/first-layout.md)

那么，什么是链表呢？说得简单一点，就是堆上的一系列数据，按顺序指向别的数据形成的结构。对使用过程式语言的程序员来说，链表是个大坑。不过函数式教徒却觉得用它可以构造一切。你要是问他们链表是什么，他们可能会给你如下的定义：

```haskell
List a = Empty | Elem a (List a)
```

意思是，一个链表 `List` 要么是空表 `Empty`，要么持有一个元素 `Elem`，同时后面跟着下一个链表 `List`。这是一种递归定义的**和类型**（*sum type*），即「可以具有不同类型的值的类型」。在 Rust 中，和类型被称作枚举类型 `enum`。如果你曾是 C-like 语言的程序员，应该会很熟悉这个名字。下面我们用 Rust 写出这个函数式版的定义：

文件名：src/first.rs

```rust, ignore
// pub 能允许别人在模块外调用 List 类型
pub enum List {
    Empty,
    Elem(i32, List),
}
```

写完了！编译一下吧。

```text
$ cargo build
   Compiling lists v0.1.0
error[E0072]: recursive type `first::List` has infinite size
 --> src/first.rs:2:1
  |
2 | pub enum List {
  | ^^^^^^^^^^^^^ recursive type has infinite size
3 |     Empty,
4 |     Elem(i32, List),
  |               ---- recursive without indirection
  |
  = help: insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make
 `first::List` representable
```

呃，我不知道你怎么想，反正我觉得我被函数式社区给坑了。

如果仔细读一下出错信息，其实就能发现 rustc 编译器已经告诉了我们解决之道。

> 提示：在某处添加间接修饰符，比如：`Box`，`Rc`，`&`，可以让类型 `first::List` 成立。

好吧，不过 `Box` 是什么东西？搜一下吧：

> [std::boxed::Box - Rust][box-doc]

摘录如下：

> 【已翻译】
>
> ## Struct std::boxed::Box
>
> ```rust, ignore
> pub struct Box<T>(_)
>  where
>     T: ?Sized;
> ```
>
> 一种用于堆分配内存的指针类型。
>
> 详细内容请参考[相关模块文档][box-doc-more]。

点击链接：

> 【已翻译】
>
> ## Module std::boxed
>
> 一种用于堆分配内存的指针类型。
>
> `Box<T>`，简称 box，提供了 Rust 中最简单的堆分配方式。在分配内存时获得所有
> 权，并在离开作用域时释放内存。
>
> ### 例子
>
> 通过装箱，把数据从栈中转移到堆中：
>
> ```rust, ignore
> let val: u8 = 5;
> let boxed: Box<u8> = Box::new(val);
> ```
>
> 通过[解引用][deref-doc]，把数据从堆中转移到栈中：
>
> ```rust, ignore
> let boxed: Box<u8> = Box::new(5);
> let val: u8 = *boxed;
> ```
>
> 构造递归定义的数据结构：
>
> ```rust
> #[derive(Debug)]
> enum List<T> {
>     Cons(T, Box<List<T>>),
>     Nil,
> }
>
> fn main() {
>     let list: List<i32> =
>         List::Cons(1, Box::new(List::Cons(2, Box::new(List::Nil))));
>     println!("{:?}", list);
> }
> ```
>
> 程序会输出 `Cons(1, Cons(2, Nil))`。
>
> 递归类型必须被装箱，不然如果像下面这样定义 `Cons`：
>
> ```rust, ignore
> # enum List<T> {
>     Cons(T, List<T>),
> #     Nil,
> # }
> ```
>
> 就会出错。因为 `List` 类型的大小依赖于内部元素的大小，但我们无法得知要给 `Cons` 分配多大内存。通过引入确定大小的 `Box<T>`，解决了 `Cons` 大小不定的问题。

哇！这可能是我见过最有帮助的文档了。开篇第一个内容就恰好是我们正尝试在写的程序，包括它可能错误的原因以及修正错误的办法。

那我们就照办吧：

```rust, ignore
pub enum List {
    Empty,
    Elem(i32, Box<List>),
}
```

```text
$ cargo build
   Compiling lists v0.1.0
    Finished dev [unoptimized + debuginfo] target(s) in 0.45s
```

成功编译过了！

不过，从很多方面来看，这样定义链表实际上很蠢。考虑有两个元素的链表：

```text
[] = 栈
() = 堆

[Elem A, 指针] -> (Elem B, 指针) -> (Empty 垃圾数据)
```

有两个关键问题：

* 我们给一个没有存储数据信息的结点分配了空间。
* 其中一个结点却没有被分配空间。

乍一看，这两个问题好像相互抵消了——尽管多用了一个结点，第一个结点却没必要在堆上分配。我们再看看另一种可能的布局：

```text
[指针] -> (Elem A, 指针) -> (Elem B, 空指针)
```

这种布局中，所有结点无一例外都在堆上分配。与第一种布局相比，最关键的区别在于没有了**垃圾数据**（*junk*）。这是什么东西？要理解它，首先得知道枚举类型在内存中是怎么样被安排的。

```rust, ignore
enum Foo {
    D1(T1),
    D2(T2),
    ...
    Dn(Tn),
}
```

`Foo` 类型的变量需要用一些整数来指示它是该枚举类型中的哪一种**变体**（*variant*），诸如 `D1`、`D2`、… `Dn`。这样的整数称作枚举类型的**标志**（*tag*）。同时它需要足够的空间来存储 `T1`、`T2`、… `Tn` 中最大的类型（可能需要额外的空间来满足内存对齐的要求）。

这就造成了很大的缺陷：`Empty` 变体只含有一位的信息，却占用了一个元素加一个指针的空间，因为它随时可能转变为 `Elem` 变体。所以说在第一种布局中，多分配的那个结点包含的全是垃圾数据，会占用更多空间。

对于第二点，可能会令人惊讶：不分配第一个结点劣于总是在堆上分配结点。这是因为前者会造成**内存布局不一致**的问题。在压入、弹出操作中，影响不大，但如果要拆分、合并链表，就大不一样了。

考虑在两种内存布局中，各自执行拆分操作：

```text
布局 1

[Elem A, 指针] -> (Elem B, 指针) -> (Elem C, 指针) -> (Empty 垃圾数据)

在 C 处分裂：

[Elem A, 指针] -> (Elem B, 指针) -> (Empty 垃圾数据)
[Elem C, 指针] -> (Empty 垃圾数据)
```

```text
布局 2

[指针] -> (Elem A, 指针) -> (Elem B, 指针) -> (Elem C, 空指针)

在 C 处分裂：

[指针] -> (Elem A, 指针) -> (Elem B, 空指针)
[指针] -> (Elem C, 空指针)
```

在布局 2 的操作中，仅仅需要把 B 元素后继指针复制到栈上，然后置其为空指针。而布局 1 最终也要干同样的事，只不过是把整个 C 元素结点从堆中复制到栈上。合并操作亦然。

在链表为数不多的优点中，有一点就是：当你在结点中创建完元素后，就可以随意地移动它到表中的任何位置而无须复制内存，因为只要修改对应指针就行了。如果用了布局 1，这样的优点就不复存在了。

现在我们有充足的理由认为原来的布局很糟糕。那要怎么重写呢？也许是这样：

```rust, ignore
pub enum List {
    Empty,
    ElemThenEmpty(i32),
    ElemThenNotEmpty(i32, Box<List>),
}
```

希望你能看出这样设计更加糟糕。最明显的是它将原本简单的逻辑复杂化了，因为凭空出现了一种非法状态 `ElemThenNotEmpty(0, Box(Empty))`。同时，它**也没有**解决内存分配不一致的问题。

不过，这种布局确实有一个优点：无须为 `Empty` 变体分配空间，堆分配次数也减少了一次。但如果真的这么做，很遗憾，它会耗费**更多空间**。这是因为原来的布局利用到了**空指针优化**。

之前我们说到，每个枚举变量都会有**标志**来标明它是哪种变体。但如果是这样的枚举类型：

```rust, ignore
enum Foo {
    A,
    B(NonNullPtr),
}
```

空指针优化就能起作用了。它可以**节省标签占用的空间**。如果是变体 `A`，就将储存内容全部置零。反之，非零内容则代表变体 `B`。这种优化运用到了变体 `B` 包含非空指针的的性质。酷！

想想看，还有没有别处会用到类似的优化？其实有很多！这也是为什么 Rust 不让你指定枚举类型的内存布局。在其他一些更复杂的情形中，Rust 也能做出优化。但是空指针优化绝对是最重要的。因为有了它就意味着 `&`，`&mut`，`Box`，`Rc`，`Arc`，`Vec` 等重要类型可以被无额外开销地放入 `Option` 结构中。（在本书完结前，这几种类型基本都能讲到。）

那么，怎么才能避免垃圾数据的额外开销，同时拥有一致的内存分配**以及**美妙的空指针优化呢？我们最好把存放此处元素的空间，与为下一链表所分配的空间分开考虑。要想做到这一点，我们得先想一想 C-like 语言：结构体！

对于我们声明的类型而言，如果说枚举体能让它持有若干值中的**一种**，那么结构体就能使它同时拥有**多个**值。据此，我们可以将原来的 `List` 类型拆成两部分：`List` 与  `Node` 类型。

与之前说的一样，一个链表要么是空表，要么持有一个元素，同时后面跟着下一个链表。现在，我们通过用一个完全独立的类型来实现「持有一个元素，同时后面跟着下一个链表」，从而可以将 `Box` 放置在一处更优的位置上。

```rust, ignore
struct Node {
    elem: i32,
    next: List,
}

pub enum List {
    Empty,
    More(Box<Node>),
}
```

让我们逐个核对一下：

* 链表末尾没有额外的垃圾数据：通过！
* `enum` 类型符合空指针优化的要求：通过！
* 所有元素都有一致的内存布局：通过！

不错！实际上，我们只不过是构造出了一种新的布局来证明原先的布局（Rust 官方文档所建议的那个）是有问题的。

```text
$ cargo build
   Compiling lists v0.1.0
error[E0446]: private type `first::Node` in public interface
 --> src/first.rs:8:10
  |
1 | struct Node {
  | - `first::Node` declared as private
...
8 |     More(Box<Node>)
  |          ^^^^^^^^^ can't leak private type
```

🙁

Rust 又对我们发怒了。为了让别人能使用 `List`，我们将其访问权限设置为公有 `pub`，但 `Node` 不是这样。问题在于公有枚举体的所有内部变体也都是公有的，而私有类型不能出现在公有的地方。理论上我们可以也把 `Node` 设为公有类型，但这不符合 Rust 编程风格，我们应该将实现细节隐藏起来。为达成这一点，我们可以将 `List` 设成结构体：

```rust, ignore
pub struct List {
    head: Link,
}

enum Link {
    Empty,
    More(Box<Node>),
}

struct Node {
    elem: i32,
    next: Link,
}
```

由于结构体 `List` 只有一个字段，那么它的大小即是字段的大小。为零抽象开销点赞！

```text
$ cargo build
   Compiling lists v0.1.0
warning: field is never used: `head`
 --> src/first.rs:2:5
  |
2 |     head: Link,
  |     ^^^^^^^^^^
  |
  = note: `#[warn(dead_code)]` on by default

warning: variant is never constructed: `Empty`
 --> src/first.rs:6:5
  |
6 |     Empty,
  |     ^^^^^

warning: variant is never constructed: `More`
 --> src/first.rs:7:5
  |
7 |     More(Box<Node>),
  |     ^^^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/first.rs:11:5
   |
11 |     elem: i32,
   |     ^^^^^^^^^

warning: field is never used: `next`
  --> src/first.rs:12:5
   |
12 |     next: Link,
   |
```

终于编译过了！不过 Rust 依旧很生气，它认为我们所写的代码都是无用的。这是因为 `head` 从来没有被用到过。同时，作为私有字段，它也不能从外部被访问。同理，与`Link` 以及 `Node` 相关代码也是毫无用处的。要解决这一问题，就得为我们的 `List` 添加实现具体功能的代码。

[box-doc]: https://doc.rust-lang.org/std/boxed/struct.Box.html
[box-doc-more]: https://doc.rust-lang.org/std/boxed/
[deref-doc]: https://doc.rust-lang.org/std/ops/trait.Deref.html
