+++
title = "双向链表的正确姿势是？"
date = "2022-02-20 22:58:16"
authors = ["茨月"]
+++

学完 [too many linked lists](https://rust-unofficial.github.io/too-many-lists/) 之后，我跃跃欲试地想要实现一个书中 WIP 的 `unsafe` 双向链表。

<!-- more -->

有了 `unsafe` 链表的经验，其实双向链表并不算很困难，很快就写出来了。核心的两个结构定义如下：

```rust
pub struct List<T> {
    head: *mut Node<T>,
    tail: *mut Node<T>
}

pub struct Node<T> {
    data: T,
    prev: *mut Node<T>,
    next: *mut Node<T>
}
```

实际上，如果把 `*mut Node<T>` 和 C++ 中的 `Node<T>*` 做简单对应，我们很快就会发现它和 C++ 版几乎是等价的：

```c++
template<typename T>
struct List<T> {
    Node<T> *head;
    Node<T> *tail;
};

template<typename T>
struct Node<T> {
    T data;
    Node<T> *prev;
    Node<T> *next;
}
```

后面的实现多数比较四平八稳，涉及一些 `unsafe`，但整体逻辑很清晰（反正和写 C 没啥区别）。

写完之后，感觉这个实现已经很合适了，于是开始看标准库的实现——为什么会是这样？

```rust
pub struct LinkedList<T> {
    head: Option<NonNull<Node<T>>>,
    tail: Option<NonNull<Node<T>>>,
    len: usize,
    marker: PhantomData<Box<Node<T>>>,
}

struct Node<T> {
    next: Option<NonNull<Node<T>>>,
    prev: Option<NonNull<Node<T>>>,
    element: T,
}
```

首先看 `Node`。

查阅标准库的定义，`std::ptr::NonNull<T>` 的定义是 "`*mut T` but non-zero and covariant"，即一个非空协变的裸指针。那么在 `NonNull` 外面再套一层 `Option`，就会得到一个可空协变的裸指针。

这样做的好处是显然的：`Node<T>` 的所有成员都是协变的，那么 `Node<T>` 就是协变的。有协变的单元对于容器使用是一件很方便的事情。

然后看 `LinkedList`。

我们知道，`PhantomData` 的作用是留住没有用到的类型参数 / 生命周期参数，但前面的 `head` 和 `tail` 里面都有 `T` 了，这里为什么还要放一个 `marker: PhantomData<Box<Node<T>>>` 呢？

这确实非常令人迷惑……直到我看到下面的 `Iter`:

```rust
pub struct Iter<'a, T: 'a> {
    head: Option<NonNull<Node<T>>>,
    tail: Option<NonNull<Node<T>>>,
    len: usize,
    marker: PhantomData<&'a Node<T>>,
}

impl<T> LinkedList<T> {
    // ...
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { head: self.head, tail: self.tail, len: self.len, marker: PhantomData }
    }
    // ...
}
```

作为对比，我实现的 Iter 里面只有两个 `Option<&'a Node<T>>`。

虽然链表本身不需要 `PhantomData` 来记录生命周期，但是 `Iter` 使用 `NonNull<Node<T>>` 而非 `&'a Node<T>` ，需要额外的生命周期标注，因此在这里引入了一个 `PhantomData`（而且注意，`PhantomData` 是 ZST，不会引起任何额外运行时开销）。

这个问题其实说明了一个事实，即 `Rust` 中并不存在与 `C` 指针行为完全一致的东西—— `*mut T` 不协变，`NonNull<T>` 不可空（硬放空值是 UB），在使用它们的时候还是要小心加小心，而且要根据自己的需求选择最合适的类型。