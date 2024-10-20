+++
title = "trait bound 咬人事件"
date = "2022-02-23 19:48:17"
authors = ["茨月"]
+++

本文讨论了 Rust 中泛型快速幂的实现，以及 higher rank trait bound 在其中的应用。

<!-- more -->

上完现代密码学之后，看了眼往年大作业，发现有一年的大作业是实现 RSA，于是打算趁着有时间先搞了。

实现 RSA 首先要实现在 $2 ^ {1023}$ 和 $2^{1024}$ 之间采样素数。这么大的地方不管是线性筛还是 $O(\sqrt{n})$ 的判定都过于耗时了，因此我们用 Miller-Rabin 法检测。实现这个则首先需要实现快速幂。

第一版快速幂很快写完了：

```rust
pub fn quick_pow(base: BigUint, mut exp: BigUint, prime: Option<BigUint>) -> BigUint {
    let mut temp_val = if let Some(prime) = &prime {
        base % prime
    } else {
        base
    };
    let mut return_val = 1_u32.into();
    let one: BigUint = 1_u32.into();
    let two: BigUint = 2_u32.into();
    while exp >= one {
        if let Some(prime) = &prime {
            temp_val %= prime;
        }
        if &exp % &two == one {
            return_val *= &temp_val;
        } 
        temp_val = &temp_val * &temp_val;
        exp /= &two;
    }
    if let Some(prime) = &prime {
        return_val % prime
    } else {
        return_val
    }
}
```

完事之后呢，有感觉应该写一个泛型的，于是把函数体留着，开始盘算要加哪些 trait bound。

算了好久，发现生命周期总是不对，然后被同学提醒需要用 [HRTB](https://nomicon.purewhite.io/hrtb.html)。这个东西的语义大概是，「对于任意一个生命周期`'a`，以下 trait bound 都能满足」。

用完 HRTB 之后，这个 trait bound 已经复杂到惨不忍睹了，大概长这样：

```rust
pub fn quick_pow<T>(base: T, mut exp: T, prime: Option<T>) -> T
    where
        for<'b> &'b T: Mul<&'b T, Output = T> + Rem<&'b T, Output = T>,
        T: From<u32> + Mul<Output = T> + for<&'b> Mul<&'b T, Output = T> + Ord + for<'b> Rem<&'bT, Output = T> + for<'b> Div<&'b T, Output = T>
{
    // body omitted
}
```

看着就要吐了。

在我吐槽了好久之后，我发现了今天的主角：`num-traits`，它考虑到了实现泛型四则运算的复杂与困难，因此预先为我们实现好了一系列组合 trait: `NumOps`, `NumRef` 和 `RefNum`:

```rust
pub trait NumOps<Rhs = Self, Output = Self>:
    Add<Rhs, Output = Output>
    + Sub<Rhs, Output = Output>
    + Mul<Rhs, Output = Output>
    + Div<Rhs, Output = Output>
    + Rem<Rhs, Output = Output>
{
}

pub trait NumRef: Num + for<'r> NumOps<&'r Self> {}

pub trait RefNum<Base>: NumOps<Base, Base> + for<'r> NumOps<&'r Base, Base> {}
```

简单地说：

- 如果要实现 `T op T = T`，用 `NumOps`

- 如果要实现 `T op &T = T`，用 `NumRef`

- 都需要可以直接 `RefNum<T>`

- 如果还需要 `&T op &T = T`，则需要额外添加 `where for<'a> &'a T: RefNum<T>`

所以说最后函数签名变成了：

```rust
pub fn quick_pow<T: RefNum<T> + From<u32> + Ord>(base: T, mut exp: T, prime: Option<T>) -> T
    where for<'a> &'a T: RefNum<T>
{
    // body omitted
}
```

赞美社区。