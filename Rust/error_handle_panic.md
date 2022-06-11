# RUST 错误处理（1）： Panic

Rust 的错误处理与众不同，在 Rust 中有两种错误处理方法：**Panic** 和 **Result**。

使用 **Result** 处理普通错误。**Result** 通常代表由程序外部的事物引发的错误，比如，错误的输入、网络问题或者权限问题等等。这种错误情况的的发生并不取决于我们，即使一个没有 bug 的程序，我们也会时不时遇到它们。

**Panic** 针对的是另外的一种错误，而这种错误 **应该** 永远不会发生（但并不代表它不会发生）。

本章将介绍一下 **Panic**。

# Panic

当一个程序遇到一些问题发生 panic。在这种情况下，程序本身必定存在一个 bug。比如说：

1. 越界的数组访问
2. 除 0 操作
3. 在恰好是 Err 的 Result 上调用 .expect()
4. 断言失败
5. panic macro


上述这些条件都有一个共同的特点：都是程序员的错。所以一个好的经验法则是：不要 panic。

但是我们总会犯错。当这些不应该发生的错误发生时，然后呢？ Rust 给了我们一个选择。当程序 panic 时，我们既可以 **展开堆栈（unwind the stack）**，也可以 **中止进程（abort the process）**。默认是 **展开堆栈（unwind the stack）**。

## Unwinding

当海盗瓜分战利品时，船长获得一半的战利品，普通船员平均分掉另一半的战利品。（海盗讨厌分数，如果除法除不尽，结果会向下取整，余数交给船上的鹦鹉保管）

```rust
fn pirate_share(total: u64, crew_size: usize) -> u64 {
    let half = total / 2;
    half / crew_size as u64
}
```
这个方法可能会正常工作很久很久，直到有一天海盗只剩下了船长一个人。

假设我们向这个函数中传入 crew_size = 0 时，就会产生 **除 0 操作**。在 c++ 中，**除 0 操作** 是未定义操作（undefined behavior）。在 Rust 中，通常会触发 panic，如下进行：

- 一个错误信息会打印到终端
    ```
    thread 'main' panicked at 'attempt to divide by zero', examples/panic.rs:7:5
    note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
    ```
    如果你按照错误信息的提示，设置了 RUST_BACKTRACE 环境变量，Rust 将会 dump stack。
    ```
    thread 'main' panicked at 'attempt to divide by zero', examples/panic.rs:7:5
    stack backtrace:
       0: rust_begin_unwind
                 at /rustc/34a6c9f26e2ce32cad0d71f5e342365b09f4d12c/library/std/src/panicking.rs:584:5
       1: core::panicking::panic_fmt
                 at /rustc/34a6c9f26e2ce32cad0d71f5e342365b09f4d12c/library/core/src/panicking.rs:142:14
       2: core::panicking::panic
                 at /rustc/34a6c9f26e2ce32cad0d71f5e342365b09f4d12c/library/core/src/panicking.rs:48:5
       3: panic::pirate_share
                 at ./examples/panic.rs:7:5
       4: panic::main
                 at ./examples/panic.rs:2:5
       5: core::ops::function::FnOnce::call_once
                 at /rustc/34a6c9f26e2ce32cad0d71f5e342365b09f4d12c/library/core/src/ops/function.rs:248:5
    note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
    ```
- 栈已经被展开。这有点像 c++ 的异常处理（exception handling）。

    当前函数正在使用的任何**临时变量**，**局部变量**或者**参数**都将被 drop 掉。drop 的顺序与创建的顺序正好相反。而在 pirate_share() 这个特殊的例子中，没有什么可以清理的。
    
    一旦完成当前函数调用的清理工作，我们就转到其调用者，以同样的方式进行清理。然后再移动到该函数的调用者，以此类推。
    
- 最后，这个线程退出了。如果 panic 的是主线程，那么整个程序都会退出（exit code 不为 0）。

也许 panic 是一个带有误导性的名称。Panic 不是 crash，也不是未定义行为（undefined behavior）。Panic 更像是 Java 中的 **RuntimeException** 或 C++ 中的 **std::logic_error**。Panic 的行为是明确定义的；它不应该发生。

Panic 是**安全**的。它不违反 Rust 的任何安全规则，即使你设法在标准库方法的中间 panic，它也不会在内存中留下悬垂指针或者半初始化的值。这个实现的想法大致是，在任何不好的事情发生之前， Rust 会捕获无效的数组访问，或者一些其他的错误。 继续执行下去是不安全的，所以 Rust 将会展开堆栈（unwind the stack）。 但该进程的其余部分可以继续运行。

Panic is pre thread。一个线程 panic 时，其他线程仍然可以正常的处理他们的业务。

还有一种方法可以捕获 stack unwinding，并允许线程存活且可以继续运行。标准库提供了一个方法做这件事情：**std::panic::catch_unwind()**。Rust 测试工具在遇到断言失败时恢复线程，便用到了它。在 c/c++ 中调用 Rust 代码时，可能也需要它，因为 unwind no-rust code 是未定义行为。

理想状态下，我们将会拥有永不 panic 的 bug-free 代码。但是人并不是完美的。您可以使用线程和catch_unwind() 来处理 panic，使程序更加健壮。**NOTE**，catch_unwind() 只能捕获 unwind stack 形式的 panic。 并非所有 panic 都以这种方式进行。


## Aborting
Stack unwinding 是默认的 panic 行为，但是在 Rust 中有两种情况，并不会 unwind the stack。

当 rust 在第一次 panic 后，如果 drop 方法触发了第二次 panic，这将是致命的。 Rust 会停止 unwinding，并且终止整个进程。

另外，Rust panic 的行为也可以自定义。如果编译时加入“-C panic=abort”，那么当你的程序发生第一个 panic 时，整个进程就会停止。（当加入该参数编译时，Rust 并不需要知道如何 unwind the stack，所以，这样做可以减少编译产物的大小）

我们对 Rust 中 panic 的讨论到此结束。确实没什么好说的，因为普通的 Rust 代码没有义务去处理 panic。即使你使用了线程和 catch_unwind()，你的 panic 的处理代码很有可能会集中到一起。期望程序中每个函数都可以去预测和处理代码中的错误是不合理的。


## Note
git 地址：https://github.com/Fengys123/err_handle