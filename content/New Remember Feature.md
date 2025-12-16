---
title: 新的 remember 特性
date: 2025-12-15T16:56:00+08:00
---

# 新的 remember 特性

tessera v3的其中一个重要目标是引入remember机制，让tessera真正拥有带状态组件，避免传统即时模式ui的无条件状态提升问题。此功能已经完成并确定了api设计，现在我主要的工作内容转向了移植完整的md3组件库，这会花费不少时间，因此现在是一个恰当的时机（趁我印象深刻时）说明一下remember将为tessera带来的改变。

## 完全状态提升的问题

在之前，tessera更像一个用函数表达组件的imgui，我们没法在组件内部保存状态，因此所有状态都必须提升到组件外部。

当时一个带动画状态的组件签名看起来是这样的

```rust
pub fn switch(args: impl Into<SwitchArgs>, state: SwitchState);
```

其中`SwitchState`是一个结构体，包含了所有的状态字段，比如当前switch是否打开，打开的进度等。使用时，用户必须在组件外部创建并维护这个状态：

```rust
let switch_state = state.switch_state.clone();
switch(
    SwitchArgs::default(),
    switch_state,
);
```

这很不方便，毕竟大多数时候父组件根本不关心switch的状态，也不应该对它的状态做什么事，它只有两件事对父组件有意义：是否被打开，以及其样式。而这只需要一个SwitchArgs就足够表达了，额外传入一个SwitchState对父组件来说没有任何实际的作用。

### 状态膨胀和命名问题

下面是一个更复杂的例子，我们有一个layer0组件，里面包含一个layer1组件，layer1组件又包含一个switch组件。

```rust
struct Layer1State {
    switch_state: SwitchState,
}

#[tessera]
fn layer1(state: Layer1State) {
    let switch_state = state.switch_state.clone();
    switch(
        SwitchArgs::default(),
        switch_state,
    );
}

struct Layer0State {
    layer1_state: Layer1State,
}

#[tessera]
fn layer0(state: Layer0State) {
    let layer1_state = state.layer1_state.clone();
    layer1(layer1_state);
}
```

如上面的例子所示，状态提升会导致大量样板代码，尤其是当组件嵌套较深时，状态提升的样板代码会变得非常冗长，因为父组件的状态必须显式的包括每个子组件的状态。

更糟糕的是，状态提升还会导致组件的重用成为一大难题，因为你不得不在重用时给这个新的组件状态编一个名字。请看下面这个没那么多组件声明但是更加恼人的场景。

```rust
struct LayerState {
    switch1_state: SwitchState,
    switch2_state: SwitchState,
    switch3_state: SwitchState,
    // ...
}

#[tessera]
fn foo(state: FooState) {
    let switch1_state = state.switch1_state.clone();
    switch(
        SwitchArgs::default(),
        switch1_state,
    );
    let switch2_state = state.switch2_state.clone();
    switch(
        SwitchArgs::default(),
        switch2_state,
    );
    let switch3_state = state.switch3_state.clone();
    switch(
        SwitchArgs::default(),
        switch3_state,
    );
    // ...
}
```

_在计算机科学中只有两个难题：命名问题和缓存失效。_ 而这就是一个命名问题。现在我们不得不给每一个带状态组件的状态命名，而这状态又完全不参与父组件的逻辑，单纯就是为了传给子组件，没有人能真正搞明白它到底应该叫什么，以及它除了让状态变成一团糟以外是否有任何意义。

### Clone地狱

另一个问题是Clone地狱。

在tessera中，组件渲染到屏幕的过程分为四个阶段：

- 构建阶段: 执行组件函数，根据调用栈自下而上自然产生组件树
- 测量和放置阶段: 计算每个组件的大小和位置，部分基础组件生成对应的绘制命令
- 事件处理阶段: 处理用户输入事件，更新组件状态
- 绘制阶段: 将绘制命令提交给GPU进行渲染

其中前三个阶段都有访问组件状态的需求。为了保证状态在这四个阶段都能被访问到，同时允许测量和放置阶段是并行的，我们一般会使用`Arc<Lock<T>>`来包装组件状态。但是，作为一个函数式的ui框架，tessera还大量使用FnOnce表示子组件的类型。而FnOnce闭包捕获的变量会被移动到闭包内部，必须使用在它move进去之前clone来避免所有权问题。

也许一个例子可以更好的说明，假设我们有一个带状态的动画组件，它每次被点击都变宽一个像素。

```rust
struct AnimatedBoxState {
    width: Arc<AtomicUsize>,
}

#[tessera]
fn animated_box(state: AnimatedBoxState) {
    let width = state.width.clone();

    let width_for_measure = width.clone();
    measure(Box::new(move |input| {
        let width = width_for_measure.load(Ordering::SeqCst);
        let width = Px::new(width as i32);
        Ok(ComputedData {
            height: Px::new(100),
            width,
        })
    }));

    let width_for_input = width.clone();
    input_handler(Box::new(move |input| {
        let pressed = // ... check if box is pressed ...
        if pressed {
            width_for_input.fetch_add(1, Ordering::SeqCst);
        }
        input.cursor_events.clear();
    }));
}
```

width_for_measure和width_for_input都必须clone一份width，现在想象你在一个column里面使用50个animated_box组件，而且想要给每个item都复用同一个状态。那你就得写50个`let animated_box_state_for_xxx = animated_box_state.clone()`。这就是Clone地狱。

## 记忆化状态

remember的作用是在组件内创建跨帧持久的状态。

```rust
#[tessera]
fn counter() {
    let count = remember(|| 0);
    count.with_mut(|c| *c += 1);
}
```

现在，counter组件第一次被构建时，remember会创建一个初始值为0的状态，并在每次渲染时使它自增1。直到它在某帧不被构建时，这个状态才会被销毁。

熟悉react或者compose的读者可能会觉得这个api很眼熟，它的设计的确和react的useState以及compose的remember大同小异。不过与react hooks不同的是，remember的限制要小的多，它的使用不必遵循[`rules-of-hooks`](https://react.dev/reference/rules/rules-of-hooks)，你可以在if, loop, match等控制流语句中安全的使用它。

不过如虚拟列表等组件可能不会每帧都构建所有子组件，而且如果仅使用remember会导致状态漂移到下个可见组件实例中去。这种情况下需要使用key来指定范围内的稳定标识符，从而让remember正确的将状态和组件实例绑定在一起。

remember完美的解决了之前提到的状态提升问题，现在可以将对父组件没有意义的状态直接放在子组件内部。这意味着之前的layer0/layer1/switch例子可以变为下面这样：

```rust
#[tessera]
fn layer1() {
    switch(SwitchArgs::default());
}

#[tessera]
fn layer0() {
    layer1();
}
```

同理，多个switch组件的例子的命名问题也得到了解决。

```rust
#[tessera]
fn foo() {
    switch(SwitchArgs::default());
    switch(SwitchArgs::default());
    switch(SwitchArgs::default());
}
```

进一步的，remember还解决了Clone地狱的问题，关键在于其返回值，remember返回的并不是一个`Arc<Lock<T>>`，而是`State<T>`，它是一个轻量级的Copy句柄，因此可以轻易的move进FnOnce，而不需要进行任何所有权转移或者Clone。

现在，上面的animated_box例子可以变成下面这样：

```rust
#[tessera]
fn animated_box() {
    let width = remember(|| Px::new(100));

    measure(Box::new(move |input| {
        Ok(ComputedData {
            height: Px::new(100),
            width: width.get(),
        })
    }));

    input_handler(Box::new(move |input| {
        let pressed = // ... check if box is pressed ...
        if pressed {
            width.with_mut(|w| {
                *w += Px::new(1);
            });
        }
        input.cursor_events.clear();
    }));
}
```

## 平行特性

除了remember以外，tessera v3还引入了一些类似，但并不完全相同的特性，比如为主题系统准备的context特性，它和remember正好正交。context用于在组件树中跨越多个层级传递数据，但不能跨越帧持久化状态，而remember用于在组件内部创建跨帧持久化状态，但不能跨越组件树层级传递数据。这些特性结合起来，将大大优化tessera的状态管理体验和代码简洁度。

## 实现原理

remember机制实际上就是**位置化记忆**。据我所知，此机制在rust ui中最早的探索是Raph Levien的[crochet](https://github.com/raphlinus/crochet)。它使用当时还很新的特性`#[track_caller]`来做到这一点（这里不讨论dioxus等库使用的hooks机制，虽然它们也算广义上的位置化机制，但是会有更多的限制）。`#[track_caller]`允许函数通过`std::panic::Location::caller()`获取调用它的源代码位置，从而可以用这个位置作为状态的唯一标识符，这其实是可行的，然而有着不少局限，而且它终归是`std::panic`的一部分，设计上并不是为这种用途。

因此tessera的remember机制并没有使用`#[track_caller]`，而是使用了更复杂但是更可靠的过程宏分析机制。`#[tessera]`过程宏会对代码的if，loop，match等控制流进行改写以插入GroupGuard。各种控制流被处理后实际上是这样的:

```rust
if condition {
    let _group_guard = ::tessera_ui::runtime::GroupGuard::new(#group_id);
    original_statement;
}

for item in iterator {
    let _group_guard = ::tessera_ui::runtime::GroupGuard::new(#group_id);
    original_statement;
}

match value {
    Pattern1 => {
        let _group_guard = ::tessera_ui::runtime::GroupGuard::new(#group_id_1);
        original_statement_1;
    }
    Pattern2 => {
        let _group_guard = ::tessera_ui::runtime::GroupGuard::new(#group_id_2);
        original_statement_2;
    }
    // ...
}

loop {
    let _group_guard = ::tessera_ui::runtime::GroupGuard::new(#group_id);
    original_statement;
}
```

根据RAII特性。`_group_guard`在作用域结束时会被立马drop，而它的Drop实现将告诉tessera当前的控制流块已经结束。通过这种方式，tessera可以精确的追踪每个remember调用的控制流位置，从而为每个remember调用生成一个唯一的，基于实际运行时控制流位置的标识符，从而实现可靠的状态化记忆。

不幸的是，由于宏的展开顺序局限性，目前tessera的remember机制还不能很好地支持在声明宏内部被使用的情况。因为宏在此时还没有被展开，tessera过程宏无法得知宏内部的控制流结构，因此无法为remember调用生成正确的标识符。目前只能避免在宏内部使用remember，或者调用组件，如果要使用，则声明宏生成的代码不能有影响remember的控制流。这需要未来rust支持以某种方式影响宏展开顺序才能解决。

而状态保存，即`State<T>`，则是一个可控的Arena gc实现。它的核心思想可以参考[gc-arena](https://github.com/kyren/gc-arena)。tessera使用的则是一个特化的精简实现，每帧根据remember的调用情况标记存活状态，未被标记的状态则在帧结束时被清除。
