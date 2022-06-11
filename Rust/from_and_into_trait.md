# Rust 常用trait 1:  From and Into

**From** 和 **Into** trait 表示了一种转换的方式，即消费一个值，返回另一个值。**From** 和 **Into** 会获取参数的所有权，并对参数进行转换，然后将转换结果的所有权返回给调用者。

**From** 和 **Into** 的定义：（看上去非常对称）


```rust
trait Into: Sized { 
    fn into(self) -> T; 
} 
trait From: Sized { 
    fn from(other: T) -> Self; 
}
```
标准库自动实现了从每个类型到自身的简单转化： 每个类型 T 都实现了 **From< T >** 和 **Into< T >**。

尽管这些 traits 提供了两个方法做同一件事情，但是它们使用的场景却不一样。

# Into

你通常可以使用 **Into** 来使方法接受的参数更加的灵活。举个例子，你可以这么写：

```rust
use std::net::Ipv4Addr;

fn ping<A>(address: A) -> std::io::Result<bool>
where
    A: Into<Ipv4Addr>,
{
    let ipv4_addr = address.into();
    todo!()
}
```

这样 ping 不仅仅可以接受 Ipv4Addr 作为参数， 参数也可以是一个 u32，或者是一个 [u8; 4]。因为这些类型都实现了 **Into< Ipv4Addr >**。（有时候，将 Ipv4Addr 视为 一个 u32 或者是一个 [u8; 4]，将非常有用）。ping 方法知道参数 address 实现了 **Into< Ipv4Addr >**，所以在调用 into 方法时，无需指定你想要的类型。因为只有一个可以有效，因此，类型推断（type inference）会帮助你填充。

按照前面给出的 ping 方法的定义，我们可以这样去调用：

```rust
println!("{:?}", ping(Ipv4Addr::new(23, 21, 68, 141)));
println!("{:?}", ping([23, 21, 68, 141]));
println!("{:?}", ping(0x1715428d_u32));
```
这种类似于 C++ 的方法重载（overloading function）。

# From

然而 **From** trait 扮演着另外一种角色，常用于通用的构造函数：从一些其他单个的值生成一个实例。

举个例子：现在要实现 [u8; 4] 或 From< u32 > 到 Ipv4Addr的转化。简单的看，我们需要实现 Ipv4Addr::from_u32 和 Ipv4Addr::from_array 方法。调用方法如下所示：
~~~rust
let addr1 = Ipv4Addr::from_u32([23, 21, 68, 141]);
let addr2 = Ipv4Addr::from_array(0x1715428d_u32);
~~~

引入 **From** trait的话， 我们只需要简单的实现 **From< [u8; 4] >** 和 **From< u32 >** 这两个 trait，而不需要拥有 from_u32 和 from_array 方法。这允许我们这么写：

```rust
let addr1 = Ipv4Addr::from([23, 21, 68, 141]);
let addr2 = Ipv4Addr::from(0x1715428d_u32);
```

我们可以让类型推断（type inference）找出符合的实现。

给出一个合适的 **From** 实现，标准库会帮助我们自动的实现相对应的 **Into**。当我们定义自己的类型时，如果只有一个参数进行构造时，那么你应该为合适的类型实现 **From< T >**，同时你还可以 “免费” 获得相对应的 **Into** 实现。

# From and Into

由于 **From** 和 **Into** 的转换方式会获取参数的所有权，所以这种转化方式可以重用（reuse）参数的资源来构建转化后的对象。假设你这么写：

```rust
// cheap case
let text = "Beautiful Soup".to_string();
let bytes: Vec<u8> = text.into();
```

**Into< Vec<u8> > For String** 的实现简单的获取了 **String** 的 **heap buffer**，并未对 **heap buffer** 进行修改，也无需重新分配内存和复制文本。这是 **move enable efficient implementations** 的另一个例子。

这些转换方式提供了一种很好的方法来将受约束的类型 into 到一些更加灵活的类型中，而不会削弱受约束类型的保证。举个例子：**String** 保证了它的值总是有效的 **utf-8**，以及它的方法也被严格的限制，以确保您所做的任何事情都不会引入错误的 **utf-8**。在这个例子中，**String** 有效的 “降级” 成 **byte** 数组，你可以使用 **byte** 数组做任何你想做的事情：也许，你要压缩它，或者将它与其他的非 **utf-8** 的二进制数据结合（combine）起来。由于 **Into** 拿走了 **text** 的所有权，所以 **text** 在转换之后一直未初始化。这就意味着，我们可以在不破坏现存的 **String** 的前提下，可以自由的访问被转换者 **String** 的 buffer。

然而，“便宜” 的（cheap）转换并不是 **Into** 和 **From** trait 的合约的一部分。**AsRef** 和 **AsMut** 转换的预期是 “便宜” 的，而 **Into** 和 **From** 转换可能会进行内存的复制、分配或者是一些处理值的操作等等。举个例子，**String** 实现了 **From< &str >**，需要复制 **String slice** 到一个新分配的 **heap buffer** 中。再举一个例子：**std::collections::BinaryHeap** 实现了 **From< Vec< T > >**，需要根据其算法的要求比较和重新排序元素。

在 **From** 和 **Into** 未引入到标准库之前，Rust 的代码里充满了 ad hoc 转换，以及具有单个参数的构造函数。 **From** 和 **Into** 这种编码方式，会使你的类型更易于使用。其他的语言也可以采用这种模式进行类型之间的转换。

**From** 和 **Into** trait 规定转换不能失败（**Tryfrom** 和 **TryInto** 允许返回一个 Result）。很不幸的是，很多转换要复杂的多，可能会出现转换失败的场景。例如：一个 i64 类型的数据要转换为 i32 类型的数据。这样做，会丢失 i64 的前 32 位的数据，可能与我们的预期不符。

附：Rust 中有一个操作符 **？** 就使用了 **From** 和 **Into** 的转化方式，来帮助简化代码。