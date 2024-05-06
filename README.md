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

这里显然超出了`bind`的能力了，我们需要的是一个第三方运行时，它可以观测到计算的结构，并自动进行调度！所以下面的问题在于我们需要把计算以某种形式的结构呈现给一个运行时，并委托其进行调度计算，同时最大化复用计算资源。这种计算的形式可以看作是一种用数据结构来表述计算的一种描述。

## 异步计算的描述
接下来我们将进入异步计算的世界。异步计算最早使用了回调（callback）来进行编写，像极了我们的`bind`模式：
```js
let fut = io.getChar()
fut.andThen(c => {
    // 剩余计算...
})
```
`fut`就是对计算的一种描述，然后我们通过类似`bind`的方法`andThen`在`fut`的基础上搭建更大的数据结构来表示剩余的计算，最后这个`fut`就可以交给一个`Runtime去跑，自动化完成IO。但是我们现在既不知道这个结构该如何构造，也不知道Runtime该如何实现。关于如何去表示计算，有两种比较常见的方式：有栈与无栈协程（coroutine）。

### Stackful Coroutine
计算的构造其实在操作系统中就已经有了！那就是我们的stack和各种表示计算上下文的register（如pc）。我们也许可以继续拓展这一概念，把剩余的计算表示成这种数据结构，任何IO计算显式地告诉Runtime自己可能会阻塞，那么runtime就可以自动yield当前计算，让其他计算先进行，然后将被阻塞的剩余的计算保存下来，等待IO结束恢复原本的计算。

实际上这就是Golang的goroutine实现，同时也是golang得以流行的核心原因（不是大道智减！）。然而有栈协程并不能解决所有问题，栈显然太重了，很多任务可能并不希望将整个栈都保存下来，尤其是在一些嵌入式领域，我们希望有更轻量的计算结构。

### Async.await
问题集中到了最小化计算结构的表示上。实际上目前已经有了一个成熟的方案：状态机。
我们直接将计算表示成一种新的数据结构，这个数据结构仅仅包含计算所必须的数据，然后通过runtime来将计算表示成不同的阶段和状态，从而成了一个状态机。这里我没有时间非常详细地阐述rust的方案了，其Future和Pin的设计更是比较复杂，感兴趣的话可以参考：[async-await in OS](https://os.phil-opp.com/async-await/)。

### 为什么颜色是个好东西
让我们回到函数副作用的视角上，很多人喜欢问：
> What color is your function?

很多人推崇golang一样的没有颜色的、非侵入式的异步方式，但是却没有注意，将有副作用的计算与无副作用的计算区分开具有十分重大的意义。有时间的话我会进一步阐述我的观点：为什么将计算效应表示出来是有价值的。

## Monad：一种描述计算的范畴化结构
铺垫了这么多饺子，其实就是为了*monad*这碟醋。既然已经解决了如何用数据结构表示计算，下面就是如何让我们的计算效应可以同时表达多种结构，比如IO和错误处理。让我们把目光聚焦回`bind`：
```rust
fn bind<A, B>(
    ma: todo!("某种确定了返回结果类型为`A`的计算结构"),
    continuation: impl FnOnce(A) -> todo!("某种确定了返回结果类型为`B`的计算结构"),
) -> todo!("某种确定了返回结果类型为`B`的计算结构"),;
```
这里的`todo!("某种确定了返回结果类型为`A`的计算结构")`可以表达任何一种计算效应的数据结构，我们如何从类型上去表示呢？

### Higher-Kinded Type
计算效应的抽象就藏在类型系统中，这种类型叫做Higher-Kinded Type（HKT）。
这一概念其实十分简单，如果说`fn`函数是value层面的函数，那么HKT就是类型层面的函数，其接受一个类型参数，返回一个新的类型，并且在编译期就完成了计算。比如在Haskell中，我们的`Functor`:
```haskell
class Functor f where
    fmap: (a -> b) -> f a -> f b
```
让我们直接翻译到Rust:
```rust
trait Hkt {
    type Ap<T>;
}

trait Functor<F: Hkt> {
    fn fmap<A, B>(f: impl FnOnce(A) -> B, fa: F::Ap<A>) -> F::Ap<B>;
}
```
由于Rust并不直接支持HKT，但是我们还有GAT！通过GAT我们一样可以完美模拟HKT。
这里的HKT就如`Java`中的`ArrayList`，就如`C++`中的`vector`，接受内部元素的类型`T`，返回新的类型`vector<T>`。

于是我们可以抽象我们的`bind`了！
```rust
trait Monad<M: Hkt> {
    fn bind<A, B>(ma: M::Ap<A>, f: impl FnOnce(A) -> M::Ap<B>) -> M::Ap<B>;
```
或者也许用Haskell的语法更清晰些：
```haskell
class Monad where
    bind: m a -> (a -> m b) -> m b
```
如果我们想处理错误，那么`M`就是：
```rust
struct ErrorHandle;

impl Hkt for ErrorHandle {
    type Ap<T> = io::Result<T>;
}
```
如果我们想处理IO：
```rust
struct GetCharFuture {
    // ...
}

impl Hkt for GetCharFuture {
    type Ap<T> = Box<dyn Future<Output = u8>>;
}
```
只是这里为了表示Future，引入了一点额外的Box dyn开销。

> Monad的完整定义不止bind：
> ```haskell
> class Monad m where
>   pure :: a -> m a
>   bind :: m a -> (a -> m b) -> m b
> ```
> 并且还需要满足一些代数性质，感兴趣的话可以了解范畴论

### 嵌套「嵌套『嵌套「...」』」
前面的例子都太简单了，以至于我们还没有触及到javascript程序员喜欢提到的回调地狱（callback hell）。让我们回到之前的错误处理例子：
```rust
fn command<Io: GetChar + Bind>() -> io::Result<()> {
    let c: io::Result<u8> = Io::get_char();
    Io::bind(c, |c| {
        // 剩余的计算...
    })
}
```
下面除了`get_char`之外，我们引入更多的可能出错的操作：
```rust
fn command<Io: GetChar + Bind>() -> io::Result<()> {
    let c: io::Result<u8> = Io::get_char();
    Io::bind(c, |c| {
        Io::bind(parser::parse(c), |s| {
            Io::bind(send(s), |ack| {
                println!("{:?}, {:?}, {:?}", c, s, ack);
                // 剩余的计算...
            })
        })
    })
}
```
仅仅三个调用就有了这么多嵌套！随着函数逻辑的复杂化，嵌套将嵌套嵌套，我想没有人会喜欢这种写法，所以Rust使用了`?`operator：
```rust
fn command<Io: GetChar + Bind>() -> io::Result<()> {
    let c: u8 = Io::get_char()?;
    let s = parser::parse(c)?;
    let ack = send(s)?;
    println!("{:?}, {:?}, {:?}", c, s, ack);
    // 剩余的计算...
}
```
清晰多了！然而`?`只能针对`Try`trait，如果是IO呢？IO我们有`.await`：
```rust
async fn command<Io: GetChar + Bind>() -> io::Result<()> {
    let c: u8 = Io::get_char().await?;
    let s = parser::parse(c)?;
    let ack = send(s).await?;
    println!("{:?}, {:?}, {:?}", c, s, ack);
    // 剩余的计算...
}
```
如果是更多的计算效应呢？Haskell发明了*do-notation*：
```haskell
command :: Monad m => m ()
command = do
    c <- getChar
    s <- parse c
    ack <- send s
    println "{:?}, {:?}, {:?}" c s ack
    -- 剩余的计算...
```
这里的`m`可以是任意满足Monad的HKT，即适用于所有构成Monad的计算效应。
我不打算深入阐述这里的实现（其实就是宏变换），其实我们并不一定需要用do-notation来表达通用的计算效应，下文会进一步阐述这里。

## 分而治之的泛化：算法的结构化
在进入下一个话题之前，我想写下我在算法领域的一点小小的发现，即：
> 几乎所有的递归问题都可以通过把分而治之和计算效应结合起来解决。

详细的阐述足够写一本书，通过计算效应我们甚至可以发展出千奇百怪的数据结构。这里我仅表达其中的核心思想：
> 每当你发现直接分治不足以合并出来更大的问题的解时，使用计算效应的数据结构扩展返回的类型，直到可以组合出更大问题的计算效应。

## Monad的组合问题
Monad看起来似乎很美好？NO！Monad居然不是可以随意组合的！？
考虑我们拿到了`M`, `N`两个monad，并打算组合它们：
```rust
struct Compose<M: Hkt, N: Hkt> {
    _data: PhantomData<M::Ap<N::Ap<()>>>,
}

impl<M: Hkt, N: Hkt> Hkt for Compose<M, N> {
    type F<A> = M::Ap<N::Ap<A>>;
}


impl<M: Monad, N: Monad> Monad for Compose<M, N> {
    // how???
}
```
理论上想给任意两个monad实现组合之后的bind是无法实现的（你可以尝试一下）。
这意味着无法随意组合不同的monad，想象一下一个lib使用`M`，另一个使用了`N`，它们却无法组合。。。这显然会在实践中带来问题，于是Haskell又发明了monad transformer...这里我不打算深入monad transformer，其核心思想就是虽然任意两个不能实现组合，但是只要确定了其中一个，就可以实现组合。可即使是这样，monad组合在实践中还有很多其他问题，比如如何确定组合嵌套的顺序问题。

## Monad为什么不适合无GC的语言
Monad除了存在组合问题，它甚至不适合无gc的语言，准确来讲，不适合存在用生命周期进行资源管理的语言。考虑下面这样一个嵌套结构：
```rust
type Nested<A> = M<'a, M<'b, M<'c, A>>>;
```
我们知道，monad可以对这种洋葱结构进行折叠，但是rust的生命周期约束要求：
```rust
'c: 'b : 'a
```
这意味着`Nested<A>`可能存在内外嵌套结构的生命周期依赖，`'c`必须比`'b`生命周期更长，monad不能随意折叠。因此rust不会大量使用monad作何抽象的核心。

## Tagless Final：抽象Monad
对于第一个monad组合问题，我们还找到了另一个更practical的解，即
> 将monad全部抽象出来，使得程序对任意monad成立



## Free monad：One monad for all

## 代数计算效应

## One-shot effects

## 基本计算效应

## Higher-level monad

## Mio & Tokio



