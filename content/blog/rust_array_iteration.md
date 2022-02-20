---
title: 到底怎么迭代一个Rust Array?
date: 2021-09-10 16:09:02
tags: Legacy
---

*This is one of the legacy posts*

----

# 到底怎么迭代一个Array？

前两天晚上摸鱼的时候看到同学发来的微信消息，当时没过脑子，只想了一句“数组也可以`into_iter`吧”就完了。

![](https://files.catbox.moe/vt9w1m.jpg)

摸完之后自己试了一下，发现这样还是不可以：

```rust
let a = [1, 2, 3, 4];
let s = a.into_iter().reduce(|sum, i| sum + i);
println!("{}", s);
```

编译器要求`reduce`中的`lambda`返回类型是`&i32`而不是`i32`。这就奇了怪了：`into_iter`不是获得所有权的迭代吗？为什么要求引用呢？

于是去查文档，在[这里](https://doc.rust-lang.org/std/primitive.array.html#editions)查到了这个：

> Prior to Rust 1.53, arrays did not implement `IntoIterator` by value, so the method call `array.into_iter()` auto-referenced into a slice iterator. Right now, the old behavior is preserved in the 2015 and 2018 editions of Rust for compatibility, ignoring `IntoIterator` by value. In the future, the behavior on the 2015 and 2018 edition might be made consistent to the behavior of later editions.
>
> 
>
> Starting in the 2021 edition, `array.into_iter()` will use `IntoIterator` normally to iterate by value, and `iter()` should be used to iterate by reference like previous editions.

换言之，在Array没有实现`IntoIterator`的情况下，对Array做`.into_iter()`会被自动变成切片的迭代器。在Rust 1.53版本及之后且Rust Edition为2021（暂时，以后可能对2015和2018 Edition统一行为）时，才会采用新行为，默认对值迭代。

来到[playground](https://play.rust-lang.org)尝试一波：

```rust
fn main() {
    let a = [1, 2, 3, 4];
    println!("{}", a.into_iter().reduce(|sum, i| sum + i).unwrap());
}
```

以上代码在`Stable version 1.55.0`+`2018 Edition`下编译不通过，提示如下：

```
   Compiling playground v0.0.1 (/playground)
error[E0308]: mismatched types
 --> src/main.rs:3:50
  |
3 |     println!("{}", a.into_iter().reduce(|sum, i| sum + i).unwrap());
  |                                                  ^^^^^^^
  |                                                  |
  |                                                  expected `&{integer}`, found integer
  |                                                  help: consider borrowing here: `&(sum + i)`

For more information about this error, try `rustc --explain E0308`.
error: could not compile `playground` due to previous error
```

能看到，这里仍然要求返回值是`&{integer}`

将工具链改为`Nightly version 1.56.0-nightly`（我尝试的版本是`50171c`）+`2021 Edition`下则编译通过，打印`10`。

**那么这里应该怎么写呢？**

rust提供了一个类似`reduce`的函数`fold`，都是对容器元素从前到后逐个操作，但不同的是`fold`可以指定初值（和类型），而`reduce`没有初值。

这就意味着`fold`中的lambda要求是`Fn(T, &i32) -> T`，其中`T`是初值的类型，只要填入一个0就可以完美完成；而`reduce`中的lambda必须是`Fn(&i32, &i32) -> &i32`（显然，中间结果的地址没什么意义，所以没办法这么写）。

所以最后的解决方案远比切换到最新的版本和最新的nightly build简单得多：

```rust
fn main() {
    let a = [1, 2, 3, 4];
    println!("{}", a.into_iter().fold(0, |sum, ptr| sum + *ptr).unwrap());
}
```

