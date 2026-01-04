# Pinning (固定)

Pinning 是一个出了名难以理解的概念，并且具有一些微妙和令人困惑的属性。本节将深入探讨这个主题（可能过于深入了）。Pinning 是 Rust 异步编程实现的关键[^design]，但你完全可以在不遇到 pinning 的情况下走得很远，当然也不需要对它有深刻的理解。

第一部分将给出 pinning 的摘要，这对于大多数异步程序员来说应该足够了。本章的其余部分是为实现者、进行高级或底层异步编程的人以及好奇的人准备的。

在摘要之后，本章将在进入 pinning 之前介绍一些关于移动语义的背景知识。我们将涵盖一般概念，然后是 `Pin` 和 `Unpin` 类型，pinning 如何实现其目标，以及关于在实践中使用 pinning 的几个主题。然后是关于 pinning 和异步编程的章节，以及一些 pinning 的替代方案和扩展（针对真正好奇的人）。在本章的最后是一些指向替代解释和参考资料的链接。

[^design]: 值得注意的是，pinning 是专门为实现异步 Rust 而设计的底层构建块。虽然它不直接绑定到异步 Rust，也可以用于其他目的，但它并不是作为通用机制设计的，特别是它不是自引用字段的开箱即用解决方案。将 pinning 用于异步代码以外的任何用途通常只有在被厚厚的抽象层包裹时才有效，因为它需要大量繁琐且难以推理的 unsafe 代码。

## TL;DR (太长不看)

`Pin` 标记一个指针指向一个在被丢弃之前不会移动的对象。Pinning 不是语言或编译器内置的；它通过简单地限制对指向对象的可变引用的访问来工作。在 unsafe 代码中打破 pinning 很容易，但像 unsafe 代码中的所有安全保证一样，程序员有责任不这样做。

通过保证对象不会移动，pinning 使得从结构体的一个字段引用另一个字段（有时称为自引用）变得安全。这是实现异步函数所必需的（异步函数被实现为数据结构，其中变量存储为字段，由于变量可能相互引用，实现异步函数的 future 的字段必须能够相互引用）。大多数情况下，程序员不需要知道这个细节，但是当直接处理 futures 时，你可能需要知道，因为 `Future::poll` 的签名要求 `self` 是被 pinned 的。

如果你通过引用使用 futures，你可能需要使用 `pin!(...)` 来 pin 一个引用，以确保该引用仍然实现 `Future` trait（这通常在使用 `select` 宏时出现）。同样，如果你想手动调用 future 上的 `poll`（通常是因为你正在实现另一个 future），你需要一个指向它的 pinned 引用（使用 `pin!` 或确保参数具有 pinned 类型）。如果你正在实现一个 future 或者由于其他原因有一个 pinned 引用，并且你想对对象的内部进行可变访问，你需要理解下面关于 pinned 字段的部分，以知道如何做以及何时是安全的。

## 移动语义 (Move semantics)

讨论 pinning 和相关主题的一个有用概念是 *place* (位置) 的想法。Place 是内存中的一块（具有地址），值可以存在于其中。引用实际上并不指向值，它指向一个 place。这就是为什么 `*ref = ...` 是有意义的：解引用给你的是 place，而不是值的副本。Places 对于语言实现者来说是众所周知的，但在编程语言中通常是隐式的（在 Rust 中也是隐式的）。程序员通常对 places 有很好的直觉，但可能不会显式地思考它们。

除了引用，变量和字段访问也求值为 places。事实上，任何可以出现在赋值左侧的东西在运行时都必须是一个 place（这就是为什么 places 在编译器术语中被称为 'lvalue' (左值)）。

在 Rust 中，可变性是 places 的属性，因借用而被“冻结”也是如此（我们可以说该 place 被借用了）。

Rust 中的赋值会 *移动* 数据（大多数情况下，一些简单数据具有复制语义，但这并不太重要）。当我们写 `let b = a;` 时，位于 `a` 标识的 place 的内存中的数据被移动到 `b` 标识的 place。这意味着赋值后，数据存在于 `b` 但不再存在于 `a`。换句话说，对象的地址因赋值而改变[^compiler]。

如果存在指向被移出的 place 的指针，这些指针将无效，因为它们不再指向该对象。这就是为什么借用引用会阻止移动：`let r = &a; let b = a;` 是非法的，`r` 的存在阻止了 `a` 被移动。

编译器只知道从对象外部指向对象内部的引用（如上面的例子，或对对象字段的引用）。完全在对象内部的引用对编译器是不可见的。想象一下如果我们被允许写类似这样的代码：

```rust,norun
struct Bad {
    field: u64,
    r: &'self u64,
}
```

我们可以有一个 `Bad` 的实例 `b`，其中 `b.r` 指向 `b.field`。在 `let a = b;` 中，指向 `b.field` 的内部引用 `b.r` 对编译器是不可见的，所以看起来没有对 `b` 的引用，因此移动到 `a` 是可以的。然而如果那样发生了，那么移动后，`a.r` 将不会像我们希望的那样指向 `a.field`，而是指向 `b.field` 的旧位置的无效内存，违反了 Rust 的安全保证。

移动数据不仅限于值。数据也可以从唯一引用中移出。解引用 `Box` 会将数据从堆移动到栈。`take`、`replace` 和 `swap`（都在 [`std::mem`](https://doc.rust-lang.org/std/mem/index.html) 中）从可变引用 (`&mut T`) 中移出数据。从 `Box` 中移出会使指向的 place 无效。从可变引用中移出会使 place 保持有效，但包含不同的数据。

[^compiler]: 我们在这里稍微混淆了源代码和运行时。为了绝对清楚，变量在运行时不存在。（编译后的）代码片段可能会被执行多次（例如，如果它在循环中或在被多次调用的函数中）。对于每次执行，源代码中的变量在运行时将由不同的地址表示。

抽象地说，移动是通过将位从源复制到目标然后擦除源位来实现的。然而，编译器可以通过多种方式优化这一点。

## Pinning

重要提示：我将首先讨论 pinning 的抽象概念，这并不完全是任何特定类型所表达的。我们将随着进展使概念更加具体，并最终得出不同类型含义的精确定义，但这些类型都不完全等同于我们将开始讨论的 pinning 概念。

如果一个对象不会被移动或以其他方式失效，那么它就是 pinned (被固定) 的。正如我上面解释的，这不是一个新概念 - 借用一个对象会在借用期间阻止对象被移动。对象是否可以被移动在 Rust 的类型中不是显式的，尽管编译器知道这一点（这就是为什么你会得到“cannot move out of”错误消息）。与借用（以及借用引起的对移动的临时限制）相对，被 pinned 是永久的。一个对象可以从不被 pinned 变为被 pinned，但一旦它被 pinned，它必须保持 pinned 直到它被丢弃[^inherent]。

正如指针类型反映了指向对象的所有权和可变性（例如，`Box` vs `&`，`&mut` vs `&`），我们也希望在指针类型中反映 pinned 状态。这不是指针的属性 - 指针不是被 pinned 或可移动的 - 它是指向 place 的属性：指向对象是否可以移出其 place。

粗略地说，`Pin<Box<T>>` 是指向拥有的、被 pinned 对象的指针，`Pin<&mut T>` 是指向唯一借用的、可变的、被 pinned 对象的指针（对比 `&mut T`，它是指向唯一借用的、可变的对象的指针，该对象可能被 pinned 也可能没有）。

Pinning 概念直到 1.0 之后才添加到 Rust 中，出于向后兼容性的原因，没有办法显式表达一个 *对象* 是否被 pinned。我们只能表达一个引用指向一个被 pinned 或未被 pinned 的对象。

Pinning 与可变性是正交的。一个对象可能是可变的且被 pinned (`Pin<&mut T>`) 或未被 pinned (`&mut T`)（即，对象可以被修改，并且要么它被固定在原地，要么可以被移动），或者是不可变的且被 pinned (`Pin<&T>`) 或未被 pinned (`T`)（即，对象不能被修改，并且要么它不能被移动，要么可以被移动但不能被修改）。注意 `&T` 不能被修改或移动，但不是被 pinned 的，因为它的不可移动性只是暂时的。

[^inherent]: 永久性不是 pinning 的基本方面，它是 Rust 中 pinning 框架及其周围安全保证的一部分。如果可以安全地表达并且 pinning 的时间范围可以被 pinning 保证的消费者所依赖，那么 pinning 是暂时的也是可以的。然而，这在今天的 Rust 或任何合理的扩展中是不可能的。

### `Unpin`

虽然移动和不移动是我们引入 pinning 的方式，并且名字也有些暗示，但 `Pin` 实际上并没有告诉你太多关于指向对象是否真的会移动的信息。

什么？唉。

Pinning 实际上是关于有效性的契约，而不是关于移动的。它保证 *如果一个对象是地址敏感的，那么* 它的地址将不会改变（因此从它派生的地址，如其字段的地址，也不会改变）。Rust 中的大多数数据不是地址敏感的。它可以被移动，一切都会好好的。`Pin` 保证指向对象相对于其地址将是有效的。如果指向对象是地址敏感的，那么它不能被移动；如果它不是地址敏感的，那么它是否被移动并不重要。

`Unpin` 是一个 trait，它表达对象是否是地址敏感的。如果一个对象实现了 `Unpin`，那么它 *不是* 地址敏感的。如果一个对象是 `!Unpin`，那么它是地址敏感的。或者，如果我们把 pinning 看作是将对象保持在其位置的行为，那么 `Unpin` 意味着撤销该行为并允许对象被移动是安全的。

`Unpin` 是一个 auto-trait，大多数类型都是 `Unpin`。只有具有 `!Unpin` 字段或显式选择退出的类型才不是 `Unpin`。你可以通过拥有一个 [`PhantomPinned`](https://doc.rust-lang.org/std/marker/struct.PhantomPinned.html) 字段或（如果你使用 nightly）使用 `impl !Unpin for ... {}` 来选择退出。

对于实现 `Unpin` 的类型，`Pin` 基本上什么都不做。`Pin<Box<T>>` 和 `Pin<&mut T>` 可以像 `Box<T>` 和 `&mut T` 一样使用。事实上，对于 `Unpin` 类型，pinned 指针和常规指针可以使用 `Pin::new` 和 `Pin::into_inner` 自由转换。值得重申：`Pin<...>` 不保证指向对象不会移动，只保证如果它是 `!Unpin`，它就不会移动。

上述内容的实际含义是，使用 `Unpin` 类型和 pinning 比使用非 `Unpin` 类型要容易得多，事实上 `Pin` 标记对 `Unpin` 类型和指向 `Unpin` 类型的指针基本上没有影响，你基本上可以忽略所有的 pinning 保证和要求。

`Unpin` 不应被理解为对象本身的属性；`Unpin` 改变的唯一事情是对象如何与 `Pin` 交互。在 pinning 上下文之外使用 `Unpin` 边界不会影响编译器的行为或可以对对象做什么。使用 `Unpin` 的唯一原因是与 pinning 结合使用，或者将边界传播到使用 pinning 的地方。

### `Pin`

[`Pin`](https://doc.rust-lang.org/std/pin/struct.Pin.html) 是一个标记类型，它对类型检查很重要，但在编译时会被消除，在运行时不存在（`Pin<Ptr>` 保证具有与 `Ptr` 相同的内存布局和 ABI）。它是指针（如 `Box`）的包装器，所以它的行为像指针类型，但它不增加间接层，`Box<Foo>` 和 `Pin<Box<Foo>>` 在程序运行时是一样的。最好把 `Pin` 看作是指针的修饰符，而不是指针本身。

`Pin<Ptr>` 意味着 `Ptr` 的指向对象（不是 `Ptr` 本身）是被 pinned 的。也就是说，`Pin` 保证指向对象（不是指针）相对于其地址将保持有效，直到指向对象被丢弃。如果指向对象是地址敏感的（即，是 `!Unpin`），那么指向对象将不会被移动。

### Pinning 值

对象不是生来就被 pinned 的。对象开始时是未 pinned 的（并且可以自由移动），当创建一个指向该对象的 pinning 指针时，它就变成了 pinned。如果对象是 `Unpin`，那么使用 `Pin::new` 这很简单，然而，如果对象不是 `Unpin`，那么 pinning 它必须确保它不能通过别名被移动或失效。

要在堆上 pin 一个对象，你可以使用 [`Box::pin`](https://doc.rust-lang.org/std/boxed/struct.Box.html#method.pin) 创建一个新的 pinning `Box`，或者使用 [`Box::into_pin`](https://doc.rust-lang.org/std/boxed/struct.Box.html#method.into_pin) 将现有的 `Box` 转换为 pinning `Box`。在任何一种情况下，你最终都会得到 `Pin<Box<T>>`。其他一些指针（如 `Arc` 和 `Rc`）也有类似的机制。对于没有这些机制的指针，或者对于你自己的指针类型，你需要使用 [`Pin::new_unchecked`](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.new_unchecked) 来创建 pinned 指针[^box-pin]。这是一个 unsafe 函数，因此程序员必须确保维护 `Pin` 的不变量。也就是说，指向对象在任何情况下都将保持有效，直到它的析构函数被调用。确保这一点有一些微妙的细节，请参阅该函数的 [文档](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.new_unchecked) 或下面的 [pinning 如何工作](#how-pinning-works) 部分以了解更多信息。

`Box::pin` 将对象 pin 到堆中的一个位置。要在栈上 pin 一个对象，你可以使用 [`pin`](https://doc.rust-lang.org/std/pin/macro.pin.html) 宏来创建并 pin 一个可变引用 (`Pin<&mut T>`)[^not-stack]。

Tokio 也有一个 [`pin`](https://docs.rs/tokio/latest/tokio/macro.pin.html) 宏，它做同样的事情，并且还支持在宏内部赋值给变量。futures-rs 和 pin-utils crates 有一个 `pin_mut` 宏，以前很常用，但现在已被弃用，转而支持 std 宏。

你也可以使用 `Pin::static_ref` 和 `Pin::static_mut` 来 pin 一个静态引用。

[^box-pin]: `Box`（或其他 std 指针）在 pinning 实现或编译器中都没有特殊待遇。`Box` 使用 `Pin` API 中的 unsafe 函数来实现 `Box::pin`。由于 `Box` 的安全保证，`Pin` 的安全要求得到了满足。

[^not-stack]: 这仅在非异步函数中严格地 pin 到栈上。在异步函数中，所有局部变量都分配在异步伪栈中，因此被 pinned 的 place 很可能作为底层异步函数的 future 的一部分存储在堆上。

### 使用 pinned 类型

理论上，使用 pinned 指针就像使用任何其他指针类型一样。然而，因为它不是最直观的抽象，而且因为它没有语言支持，使用 pinned 指针往往非常不符合人体工程学。使用 pinning 最常见的情况是处理 futures 和 streams 时，我们将在下面更详细地介绍这些细节。

将 pinned 指针作为不可变借用引用使用是很简单的，因为 `Pin` 实现了 `Deref`。你基本上可以将 `Poll<Ptr<T>>` 视为 `&T`，如果需要可以使用显式的 `deref()`。同样，使用 `as_ref()` 获取 `Pin<&T>` 也很容易。

使用 pinned 类型最常见的方式是使用 `Pin<&mut T>`（例如，在 [`Future::poll`](https://doc.rust-lang.org/std/future/trait.Future.html#tymethod.poll) 中），然而，生成 pinned 对象最简单的方法是 `Box::pin`，它给出 `Pin<Box<T>>`。你可以使用 [`Pin::as_mut`](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.as_mut) 将后者转换为前者。然而，由于没有重用引用（隐式 reborrowing）的语言支持，你必须不断调用 `as_mut` 而不是重用结果。例如（来自 `as_mut` 文档），

```rust,norun
impl Type {
    fn method(self: Pin<&mut Self>) {
        // 做一些事情
    }

    fn call_method_twice(mut self: Pin<&mut Self>) {
        // `method` 消耗 `self`，所以通过 `as_mut` reborrow `Pin<&mut Self>`。
        self.as_mut().method();
        self.as_mut().method();
    }
}
```

如果你需要以其他方式访问 pinned 指向对象，你可以通过 [`Pin::into_inner_unchecked`](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.into_inner_unchecked) 来做。然而，这是 unsafe 的，你必须 *非常* 小心地确保遵守 `Pin` 的安全要求。

### Pinning 如何工作

`Pin` 是指针的一个简单包装结构体（又名 newtype）。它通过要求其泛型参数上的 `Deref` 边界来强制仅对指针起作用，但这只是为了表达意图，而不是为了保持安全性。与大多数 newtype 包装器一样，`Pin` 的存在是为了在编译时表达不变量，而不是为了任何运行时效果。实际上，在大多数情况下，`Pin` 和 pinning 机制在编译期间会完全消失。

准确地说，`Pin` 表达的不变量是关于有效性的，而不仅仅是可移动性。这也是一个仅在指针被 pinned 后才适用的有效性不变量 - 在那之前 `Pin` 没有效果，也不对被 pinned 之前发生的事情提出要求。一旦指针被 pinned，`Pin` 要求（并在安全代码中保证）指向对象将保持在内存中的同一地址有效，直到对象的析构函数被调用。

对于不可变指针（例如，借用引用），`Pin` 没有效果 - 因为指向对象不能被修改或替换，所以没有失效的危险。

对于允许修改的指针（例如，`Box` 或 `&mut`），拥有对该指针的直接访问权限或对指向对象的可变引用 (`&mut`) 的访问权限可能允许修改或移动指向对象。`Pin` 根本不提供任何（非 `unsafe`）方式来获取对指针的直接访问或对指向对象的可变引用。指针提供对其指向对象的可变引用的通常方式是实现 [`DerefMut`](https://doc.rust-lang.org/std/ops/trait.DerefMut.html)，`Pin` 仅在指向对象是 `Unpin` 时才实现 `DerefMut`。

这个实现非常简单！总结一下：`Pin` 是指针周围的一个包装结构体，它仅提供对指向对象的不可变访问（如果指向对象是 `Unpin`，则提供可变访问）。其他一切都是细节（以及 unsafe 代码的微妙不变量）。为了方便起见，`Pin` 提供了在 `Pin` 类型之间转换的功能（总是安全的，因为指针不能逃脱 `Pin`）等。

`Pin` 还提供了用于创建 pinned 指针和访问底层数据的 unsafe 函数。与所有 `unsafe` 函数一样，维护安全不变量是程序员的责任，而不是编译器的责任。不幸的是，pinning 的安全不变量有些分散，因为它们在不同的地方被强制执行，很难以全局、统一的方式描述。我不会在这里详细描述它们，而是建议你参考文档，但我会尝试总结（有关详细概述，请参阅 [模块文档](https://doc.rust-lang.org/std/pin/index.html)）：

- 创建一个新的 pinned 指针 [`new_unchecked`](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.new_unchecked)。程序员必须确保指向对象是 pinned 的（即，遵守 pinning 不变量）。这个要求可能仅由指针类型满足（例如，在 `Box` 的情况下），或者可能需要指向对象类型的参与（例如，在 `&mut` 的情况下）。这包括（但不限于）：
  - 在 `Deref` 和 `DerefMut` 中不移出 `self`。
  - 正确实现 `Drop`，见 [drop 保证](https://doc.rust-lang.org/std/pin/index.html#subtle-details-and-the-drop-guarantee)。
  - 如果你需要 pinning 保证，选择退出 `Unpin`（通过使用 [`PhantomPinned`](https://doc.rust-lang.org/std/marker/struct.PhantomPinned.html)）。
  - 指向对象不能是 `#[repr(packed)]`。
- 访问 pinned 值 [`into_inner_unchecked`](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.into_inner_unchecked)，[`get_unchecked_mut`](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.get_unchecked_mut)，[`map_unchecked`](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.map_unchecked)，和 [`map_unchecked_mut`](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.map_unchecked_mut)。程序员有责任从访问数据的那一刻起直到其析构函数运行为止，强制执行 pinning 保证（包括不移动数据）（注意，这个责任范围超出了 unsafe 调用，并适用于底层数据发生的任何事情）。
- 不提供任何其他方式将数据从 pinned 类型中移出（这将需要 unsafe 实现）。

#### Pinning 指针类型

我们之前说过 `Pin` 包装了一个指针类型。常见的是 `Pin<Box<T>>`，`Pin<&T>` 和 `Pin<&mut T>`。从技术上讲，pinning 指针类型的唯一要求是它实现 `Deref`。然而，除了使用 unsafe 代码（通过 `new_unchecked`）之外，没有办法为任何其他指针类型创建 `Pin<Ptr>`。这样做对指针类型有要求，以确保 pinning 契约：

- 指针的 `Deref` 和 `DerefMut` 实现必须不移出它们的指向对象。
- 在创建 `Pin` 之后的任何时候，甚至在 `Pin` 被丢弃之后，都必须不可能获得指向对象的 `&mut` 引用（这就是为什么你不能安全地从 `&mut T` 构造 `Pin<&mut T>`）。这必须通过多个步骤或通过引用保持为真（这阻止了使用 `Rc` 或 `Arc`）。
- 指针的 `Drop` 实现必须不移动（或以其他方式使失效）其指向对象。

有关更多详细信息，请参阅 `new_unchecked` [文档](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.new_unchecked)。

### Pinning 和 `Drop`

Pinning 契约适用直到 pinned 对象被丢弃（技术上，这意味着当它的 `drop` 方法返回时，而不是当它被调用时）。这通常相当直接，因为当对象被销毁时会自动调用 `drop`。如果你手动处理对象的生命周期，你可能需要多加考虑。如果你有一个是（或可能是）pinned 的对象，并且该对象不是 `Unpin`，那么在释放或重用对象的内存或地址之前，你必须调用它的 `drop` 方法（使用 [`drop_in_place`](https://doc.rust-lang.org/std/ptr/fn.drop_in_place.html)）。有关详细信息，请参阅 [std 文档](https://doc.rust-lang.org/std/pin/index.html#drop-guarantee)。

如果你正在实现一个地址敏感类型（即，一个是 `!Unpin` 的类型），那么你必须格外小心 `Drop` 实现。即使 `drop` 中的 self 类型是 `&mut Self`，你也必须将 self 类型视为 `Pin<&mut Self>`。换句话说，你必须确保对象在 `drop` 函数返回之前保持有效。在源代码中明确这一点的一种方法是遵循以下惯用语：

```rust,norun
impl Drop for Type {
    fn drop(&mut self) {
        // `new_unchecked` 是可以的，因为我们知道这个值在被丢弃后
        // 永远不会再被使用。
        inner_drop(unsafe { Pin::new_unchecked(self)});

        fn inner_drop(this: Pin<&mut Self>) {
            // 实际的 drop 代码在这里。
        }
    }
}
```

注意，有效性要求将取决于正在实现的类型。精确定义这些要求，特别是关于对象销毁的要求是推荐的，特别是如果涉及多个对象（例如，侵入式链表）。确保这里的正确性可能会很有趣！

### 方法中的 Pinned self

在 pinned 类型上调用方法会导致思考这些方法中的 self 类型。如果方法不需要修改 `self`，那么你仍然可以使用 `&self`，因为 `Pin<...>` 可以解引用为借用引用。然而，如果你需要修改 `self`（并且你的类型不是 `Unpin`），那么你需要在 `&mut self` 和 `self: Pin<&mut Self>` 之间做出选择（虽然 pinned 指针不能隐式强制转换为后者类型，但可以使用 `Pin::as_mut` 轻松转换）。

使用 `&mut self` 使实现变得容易，但意味着该方法不能在 pinned 对象上调用。使用 `self: Pin<&mut Self>` 意味着考虑 pin projection (pin 投影)（见下一节），并且只能在 pinned 对象上调用。虽然这有点令人困惑，但当你记住 pinning 是一个分阶段的概念时，这就直观了 - 对象开始时未 pinned，在某个时刻经历阶段变化变为 pinned。`&mut self` 方法是可以在第一阶段（未 pinned）调用的方法，`self: Pin<&mut Self>` 方法是可以在第二阶段（pinned）调用的方法。

注意 `drop` 接受 `&mut self`（即使它可能在任一阶段被调用）。这是由于语言的限制和对向后兼容性的渴望。它需要编译器中的特殊处理，并带有安全要求。

### Pinned 字段、结构化 pinning 和 pin projection

鉴于一个对象是 pinned 的，这告诉我们关于其字段的 'pinned' 状态什么信息？答案取决于数据类型实现者的选择，没有通用的答案（实际上对于同一对象的不同字段可能不同）。

如果对象的 pinned 状态传播到字段，我们说该字段表现出 'structural pinning' (结构化 pinning) 或者 pinning 随字段投影。在这种情况下，应该有一个投影方法 `fn get_field(self: Pin<&mut Self>) -> Pin<&mut Field>`。如果字段不是结构化 pinned 的，那么投影方法应该具有签名 `fn get_field(self: Pin<&mut Self>) -> &mut Field`。实现任一方法（或实现类似代码）都需要 `unsafe` 代码，并且任一选择都有安全含义。Pin 传播必须是一致的，字段必须始终是结构化 pinned 的或不是，字段有时是结构化 pinned 的而有时不是几乎总是不安全的。

如果字段是聚合数据类型的地址敏感部分，则 Pinning 应该投影到该字段。也就是说，如果被 pinned 的聚合体依赖于被 pinned 的字段，那么 pinning 必须投影到该字段。例如，如果有从聚合体的另一部分到该字段的引用，或者如果字段内有自引用，那么 pinning 必须投影到该字段。另一方面，对于泛型集合，pinning 不需要投影到其内容，因为集合不依赖于它们的行为（这是因为集合不能依赖于它包含的泛型项的实现，所以集合本身不能依赖于其项的地址）。

编写 unsafe 代码时，你只能假设 pinning 保证适用于结构化 pinned 的对象字段。另一方面，你可以安全地将非结构化 pinned 字段视为可移动的，而不必担心它们的 pinning 要求。特别是，即使字段不是 `Unpin`，结构体也可以是 `Unpin`，只要该字段始终被视为非结构化 pinned。

如果字段是结构化 pinned 的，那么聚合结构体上的 pinning 要求扩展到该字段。在任何情况下，代码都不能在聚合体被 pinned 时移动字段的内容（这总是需要 unsafe 代码）。结构化 pinned 字段必须在它们被移动（包括释放）之前被丢弃，即使在 panic 的情况下也是如此，这意味着必须在聚合体的 `Drop` impl 中小心。此外，除非其所有结构化 pinned 字段都是 `Unpin`，否则聚合结构体不能是 `Unpin`。

#### Pin projection 的宏

有一些宏可用于帮助 pin projection。

[pin-project](https://docs.rs/pin-project/latest/pin_project/) crate 提供了 `#[pin_project]` 属性宏（和 `#[pin]` 辅助属性），它通过创建注释类型的 pinned 版本来实现安全的 pin projection，该版本可以使用注释类型上的 `project` 方法访问。

[Pin-project-lite](https://docs.rs/pin-project-lite/latest/pin_project_lite/) 是一个使用声明性宏 (`pin_project!`) 的替代方案，其工作方式与 pin-project 非常相似。Pin-project-lite 是轻量级的，因为它不是过程宏，因此不会向你的项目添加实现过程宏的依赖项。然而，它的表达能力不如 pin-project，并且不提供自定义错误消息。如果你想避免添加过程宏依赖项，建议使用 Pin-project-lite，否则建议使用 pin-project。

Pin-utils 提供了 [`unsafe_pinned`](https://docs.rs/pin-utils/latest/pin_utils/macro.unsafe_pinned.html) 宏来帮助实现 pin projection，但整个 crate 已被弃用，转而支持上述 crates 和现在 std 中的功能。

### 赋值给 pinned 指针

通常 [赋值给 pinned 指针](https://doc.rust-lang.org/std/pin/index.html#assigning-pinned-data) 是安全的。虽然不能以通常的方式完成 (`*p = ...`)，但可以使用 [`Pin::set`](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.set) 完成。更一般地，你可以使用 unsafe 代码赋值给指向对象的字段。

使用 `Pin::set` 总是安全的，因为先前 pinned 的指向对象将被丢弃，满足 pin 要求，并且新的指向对象在移动到 pinned place 完成之前不是 pinned 的。赋值给单个字段不会自动违反 pinning 要求，但必须小心确保对象作为一个整体保持有效。例如，如果赋值给一个字段，那么引用该字段的任何其他字段必须仍然对新对象有效（这不是 pinning 要求的一部分，但可能是对象其他不变量的一部分）。

将一个 pinned 对象复制到另一个 pinned place 只能在 unsafe 代码中完成，如何维护安全性取决于单个对象。没有普遍违反 pinning 要求 - 被替换的对象没有移动，被复制的对象也没有移动。然而，被替换对象的有效性可能有通常受 pinning 保护的安全要求，但在这种情况下必须由程序员建立。例如，如果我们有一个带有两个字段 `a` 和 `b` 的结构体，其中 `b` 引用 `a`，该引用需要 pinning 才能保持有效。如果这样的结构体被复制到另一个 place，那么 `b` 的值必须更新为指向新的 `a` 而不是旧的 `a`。

## Pinning 和异步编程

希望你能做你想用异步 Rust 做的一切，而不用担心 pinning。有时你会遇到需要使用 pinning 的极端情况，如果你想实现 futures、运行时或类似的东西，你需要了解 pinning。在本节中，我将解释原因。

异步函数被实现为 futures（见章节 TODO - 这是一个摘要概述，确保我们在其他地方更深入地解释并举例）。在每个 await 点，函数的执行可能会暂停，在此期间，活动变量的值必须被保存。它们本质上变成了结构体的字段（这是枚举的一部分）。这些变量可能引用 future 中保存的其他变量，例如，考虑：

```rust,norun
async fn foo() {
  let a = ...;
  let b = &a;
  bar().await;
  // 使用 b
}
```

这里生成的 future 对象将类似于：

```rust,norun
struct Foo {
  a: A,
  b: &'self A,  // 不变量 `self.b == &self.a`
}
```

（我稍微简化了一下，忽略了执行状态等，但重要的是变量/字段）。

这在直觉上是有道理的，不幸的是 `'self` 在 Rust 中不存在。这是有充分理由的！记住 Rust 对象可以被移动，所以像下面的代码将是不安全的：

```rust,norun
let f1 = Foo { ... }; // f1.b == &f1.a
let f2 = f1; // f2.b == &f1.a, 但 f1 不再存在，因为它移动到了 f2
```

注意，这不仅仅是无法命名生命周期的问题，即使我们使用裸指针，这样的代码仍然是不正确的。

然而，如果我们知道一旦创建，`Foo` 的实例将永远不会移动，那么一切就正常工作了。（编译器内部有一个类似于 `'self` 的概念用于这种情况，作为程序员，我们将不得不使用裸指针和 unsafe 代码）。这种不移动的概念正是 pinning 所描述的。

我们在 `Future::poll` 的签名中看到了这个要求，其中 `self`（future）的类型是 `Pin<&mut Self>`。大多数情况下，当使用 async/await 时，编译器会处理 pinning 和 unpinning，作为程序员你不需要担心它。

### 手动 Pinning

有一些地方 pinning 会通过 async/await 的抽象泄漏出来。从根本上说，这是由于 `Future::poll` 和 `Stream::poll_next` 签名中的 `Pin`。当直接使用 futures 和 streams（而不是通过 async/await）时，我们可能需要考虑 pinning 才能使事情正常工作。需要 pinned 类型的一些常见原因是：

- 轮询 future 或 stream - 无论是在应用程序代码中还是在实现你自己的 future 时。
- 使用 boxed futures。如果你使用 boxed futures（或 streams），因此写出 future 类型而不是使用异步函数，你可能会在这些类型中看到很多 `Pin<...>`，并且需要使用 `Box::pin` 来创建 futures。
- 实现 future - 在 `poll` 内部，`self` 是 pinned 的，因此你需要使用 pin projection 和/或 unsafe 代码来获得对 `self` 字段的可变访问。
- 组合 futures 或 streams。这大多数情况下都能正常工作，但如果你需要获取对 future 的引用然后轮询它（例如，在循环外定义 future 并在循环内的 `select!` 中使用它），那么你需要 pin 对 future 的引用才能像 future 一样使用该引用。
- 使用 streams - 目前 Rust 中关于 streams 的抽象比 futures 少，所以你比使用 futures 时更有可能使用组合器方法（技术上不需要 pinning，但似乎使围绕引用或创建 futures/streams 的问题更加普遍）甚至手动 `poll`。

## 替代方案和扩展

本节是为那些对围绕 pinning 的语言设计感到好奇的人准备的。如果你只想阅读、理解和编写异步程序，你绝对不需要阅读本节。

Pinning 很难理解，可能会感觉有点笨拙，所以人们经常想知道是否有更好的替代方案或变体。我将介绍几个替代方案，并说明为什么它们要么不起作用，要么比你预期的更复杂。

然而在此之前，了解 pinning 的历史背景很重要。如果你正在设计一种全新的语言并希望支持 async/await、自引用或不可移动类型，肯定有比 Rust 的 pinning 更好的方法。然而，async/await、futures 和 pinning 是在 Rust 1.0 发布后添加的，并且是在强向后兼容性保证的背景下设计的。除了那个硬性要求之外，还有一个要求是在合理的时间框架内设计和实现此功能。一些解决方案（例如，涉及线性类型的解决方案）将需要基础研究、设计和实现，考虑到 Rust 项目的资源和限制，这实际上可能需要几十年的时间。

### 替代方案

首先，让我们考虑使 Rust 类型默认不可移动的解决方案类别。注意，这是对 Rust 基本语义的重大改变；此类中的任何解决方案都可能需要付出巨大努力才能实现向后兼容性（我不会推测特定解决方案是否可能，但通过 auto-traits、derive 属性、editions、迁移工具等技术，这可能是可能的）。

一个提议（实际上是一组提议，因为有各种定义语义的方式）是拥有一个 `Move` 标记 trait（类似于 `Copy`），它将对象标记为可移动，所有其他类型将是不可移动的。与 `Pin` 相比，这是值的属性，而不是指针的属性，因此影响更加深远，例如，如果 `b` 没有实现 `Move`，`let a = b;` 将是一个错误。

这种方法的基本问题是，今天的 pinning 是一个分阶段的概念（一个 place 开始未 pinned 并变为 pinned），而类型适用于值的整个生命周期。（Pinning 也最好理解为 places 的属性而不是值的属性，但类型适用于值，这对于任何基于 trait 的方法是否是一个基本问题，我不知道）。这两篇博客文章探讨了这一点：[Two Ways Not to Move](https://theincredibleholk.org/blog/2024/07/15/two-ways-not-to-move/) 和 [Ergonomic Self-Referential Types for Rust](https://blog.yoshuawuyts.com/self-referential-types/#immovable-types)。

此外，任何 `Move` trait 都可能存在 [向后兼容性](https://without.boats/blog/pin/) 问题，并导致 'infectious bounds' (传染性边界)（即，`Move` 或 `!Move` 将在许多许多地方被要求）。

另一个提议是支持类似于 C++ 的移动构造函数。然而，这打破了 Rust 的基本不变量，即对象总是可以按位移动。这将使 Rust 变得更加不可预测，从而使 Rust 程序更难理解和调试。这是最糟糕的向后不兼容更改，因为它会默默地破坏 unsafe 代码，因为它改变了代码作者可能做出的基本假设。此外，这种根本性改变所需的设计和实现工作将是巨大的。除了这些实际问题之外，还不清楚它是否甚至有效：移动构造函数可以用于修复被移动对象中的引用，但可能存在从对象外部指向被移动对象的引用，这些引用无法修复。

另一种不同类型的潜在解决方案是偏移引用的想法。这是一个相对引用而不是绝对引用，即，作为对另一个字段的偏移引用的字段将始终指向同一对象内，即使对象在内存中移动。偏移指针的问题是字段必须是偏移指针或绝对指针。但是异步函数中的引用变成了字段，这些字段有时引用 future 对象内部的内存，有时引用其外部的内存。

### 扩展

有多个提议使 pinning 更强大和/或更易于使用。这些大多是提议使 pinning 成为语言中更一等公民的部分，而不仅仅是一个纯粹的库概念（它们通常包括对 std 以及语言的扩展）。我将介绍几个更成熟的想法，它们相互关联，并且都有通过使创建和使用 pinned places 更容易来改善 pinning 人体工程学的总体目标，特别是在结构化 pinning 和 `drop` 方面。

[Pinned places](https://without.boats/blog/pinned-places/) 沿用了 pinning 是 places 的属性而不是值或类型的属性的想法，并向引用添加了 `pin`/`pinned` 修饰符，类似于 `mut`。这与 reborrowing 和方法解析集成，以改善使用 pinned `self` 的方法调用的人体工程学。

[`UnpinCell`](https://without.boats/blog/unpin-cell/) 扩展了 pinned places 的想法，以支持字段的原生 pin projection。[MinPin](https://smallcultfollowing.com/babysteps/blog/2024/11/05/minpin/) 是一个更最小化（且向后兼容）的提议，用于原生 pin projection 和更好的 `drop` 支持。

[`Overwrite` trait](https://smallcultfollowing.com/babysteps/series/overwrite-trait/) 是一个提议的 trait，它明确区分了修改对象一部分的权限 (`foo.f = ...`) 和覆盖整个对象的权限 (`*foo = ...`)，目前所有可变引用都允许这两种权限。该提议还包括不可变字段。`Overwrite` 是 `Unpin` 的一种替代品，它（连同 pinned places 中的一些想法）可以改善与 pinning 的协作。不幸的是，虽然它可以向后兼容地采用，但过渡将比其他扩展需要更多的工作。

## 参考资料

- [std 文档](https://doc.rust-lang.org/std/pin/index.html) `Pin` 等的行为和保证的真实来源。很好的文档。
  - [`Pin`](https://doc.rust-lang.org/std/pin/struct.Pin.html), [`Unpin`](https://doc.rust-lang.org/std/marker/trait.Unpin.html), [`pin` 宏](https://doc.rust-lang.org/std/pin/macro.pin.html)
- [RFC 2349](https://rust-lang.github.io/rfcs/2349-pin.html) 提议 pinning 的 RFC。稳定的 API 与这里提议的有点不同，但 RFC 中对核心概念和基本原理有很好的解释。
- 一些解释 pinning 的博客文章或其他资源：
  - [Pin](https://without.boats/blog/pin/) by WithoutBoats（pinning 的主要设计者）关于 pinning 的历史、背景和基本原理，以及为什么它是一个困难的概念。
  - [Why is std::pin::Pin so weird?](https://sander.saares.eu/2024/11/06/why-is-stdpinpin-so-weird/) 深入探讨 pinning 设计的基本原理以及在实践中使用 pinning。
  - [Pin, Unpin, and why Rust needs them](https://blog.cloudflare.com/pin-and-unpin-in-rust/)
  - [async/await 的 Pinning 章节](https://os.phil-opp.com/async-await/#pinning)
  - [Pin and suffering](https://fasterthanli.me/articles/pin-and-suffering) 一篇非常口语化的博客文章，关于理解异步代码和 pinning，有很多例子。
  - Jon Gjengset 的书 *Rust for Rustaceans* 对为什么 async/await 的实现需要 pinning 以及 pinning 如何工作有很好的描述。
