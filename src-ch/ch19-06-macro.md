# 宏

我们使用了像这样的宏 println!贯穿本书，但我们还没有完全 探索了宏是什么以及它是如何工作的。 *macro*一词 是指 Rust 家族的特性： *declarative*(声明性) 宏 `macro_rules!` 和三种 *procedural* (过程宏)： 

- 自定义 #[derive] 宏，用于struct 或 enum, 可以为其指定随 derive 属性添加的代码
- 类似属性的宏， 在任何条目上添加自定义属性
- 类似函数的宏， 看起来像函数调用， 对其指定为参数的 token 进行操作

我们将依次讨论这些，但首先，让我们看看为什么我们甚至 当我们已经有函数时需要宏。

### 宏与函数的区别

从根本上说，宏是一种编写代码的方式，它可以编写其他代码， 被称为*metaprogramming*元编程 。 在附录 C 中，我们讨论了`derive`属性，它为您生成各种特征的实现。 我们已经使用了`println!`和`vec!`贯穿全书的宏。 所有这些 宏扩展*expand*以生成比您手动编写的代码更多的代码。

*Metaprogramming*元编程对于减少您必须编写的代码量和 维护，这也是函数的作用之一。 但是，宏有一些功能没有的额外能力。

函数签名必须声明函数的参数数量和类型。 另一方面，宏可以采用*可变数量的参数*：我们可以调用有一个参数的`println!("hello")`或有两个参数的`println!("hello {}", name)`。 此外，宏被扩展在编译器解释代码的含义之前，所以一个宏可以做例如，在给`绑定标注`的类型上默认实现一些`trait`。 一个函数不能，因为它得到 在运行时调用，并且需要在编译时实现特征。

用`marco`来实现一些功能，而不是函数，所带来的唯一缺点，就是定义更为复杂，因为您正在使用Rust编写 Rust的代码。 由于这种间接性，宏定义通常比函数更难阅读、理解和维护。

宏和函数之间的另一个重要区别是：定义宏或将它们带入作用域*之前*在文件中调用它们。与之对应的函数，则可以定义任何地方定义，并在任何地方调用。

### 声明性宏 `macro_rules!` 通用元编程

Rust中使用最广泛的宏形式是*declarative macros*声明性宏 。 有时也称为`macros by example示例宏`、`macro_rules!宏`或只是简单的`macro宏`。 在其核心，声明性宏允许您编写 类似于Rust `match`表达式的代码。 

如第 6 章所述， 
`match`*表达式*是采用表达式的控制结构，将表达式的结果值与模式进行比较，然后运行与匹配模式关联的代码。
`macro`*宏*将一个*值*（传递给宏的参数）与*特定代码关联的模式*进行比较：在这种情况下，传递给宏的值（参数）是Rust源代码（字面量）；将这个*模式*与*该源代码的结构*进行比较； *并且与每个模式关联的代码在匹配时会替换传递给宏的代码*。这一切都发生在*编译*期间。 

要定义`macro`，请使用宏`macro_rules!`构造。 让我们通过查看 `vec!` 宏的定义来探索如何使用 `macro_rules!`。第 8 章介绍了如何使用 `vec!` 宏来创建具有特定值的集合向量vector。例如，以下宏创建一个包含三个整数的集合向量vector： 

```rust
let v: Vec<u32> = vec![1, 2, 3];
```

我们还可以使用 `vec!` 宏来制作两个整数的集合向量或五个字符串切片的集合向量。 我们不能使用函数来做同样的事情，因为我们事先不知道值的数量或类型。 

示例 19-28 显示了一个稍微简化的定义 vec!宏。

<span class="filename">Filename: src/lib.rs</span>
```rust
#[macro_export]
macro_rules! vec {
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}
```
示例 19-28：简化版 vec!宏 定义
```
    注：实际定义 vec!标准库中的宏 包括预先分配正确内存量的代码。 那个代码 是一个优化，我们没有在此处包含以使示例更简单。
```
`#[macro_export]` 注释表明，只要将定义宏的 crate 带入范围，就应该使该宏可用。 如果没有这个注解，宏就不能被引入作用域。 

然后我们以`macro_rules!`开始宏定义，我们定义的宏的名称*不带*感叹号。 名称，在本例中为 `vec`，后跟大括号，表示宏定义的主体。 （其实就是在编译期间，自动写入的代码）

`vec!` 主体中的结构类似于 `match` 表达式的结构。 在这里，我们有一个带有 `( $( $x:expr ),* )` 模式的句柄，然后是 `=>` 和与此模式相关的代码块。 如果模式匹配，将发出相关的代码块。 鉴于这是此宏中唯一的模式，因此只有一种有效的匹配方式； 任何其他模式都会导致错误。 更复杂的宏将有不止一个句柄。 

宏定义中的有效模式语法与第 18 章中介绍的模式语法不同，因为宏模式与 Rust 代码结构而不是值匹配。 让我们看一下示例 19-28 中的模式图片的含义； 有关完整的宏模式语法，请参阅 [参考资料](../reference/macros-by-example.html)。 


首先，一组括号包含整个模式。 接下来是一个美元符号 (`$`)，后跟一组括号，这些括号捕获与括号内的模式匹配的值，以用于替换代码。 在 `$()` 中是 `$x:expr`，它匹配任何 Rust 表达式并将表达式命名为 `$x`。 

`$()` 后面的逗号表示文字逗号分隔符可以可选地出现在与 `$()` 中的代码匹配的代码之后。 `*` 指定模式匹配 `*` 之前的零个或多个参数。

当我们用 `vec![1, 2, 3];` 调用这个宏时，`$x` 模式与三个表达式 `1`、`2` 和 `3` 匹配了 3 次。

现在让我们看看与这个句柄相关联的代码主体中的模式： `$()*` 中的`temp_vec.push()` 是为每个与模式中的`$()` 匹配零次或多次的部分生成的 取决于模式匹配的次数。 `$x` 被每个匹配的表达式替换。 当我们用 `vec![1, 2, 3];` 调用这个宏时，生成的替换这个宏调用的代码如下：

```rust
{
    let mut temp_vec = Vec::new();
    temp_vec.push(1);
    temp_vec.push(2);
    temp_vec.push(3);
    temp_vec
}
```

我们定义了一个宏，它可以接受任意数量的任何类型的参数，并且可以生成代码来创建一个包含指定元素的向量。 

`macro_rules!` 有一些奇怪的特例情况。 将来，Rust 将有第二种声明性宏，它会以类似的方式工作，但会修复其中一些特例情况。 在那次更新之后，`macro_rules!` 将被有效弃用。 考虑到这一点，以及大多数 Rust 程序员将*使用*宏而不是*编写*宏的事实，我们将不再讨论“macro_rules！”。 要了解有关如何编写宏的更多信息，请查阅在线文档或其他资源，例如由 Daniel Keep 开始并由 Lukas Wirth 继续的 [“The Little Book of Rust Macros”][tlborm](https://veykril.github.io/tlborm/)。 

### 从属性生成代码的过程宏

宏的第二种形式是*procedural macros过程宏*，它的作用更像函数（并且是一种过程）。 过程宏接受一些*代码*作为输入，对该代码进行操作，并*生成一些代码*作为输出，而不是像*declarative macro*声明性宏那样与模式匹配并用其他代码替换代码。 

三种过程宏（自定义派生宏(Custom Macro)、类属性宏(attribute-like) 和 类函数宏(function-like)） 都以类似的方式工作。

创建过程宏时，定义必须驻留在它们自己的具有特殊 crate 类型的 crate 中。 这是出于复杂的技术原因，我们希望在未来消除这些原因。 定义过程宏类似于示例 19-29 中的代码，其中 `some_attribute` 是使用特定宏种类的占位符。 

<span class="filename">Filename: src/lib.rs</span>

```rust,ignore
use proc_macro;

#[some_attribute]
pub fn some_name(input: TokenStream) -> TokenStream {
}
```

<span class="caption">
示例 19-29：使用程序的示例 宏
</span>

定义过程宏的函数将`TokenStream`作为输入，并生成`TokenStream`作为输出。 `TokenStream` 类型由 Rust 包含的 `proc_macro` crate 定义，表示一系列标记。 这是宏的核心：宏操作的源代码构成了输入`TokenStream`，宏产生的代码是输出`TokenStream`。 该函数还附加了一个属性，用于指定我们正在创建哪种过程宏。 我们可以在同一个 crate 中拥有多种过程宏。 

让我们看看不同类型的过程宏。 我们将从自定义派生宏开始，然后解释使其他形式不同的细微差异。 

### 如何使用derive编写自定义宏(Custom Macro)

让我们创建一个名为 `hello_macro` 的 crate，它定义了一个名为 `HelloMacro` 的 trait 和一个名为 `hello_macro` 的关联函数。 与其让我们的 crate 用户为他们的每个类型实现 `HelloMacro` 特征，我们将提供一个过程宏，以便用户可以使用 `#[derive(HelloMacro)]` 注释他们的类型以获得 `hello_macro 的默认实现 `函数。 默认实现将打印 `Hello, Macro! 我的名字是 TypeName!`，其中 `TypeName` 是定义了这个 trait 的类型的名称。 换句话说，我们将编写一个 crate，使另一个程序员可以使用我们的 crate 编写示例 19-30 之类的代码。 

<span class="filename">Filename: src/main.rs</span>

```rust,ignore,does_not_compile
use hello_macro::HelloMacro;
use hello_macro_derive::HelloMacro;

#[derive(HelloMacro)]
struct Pancakes;

fn main() {
    Pancakes::hello_macro();
}
```

<span class="caption">
示例 19-30：我们 crate 的用户将能够使用的代码 在使用我们的过程宏时编写
</span>

此代码将打印Hello, Macro! My name is Pancakes!当我们完成。 第一步是制作一个新的库 crate，如下所示： 

```console
$ cargo new hello_macro --lib
```

接下来，我们将定义 HelloMacro特征及其相关功能：

<span class="filename">Filename: src/lib.rs</span>

```rust,noplayground
pub trait HelloMacro {
    fn hello_macro();
}
```

我们有一个特点和它的功能。 此时，我们的 crate 用户可以实现 实现所需功能的特征，如下所示：

```rust,ignore
use hello_macro::HelloMacro;

struct Pancakes;

impl HelloMacro for Pancakes {
    fn hello_macro() {
        println!("Hello, Macro! My name is Pancakes!");
    }
}

fn main() {
    Pancakes::hello_macro();
}
```

然而，他们需要为他们想要与 `hello_macro` 一起使用的每种类型编写实现块； 我们希望让他们不必做这项工作。

此外，我们还不能为 `hello_macro` 函数提供默认实现来打印实现 trait 的类型的名称：Rust 没有反射功能，因此它无法在运行时查找类型的名称 . 我们需要一个宏来在编译时生成代码。

下一步是定义*procedural macro过程宏*。 在撰写本文时，*过程宏*需要在它们自己的 crate 中。 最终，这个限制可能会被取消。 构造 crate 和宏 crate 的约定如下：对于名为 `foo` 的 crate，自定义派生过程宏 crate 称为 `foo_derive`。 让我们在 `hello_macro` 项目中创建一个名为 `hello_macro_derive` 的新 crate： 

```console
$ cargo new hello_macro_derive --lib
```

我们的两个 crate 紧密相关，因此我们在 `hello_macro` crate 的目录中创建了过程宏 crate。如果我们更改 `hello_macro` 中的 trait 定义，我们也必须更改 `hello_macro_derive` 中的过程宏的实现。这两个 crate 需要单独发布，使用这些 crate 的程序员需要将*两者*都添加为依赖项并将它们都纳入范围。我们可以让 `hello_macro` crate 使用 `hello_macro_derive` 作为依赖项并重新导出过程宏代码。但是，我们构建项目的方式使程序员可以使用 `hello_macro`，即使他们不想要 `derive` 功能。

我们需要将 `hello_macro_derive` crate 声明为过程宏 crate。我们还需要来自 `syn` 和 `quote` crates 的功能，稍后您会看到，因此我们需要将它们添加为依赖项。将以下内容添加到 `hello_macro_derive` 的 *Cargo.toml* 文件中： 

<span class="filename">Filename: hello_macro_derive/Cargo.toml</span>

```toml
[lib]
proc-macro = true

[dependencies]
syn = "1.0"
quote = "1.0"
```

要开始定义过程宏，请将清单 19-31 中的代码放入 `hello_macro_derive` crate 的 *src/lib.rs* 文件中。 请注意，在我们为 `impl_hello_macro` 函数添加定义之前，此代码不会编译。 

<span class="filename">Filename: hello_macro_derive/src/lib.rs</span>

```rust,ignore,does_not_compile
extern crate proc_macro;

use proc_macro::TokenStream;
use quote::quote;
use syn;

#[proc_macro_derive(HelloMacro)]
pub fn hello_macro_derive(input: TokenStream) -> TokenStream {
    // Construct a representation of Rust code as a syntax tree
    // that we can manipulate
    let ast = syn::parse(input).unwrap();

    // Build the trait implementation
    impl_hello_macro(&ast)
}
```
<span class="caption">
示例 19-31：大多数过程宏 crate 的代码 将需要为了处理 Rust 代码
</span>

请注意，我们将代码拆分为负责解析`TokenStream`的`hello_macro_derive`函数和负责转换语法树的 `impl_hello_macro`函数：这使得编写过程宏更方便。 外部函数中的代码（在这种情况下为`hello_macro_derive`）对于您看到或创建的几乎每个*过程宏*包都是相同的。 您在内部函数主体中指定的代码（在本例中为`impl_hello_macro`）将根据您的*过程宏*的用途而有所不同。

我们引入了三个新的 crate：`proc_macro`、[`syn`] 和 [`quote`]。 `proc_macro` crate 是 Rust 自带的，所以我们不需要将它添加到 *Cargo.toml* 中的依赖项中。 `proc_macro` crate 是编译器的 API，它允许我们从代码中读取和操作 Rust 代码。 

[`syn`]: https://crates.io/crates/syn
[`quote`]: https://crates.io/crates/quote


`syn` crate 将 Rust 代码从字符串解析为我们可以对其执行操作的数据结构。 `quote` crate 将 `syn` 数据结构转换回 Rust 代码。这些 crate 使解析我们可能想要处理的任何类型的 Rust 代码变得更加简单：为 Rust 代码编写完整的解析器并非易事。

当我们库的用户在类型上指定 `#[derive(HelloMacro)]` 时，将调用 `hello_macro_derive` 函数。这是可能的，因为我们在这里用 `proc_macro_derive` 注释了 `hello_macro_derive` 函数，并指定了与我们的特征名称匹配的名称 `HelloMacro`；这是大多数过程宏遵循的约定。

`hello_macro_derive` 函数首先将 `input` 从 `TokenStream` 转换为数据结构，然后我们可以对其进行解释和执行操作。这就是 `syn` 发挥作用的地方。 `syn` 中的 `parse` 函数接受一个 `TokenStream` 并返回一个 `DeriveInput` 结构体，表示解析后的 Rust 代码。示例 19-32 显示了我们通过解析 `struct Pancakes;` 字符串得到的 `DeriveInput` 结构的相关部分： 

```rust,ignore
DeriveInput {
    // --snip--

    ident: Ident {
        ident: "Pancakes",
        span: #0 bytes(95..103)
    },
    data: Struct(
        DataStruct {
            struct_token: Struct,
            fields: Unit,
            semi_token: Some(
                Semi
            )
        }
    )
}
```

<span class="caption">
示例 19-32： DeriveInput我们得到的实例是什么时候 解析具有示例 19-30 中宏属性的代码
</span>

这个结构体的字段表明，我们解析的 Rust 代码是一个*单元结构体*`unit struct`，带有 `Pancakes` 的 `ident`（标识符，意思是名称）。这个结构体上有更多字段用于描述各种 Rust 代码；查看 [`Syn` 文档中的 `DeriveInput`][syn-docs] 了解更多信息。

[syn-docs]：https://docs.rs/syn/1.0/syn/struct.DeriveInput.html

很快我们将定义 `impl_hello_macro` 函数，我们将在其中构建我们想要包含的新 Rust 代码。但在我们这样做之前，请注意我们的派生宏的输出也是一个 `TokenStream`。返回的 `TokenStream` 被添加到我们的 crate 用户编写的代码中，因此当他们编译他们的 crate 时，他们将获得我们在修改后的 `TokenStream` 中提供的额外功能。

你可能已经注意到，如果在这里调用 `syn::parse` 函数失败，我们调用 `unwrap` 会导致 `hello_macro_derive` 函数恐慌。我们的过程宏有必要对错误进行恐慌，因为 proc_macro_derive 函数必须返回 TokenStream 而不是 Result 以符合过程宏 API。我们使用 `unwrap` 简化了这个例子；在生产代码中，您应该通过使用 `panic!` 或 `expect` 提供更具体的错误消息来说明问题所在。

现在我们已经有了将带注释的 Rust 代码从 `TokenStream` 转换为 `DeriveInput` 实例的代码，让我们生成在带注释的类型上实现 `HelloMacro` trait 的代码，如示例 19-33 所示。 


<span class="filename">Filename: hello_macro_derive/src/lib.rs</span>

```rust,ignore
fn impl_hello_macro(ast: &syn::DeriveInput) -> TokenStream {
    let name = &ast.ident;
    let gen = quote! {
        impl HelloMacro for #name {
            fn hello_macro() {
                println!("Hello, Macro! My name is {}!", stringify!(#name));
            }
        }
    };
    gen.into()
}
```

<span class="caption">
示例 19-33：实现 HelloMacro特征使用 解析后的 Rust 代码
</span>


我们使用 `ast.ident` 获得一个 `Ident` 结构实例，其中包含带注释类型的名称（标识符）。示例 19-32 中的结构表明，当我们对示例 19-30 中的代码运行 `impl_hello_macro` 函数时，我们得到的 `ident` 将具有 `ident` 字段，其值为 `"Pancakes"`。因此，示例 19-33 中的 `name` 变量将包含一个 `Ident` 结构实例，打印时将是字符串 `"Pancakes"`，即示例 19-30 中结构的名称。

`quote!` 宏让我们定义我们想要返回的 Rust 代码。编译器期望与 `quote!` 宏执行的直接结果不同，因此我们需要将其转换为 `TokenStream`。我们通过调用 `into` 方法来做到这一点，该方法使用此中间表示并返回所需的 `TokenStream` 类型的值。

`quote!` 宏还提供了一些非常酷的模板机制：我们可以输入`#name`，`quote!` 将用变量`name` 中的值替换它。你甚至可以做一些类似于常规宏工作方式的重复。查看 [`quote` crate's docs][quote-docs] 以获得详尽的介绍。

[quote-docs]：https://docs.rs/quote

我们希望我们的过程宏为用户注释的类型生成我们的`HelloMacro`特征的实现，我们可以通过使用`#name`来获得它。 trait 实现有一个函数，`hello_macro`，它的主体包含我们想要提供的功能：打印 `Hello, Macro!我的名字是`，然后是注释类型的名称。

此处使用的 `stringify!` 宏内置于 Rust。它采用 Rust 表达式，例如 `1 + 2`，并在编译时将表达式转换为字符串文字，例如 `"1 + 2"`。这与 `format!` 或 `println!` 不同，宏会评估表达式，然后将结果转换为 `String`。 `#name` 输入有可能是要按字面意思打印的表达式，因此我们使用 `stringify!`。使用 `stringify!` 还可以通过在编译时将 `#name` 转换为字符串文字来节省分配。

此时，`cargo build` 应该在 `hello_macro` 和 `hello_macro_derive` 中成功完成。 让我们将这些 crate 连接到示例 19-30 中的代码，以查看过程宏的运行情况！ 使用 `cargo new pancakes` 在你的 *projects* 目录中创建一个新的二进制项目。 我们需要在 `pancakes` crate 的 *Cargo.toml* 中添加 `hello_macro` 和 hello_macro_derive` 作为依赖项。 如果您将 `hello_macro` 和 `hello_macro_derive` 的版本发布到 [crates.io](https://crates.io/)，它们将是常规依赖项； 如果没有，您可以将它们指定为 `path` 依赖项，如下所示： 

```toml
hello_macro = { path = "../hello_macro" }
hello_macro_derive = { path = "../hello_macro/hello_macro_derive" }
```

将示例 19-30 中的代码放入 *src/main.rs* 中，然后运行 `cargo run`：
应该打印'你好，宏！ 我的名字是 Pancakes！` 过程宏中的 `HelloMacro` 特征的实现被包含在内，而 `pancakes` 板条箱不需要实现它； `#[derive(HelloMacro)]` 添加了 trait 实现。

接下来，让我们探讨其他类型的*过程宏*与*自定义派生宏*有何不同。 

### 类属性宏

类属性宏类似于自定义派生宏，但它们不是为`派生`属性*生成代码*，而是允许您*创建新属性*。 它们也更灵活：`derive` 仅适用于结构和枚举； 属性也可以应用于其他项目，例如函数。

下面是一个使用类属性宏的示例：假设您有一个名为 `route` 的属性，它在使用 Web 应用程序框架时对函数进行注释：

```rust,ignore
#[route(GET, "/")]
fn index() {
```

这个 `#[route]` 属性将由框架定义为过程宏。 宏定义函数的签名如下所示： 

```rust,ignore
#[proc_macro_attribute]
pub fn route(attr: TokenStream, item: TokenStream) -> TokenStream {
```

在这里，我们有两个 `TokenStream` 类型的参数。 第一个是属性的内容：`GET, "/"` 部分。 第二个是属性附加到的项目的主体：在这种情况下，`fn index() {}` 和函数主体的其余部分。

除此之外，类似属性的宏与自定义派生宏的工作方式相同：您创建一个具有 `proc-macro` crate 类型的 crate 并实现一个生成所需代码的函数！ 

### 类函数宏

类函数宏定义看起来像函数调用的宏。 类似于 `macro_rules!` 宏，它们比函数更灵活； 例如，它们可以采用未知数量的参数。 

然而，`macro_rules!` 宏只能使用我们在 [“用于通用元编程的带有 `macro_rules!` 的声明性宏”][decl]<!-- ignore --> 部分中讨论的类似匹配的语法来定义。 

类似函数的宏带有一个 `TokenStream` 参数，并且它们的定义使用 Rust 代码来操纵 `TokenStream`，就像其他两种类型的过程宏一样。 类似函数的宏的一个例子是一个 `sql!` 宏，它可以这样调用： 

[decl]: #declarative-macros-with-macro_rules-for-general-metaprogramming


```rust,ignore
let sql = sql!(SELECT * FROM posts WHERE id=1);
```

这个宏会解析其中的 SQL 语句并检查它的语法是否正确，这比“macro_rules!”宏所能做的要复杂得多。 `sql!` 宏的定义如下： 

```rust,ignore
#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {
```
这个定义类似于自定义派生宏的签名：我们接收括号内的标记并返回我们想要生成的代码。 

## 概括

哇！ 现在，您的工具箱中有一些不会经常使用的 Rust 功能，但您会知道它们在非常特殊的情况下可用。 我们介绍了几个复杂的主题，以便当您在错误消息建议或其他人的代码中遇到它们时，您将能够识别这些概念和语法。 使用本章作为参考来指导您解决问题。

接下来，我们将把我们在整本书中讨论过的所有内容付诸实践，并再做一个项目！