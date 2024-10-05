本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------



高阶Trait边界。它是一个非常重要的特性，理解它有两个原因：

- 首先，你可能会在流行的rust库(如Tokio或Axum)的源代码中遇到它。

- 其次，它将从根本上改变你对泛型、Trait和生命周期在Rust中协同工作的理解，掌握最大灵活性的高级泛型编程技术。

  

高阶Trait边界难以理解的部分原因是该特性的文档非常稀少。所以在这篇文章中，我将解释什么是高阶Trait边界，以及如何在高级泛型代码中使用它们来创建非常灵活的api。

为了理解什么是高阶Trait边界，我们必须简单地介绍一下Trait、Trait边界和生命周期注释。

Trait类似于其他语言中的接口，它允许我们以一种抽象的方式通过函数和方法定义共享行为。另一方面，Trait边界允许我们将Trait和泛型结合起来。

```
fn print_debug<T>(input: T) {
        println!("{:?}", input) // `T` doesn't implement `Debug`
    }
```

这段代码给了我们一个编译时错误，说泛型T没有实现Debug Trait。问题是我们的函数接受任何泛型类型，这些泛型类型有可能可以被debug格式打印，也有可能不可以被打印。解决方案是使用Trait边界来限制可能的具体类型。

```
fn print_debug<T: std::fmt::Debug>(input: T) {
        println!("{:?}", input)
    }
```

这个泛型将只接受那些实现了Debug trait的类型，换句话说，这个泛型被Debug trait约束。

重要的是我们还可以使用where子句或impl语法指定Trait边界：

```
fn print_debug<T>(input: T) where T:std::fmt::Debug {
        println!("{:?}", input) 
    }

    fn print_debug1(input: impl std::fmt::Debug) {
        println!("{:?}", input) 
    }
```

在后面的高阶Trait边界例子中我们会看到高阶Trait边界在Rust中使用了两种不同类型的泛型。到目前为止，我们已经看到了类型泛型，它允许我们对具体类型进行抽象。

Rust还提供了另一种类型的泛型：生命周期注释，这与类型泛型不同，它允许我们编写可以处理多个具体类型的泛型代码。生命周期注释主要用于表达引用的生命周期之间的关系，这有助于编译器确保引用在使用时仍然有效。

假设我们有一个函数，它接受字符串切片作为输入，并返回切片中的第一个单词：

```
fn first_word(s: &str) -> &str{
        let bytes = s.as_bytes();
        for (i, &item) in bytes.iter().enumerate(){
            if item == b' '{
                return &s[..i];
            }
        }
        s
    }
```

编译器实际上将函数签名扩展了一个标记'a，这是一个通用的生命周期注释：

```
fn first_word<'a>(s: &'a str) -> &'a str{
        let bytes = s.as_bytes();
        for (i, &item) in bytes.iter().enumerate(){
            if item == b' '{
                return &s[..i];
            }
        }
        s
    }
```

它创建了输入引用的生命周期和返回引用的生命周期之间的关系，这种关系表明，这个函数返回的引用必须至少在输入引用有效的时候有效，这有助于编译器检查无效的引用。

如果使返回引用的生命周期长于输入引用的生命周期，编译器将抛出错误。

```
 		#[test]
    fn test_first_word() {
        let word;
        {
            let my_string = String::from("abcss");
            word = first_word(&my_string); 
            // `my_string` does not live long enough
            //borrowed value does not live long enough
        }
        println!("First word: {}", word);
    }
```

在这个例子中，my_string将在内部作用域结束时被释放，因此之后使用返回的引用将是无效的。Rust编译器可以防止这种内存安全错误，幸运的是，在大多数情况下，我们甚至不需要显式地编写泛型生命周期注释，因为rust编译器足够聪明，可以推断出引用的生命周期。

现在我们已经理解了Trait边界和泛型生命周期注释，让我们讨论一下更高级别的Trait边界。到目前为止，我们已经分别使用了Trait边界和生命周期泛型，高阶Trait边界把这些概念结合起来，它允许我们指定一个Trait边界在所有可能的生命周期内都成立，这在表达复杂的生命周期关系时非常有用。

```
	trait Formatter {
        fn format<T: Display>(&self,value:T) -> String;
    }
```

这里我们有一个名为Formatter的Trait，它定义了一个函数，该函数接受任何实现Display Trait的类型，并返回一个格式化的字符串。

```
		struct  SimpleFormatter;

    impl Formatter for SimpleFormatter {
        fn format<T: Display>(&self,value:T) -> String{
            format!("Value: {}",value)
        }
    }

    fn  apply_format<F>(formatter:F) -> impl Fn(&str) ->String 
    where F: Formatter,
    {
       move |s| formatter.format(s) 
    }
```

我们在SimpleFormatter结构体上实现这个特性，然后创建一个函数，该函数接受formatter并返回一个闭包，该闭包使用该formatter格式化给定的字符串。注意，为了返回闭包，我们使用了特殊的Fn trait来定义闭包签名。

```
 #[test]
    fn test_formatter() {
        let formatter = SimpleFormatter;
        let format_fn = apply_format(formatter);

        let s1 = "Hello";
        let s2 = String::from("Word");

        println!("{}", format_fn(s1));
        // println!("{}", format_fn(s2)); // consider borrowing here: `&s2`

    }
```

在test_formatter中，我们创建这个闭包，并使用字符串切片和堆分配的字符串调用它。如果我们回头看看apply_format函数，会注意到返回闭包接受一个引用。

编译器会推断出这个引用的生命周期，但是如果我们要显式地编写它会是什么样子呢？可以直观地这样写：在函数apply_format上定义泛型生命周期，并将泛型生命周期分配给引用：

```
fn apply_format<'a, F>(formatter: F) -> impl Fn(&'a str) -> String
    where
        F: Formatter,
    {
        move |s| formatter.format(s)
    }
```

然而，这段代码给了我们一个编译时错误，说借用的值可能存在的时间不够长

```
#[test]
    fn test_formatter() {
        let formatter = SimpleFormatter;
        let format_fn = apply_format(formatter);

        let s1 = "Hello";
        let s2 = String::from("Word");

        println!("{}", format_fn(s1));
        // println!("{}", format_fn(&s2)); 
        // `s2` does not live long enough
        // values in a scope are dropped in the opposite order they are defined
    }
```

要理解这个错误，我们需要分解apply_format的函数签名，'a被声明为函数的通用生命周期参数，但它没有在输入参数中使用而是只出现在返回类型中。那么'a在这里创建了什么关系呢？

在这种情况下，'a在返回闭包接受的输入引用的生命周期和闭包本身的生命周期之间建立了关系。这种关系确保了通过apply_format返回的闭包只能被至少与闭包本身一样长的引用调用。

回到test_formatter，我们可以看到闭包被存储在format_fn变量中，该变量一直存在到main作用域结束。在Rust中，按定义的相反顺序删除变量，这意味着字符串s2将在format_fn中存储的闭包之前被删除。

一旦s2被删除，所有对s2的引用都将失效，因为这些引用指向的内存已释放。所以所有指向字符串s2的引用的生命周期将在s2被删除时结束，问题是，存储在format函数中的闭包期望传递给它的所有引用与闭包本身一样长。

解决此错误的一种方法是在创建s2之后定义format_fn变量，以便在s2之前删除format_fn变量。

```
		#[test]
    fn test_formatter() {
        let formatter = SimpleFormatter;

        let s1 = "Hello";
        let s2 = String::from("Word");

        let format_fn = apply_format(formatter);


        println!("{}", format_fn(s1));
        println!("{}", format_fn(&s2));
    }
```

但是，这并不能解决该闭包的所有问题。例如，如果我们创建了一个内部作用域，并调用闭包时引用了该内部作用域中定义的字符串。

```
		#[test]
    fn test_formatter() {
        let formatter = SimpleFormatter;

        let s1 = "Hello";
        let s2 = String::from("Word");

        let format_fn = apply_format(formatter);

        println!("{}", format_fn(s1));
        {
            let s3 = String::from("Abc");
            println!("{}", format_fn(&s3));
        }
        println!("{}", format_fn(&s2));
    }
```

我们能正常边缘且运行。



现在这段代码在技术上是内存安全的，但是它不完全满足我们代码最初设置的生命周期限制，因此，编译时错误的核心问题是我们最初设置的生命周期约束过于严格。原因在于我们定义泛型生命周期注释的方式，传递给闭包的引用的生命周期与闭包的生命周期绑定在一起，我们真正想要的是闭包能够处理任何生命周期的引用。

我们可以通过高阶Trait边界来实现这一点，而不是在函数上定义生命周期泛型，

```
fn apply_format<F>(formatter: F) -> impl for<'a> Fn(&'a str) -> String
    where
        F: Formatter,
    {
        move |s| formatter.format(s)
    }
```

我们使用了for<'a>语法，这意味着返回的闭包可以使用任何生命周期的字符串切片，只要该生命周期至少覆盖闭包执行的持续时间，这个简单的修改使我们的代码可以编译。

就像生命周期泛型一样，在许多情况下，没有必要显式地编写高阶Trait边界，因为编译器足够聪明，可以推断出它们，但在某些情况下，还是需要显式地将它们写出来，这就是为什么在查看处理复杂泛型抽象的rust crate时可能会遇到它们的原因。

现在，当你遇到这种语法时，你就会知道它的意思，以及为什么要使用它。