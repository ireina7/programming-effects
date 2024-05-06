# 计算效应：计算的构造与副作用

这是一篇个人在编程领域的感想，很多想法可能是错误的，但是作者本人认为其中仍然存在着一些价值，如果可以的话想记在这里，如果我没有时间去深入研究，也许可以给其他人带来一点新的视角。

我们大多被教育数据结构和算法是分开的，正如我们习惯把数据和计算分开一样；但是我认为这两者在概念上是没有区别的，正如在冯诺伊曼体系下程序也可以作为数据被读取和存储一样，计算本身也可以和数据一样拥有自己的结构和构造。可惜的是大多数的资料和书本大多不重视计算本身的结构化与抽象，仅仅在函数式编程和一些编程语言理论的资料中有所研究和涉及。其实计算的结构和数据结构一样有着非常大的价值，而且不仅仅是理论上，在擅长取舍的工程上甚至具有更大的意义。

计算的结构可以大概分为两个成分：函数和效应（副作用）。在函数式编程中，想必我们都已经很熟悉函数作为一个一等公民在程序中被构建和传递的能力。在函数的视角下，计算被划分成了函数组合（复合），这里的函数恰如数学中的函数，具有优良的可代换性质，任何一个函数调用可以等效替换为其调用结果而不改变程序语义。然而如果仅仅只有函数，那么我们将会错过编程之于数学相比多出来的最美妙的世界：副作用（或者如果你喜欢的话，可以称为IO）。一旦我们引入副作用，函数就不再符合数学一样的可代换性质，即使传入的参数相同，其结果也可能不同；并且我们不止关注函数调用的结果，函数调用的过程可能才是我们恰恰所需要的（考虑一个服务器的循环接受client请求）。加入了副作用，我们便无法随意替换表达式，无法随意重构和测试，复杂的副作用还会带来各种不确定性，程序的复杂性由此而生。

让我们把目光聚焦于计算的副作用，计算的副作用其实是计算效应（Computation Effects）的一部分，那么计算效应是什么呢？简单来说，计算效应就是除去函数输入和输出之外的一切（如果你想要严格定义的话会涉及到范畴论或一切其他的模型知识，不是我们的重点），如果换一种更直观的说法：
> 通过计算效应，程序员可以不关注函数的副作用与具体实现，仅关注业务逻辑本身。

## 简单IO的可测试性
平时我们所熟悉的带有副作用的函数往往是不会从类型（接口）层面体现出可能存在的计算效应的，比如：
```rust
fn get_char() -> u8;
```
单纯从函数签名来看看不出任何的IO，如果不是因为函数的名字叫`getchar`，甚至可以认为这个函数每次都返回相同的结果。想象一下我们有一个函数依赖这个`getchar`：
```rust
fn command() {
    let c = get_char();
    // ...
}
```
那么我们应当如何测试`command`函数呢？从stdin中手动输入对应的char？显然这是不够的，如果我们的char是通过网络来自另外一台计算机呢？显然，对于`command`，我们希望`get_char`可以有不同的实现，因此我们需要抽象：
```rust
trait GetChar {
    fn get_char() -> u8;
}

fn command<Io: GetChar>() {
    let c = Io::get_char();
    // ...
}
```
现在我们可以通过传入不同的`Io`来允许`command`内进行不同的`get_char`了：
```rust
// Simple local dummy
struct Local;
impl GetChar for Local {
    fn get_char() -> u8 {
        b'a'
    }
}

struct FromStdIn;
impl GetChar for Local {
    fn get_char() -> u8 {
        std::io::get_char() // Let's pretend the std::io has this fn
    }
}

#[test]
fn test_command() {
    command<Local>();
    command<FromStdIn>();
}
```
看起来计算效应直接用trait或接口就可以表示，好像并没有什么新东西？非也，计算效应远不止简单的接口抽象，我们忽略了另外一个东西：控制流。

## 错误处理与控制流
如果你和我一样碰巧需要使用`Golang`，你一定听说过著名（臭名昭著）的`if err != nil`。其本意是如果当前函数出错（i.e., `err != nil`），就直接返回跳出，不执行剩余的计算：
```golang
func example() error {
    ch, err := GetChar()
    if err != nil {
        return err
    }
    // 剩余的计算...
}
```
这个东西本身将错误作为value返回其实问题不大（甚至是正确的），不过其nil的判等机制很恶心，而且`err`的命名覆盖机制也有坑（不是我们的重点暂不讨论）。其实错误处理也是一种计算效应，它表示**我们的函数可能会出错的额外效应**。如果接口就可以抽象计算效应的话，那么拥有interface机制的golang应该就不需要反复手写`if err != nil`了才对，然而事实是golang程序员仍然需要每天手写大量的`if err != nil`且几乎没有任何通用的简化方式，原因就在于这里涉及到了控制流。

熟悉`Java`或`C++`的同学这里一定有话想说，我们用异常的try-catch不就没有这种烦恼吗？没错！异常就是一种可能出错的计算效应的建模。当当前的函数出错之后，控制流就会执行到catch块中，使用对应的handler处理之后抛弃掉原本的函数的下面要执行的上下文（continuation）。这里的*抛弃剩余的计算上下文*就是golang无法简单抽象错误处理的原因，因为仅通过接口无法控制接口方法返回之后的控制流。这里我们可以得到新的发现：计算效应至少需要拥有接口和剩余计算控制流的抽象能力。除去异常机制，Rust提供了`?`operator：
```rust
fn exmaple() -> Result<(), IoErr> {
    let c = get_char()?; // 如果出错这里会直接跳出函数并返回对应错误
    // 剩余的计算...
}
```

## 具像化计算效应的幽灵
那么我们该如何抽象控制流呢？还记得前面我们提到的“剩余的计算”吗？其实我们只要把这个continuation抽象出来（或者具像化），一切就十分显然了。让我们用一个闭包来表示这个剩余的计算：
```rust
fn command<Io: GetChar>() {
    let continuation = |a| {
        // 剩余的计算...
    }
    let c = Io::get_char();
    continuation(c)
}
```
这里仅仅是将剩余的计算明确表示成continuation，为了控制剩余的计算如何继续，我们需要一个额外的函数：
```rust
fn bind<A, B>(a: A, f: impl FnOnce(A) -> B) -> B;

fn command<Io: GetChar>() {
    let continuation = |a| {
        // 剩余的计算...
    }
    let c = Io::get_char();
    bind(c, continuation)
}
```
将`bind`抽象进我们的接口：
```rust
trait Bind {
    fn bind<A, B>(a: A, f: impl FnOnce(A) -> B) -> B;
}

fn command<Io: GetChar + Bind>() {
    let continuation = |a| {
        // 剩余的计算...
    }
    let c = Io::get_char();
    Io::bind(c, continuation)
}
```
于是我们发明了CPS（Continuation Passing Style）。
赋予了函数控制continuation的能力，剩下的就是给`get_char`的返回增加一点内容，让`bind`可以根据`c`来决定对continuation的用法。比如这里我们想根据`get_char`是否出错来控制控制流：
```rust
trait Bind {
    fn bind<A, B>(a: io::Result<A>, f: impl FnOnce(A) -> B) -> io::Result<B>;
}

fn command<Io: GetChar + Bind>() -> io::Result<()> {
    let c: io::Result<u8> = Io::get_char();
    Io::bind(c, |c| {
        // 剩余的计算...
    })
}
```
需要注意的是，如果`command`中任何一步出错了，我们仍然需要`command`本身将错误表达出来，所以需要把`command`的返回类型也变成`io::Result`，顺便`io::Result`的定义是：
```rust
enum Result<A, E> {
    Ok(A),
    Err(E),
}

mod io {
    type Result<A> = super::Result<A, IoErr>;
}
```

## 延迟计算：生成计算的描述
通过结合trait、CPS和返回额外数据，我们似乎已经完美解决了计算副作用的问题，我们不再关心函数计算的副作用，得以聚焦于逻辑本身。只是最后我们还剩一下最后一环：延迟计算。

让我们回到IO，IO除了可测试性和获取IO的渠道可多样性之外，还有一个重要的问题：IO操作相较CPU其它操作速度要慢上不少！如果我们将我们目前的模型应用于IO计算副作用就会发现我们不得不让我们的线程苦苦等待`get_char`的IO操作完成才能继续线程的其他工作，这显然是十分浪费线程资源的。由此引出了阻塞与非阻塞、同步于异步的概念。

这里如果熟悉rust的同学一定已经脱口而出：*async-await*！但是我们还没有抵达这么远，我们还需要一步一步抵达async的真实。如果我们仅仅聚焦于阻塞的问题，仅仅考虑如何最大化利用计算的资源，那么模型是十分多样的：IO复用、信号驱动IO、完全异步模型、事件循环etc。选择任何一种都是有可能的，但是我们不想让我们的程序被局限在任何一种方式中，因为程序本身的业务逻辑应该是独立于这种IO副作用的！我们希望程序员不需要关心副作用的同时，可以自由选择副作用的实现方式，同一份代码可以应用于各种各样不同的IO场景下！如果我们的`get_char`函数进行了实际的计算，尽管我们已经用trait进行了抽象，`get_char`可以使用不同的计算方式来实现，但是无论哪一种方式，其continuation都依赖了`c`，意味着continuation本身都不得不等待得到`get_char`的运算结果`c`之后才能继续运算！如果`get_char`没有运算完，也许`command`函数之外的continuation不依赖这里的逻辑，也许可以让其他计算先进行计算，不必让线程等待在这里。

## Async.await

## Monad：一种描述计算的范畴化结构

## 分而治之的泛化：计算的结构化

## Monad的组合问题

## Monad为什么不适合无GC的语言

## Tagless Final：抽象Monad

## Free monad：One monad for all

## Coroutine

## 代数计算效应

## One-shot effects

## 基本计算效应

## Trait-level monad

## Tokio



