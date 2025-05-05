本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 背景

众所周知，在rust开发时难免会碰到底层方法传递过来的错误，这种错误很容易造成进程panic，同时进行捕获的时候容易把代码写得不怎么好看。

Rust作为一门注重安全与性能的系统级编程语言，其错误处理机制以简洁和高效著称。与其他语言依赖异常机制不同，Rust通过`Result<T, E>`枚举类型和`?`运算符的组合，提供了一种显式且类型安全的错误处理方式。

Rust的错误处理机制不仅保障了内存安全，还通过类型系统和简洁的语法，使开发者能够编写出既可靠又易维护的代码。掌握这些惯用模式后，你将发现Rust代码在表达力与健壮性之间的完美平衡。正如社区所言：“让所有的`Result`皆为`Ok`”——但在错误不可避免时，Rust确保你能以最优雅的方式应对。

对此推荐一些比较厉害的库使用。



### 原始底层error+?示例

原生Rust的`Result<T, E>`枚举是错误处理的基石，其定义如下：

```rust
#[doc(search_unbox)]
#[derive(Copy, PartialEq, PartialOrd, Eq, Ord, Debug, Hash)]
#[must_use = "this `Result` may be an `Err` variant, which should be handled"]
#[rustc_diagnostic_item = "Result"]
#[stable(feature = "rust1", since = "1.0.0")]
pub enum Result<T, E> {
    /// Contains the success value
    #[lang = "Ok"]
    #[stable(feature = "rust1", since = "1.0.0")]
    Ok(#[stable(feature = "rust1", since = "1.0.0")] T),

    /// Contains the error value
    #[lang = "Err"]
    #[stable(feature = "rust1", since = "1.0.0")]
    Err(#[stable(feature = "rust1", since = "1.0.0")] E),
}
```

当一个函数可能失败时，应返回`Result`类型。例如，读取文件内容的函数可以这样实现：

```rust
fn read_number(path: &str) -> Result<i32, Error> {
        let content = std::fs::read_to_string(path)?;
        let number = content.trim().parse::<i32>().unwrap();
        Ok(number)
    }
```

此处使用的`?`运算符是Rust错误传播的“灵魂”。它的作用简明扼要：若表达式返回`Ok`，则解包取值；若为`Err`，则立即从当前函数返回该错误。这种设计消除了繁琐的`match`语句，使代码保持线性可读性。

`?` 依赖 `From` trait 实现错误类型的自动转换。

使用?能减少一部分的match操作，比如之前的match示例：

```
let contents = match std::fs::read_to_string(path) {
    Ok(c) => c,
    Err(e) => return Err(e),
};
```

**使用`?`的惯用写法**：

```
fn process_file(path: &str) {
        match read_number(path) {
            Ok(n) => println!("数值: {n}"),
            Err(e) => println!("处理文件{path}失败: {e}"),
        }
    }
```

显然，`?`显著简化了代码，尤其在涉及多个可能失败的操作时，其优势更为明显。



何时避免使用`?`：灵活处理错误场景：

尽管`?`在错误传播中极为高效，但某些场景需手动处理错误：

1. 错误恢复与日志记录

```
fn process_file(path: &str) {
    match read_number(path) {
        Ok(n) => println!("数值: {n}"),
        Err(e) => println!("处理文件{path}失败: {e}"),
    }
}
```

在此例中，错误被捕获并记录，而非直接传播，适用于需继续执行后续逻辑的场景。

2. 错误组合与重试

```
fn retry_operation() -> Result<()> {
    let mut last_error = None;
    for _ in 0..3 {
        match perform_operation() {
            Ok(()) => return Ok(()),
            Err(e) => last_error = Some(e),
        }
    }
    Err(last_error.unwrap())
}
```

通过循环和显式错误处理，可实现重试机制，提升系统容错性。



### 自定义错误类型：`thiserror`库

在开发可复用的库时，定义明确的错误类型至关重要。`thiserror`库通过派生宏简化了这一过程。以下是一个自定义错误类型的示例：

**Cargo.toml依赖配置**：

```
[dependencies]
thiserror = "1.0.69"
```

**错误类型定义**：

```
use thiserror::Error;

#[derive(Error, Debug)]
enum MyError {
    #[error("文件读取失败: {0}")]
    Io(#[from] std::io::Error),
    #[error("数值解析错误: {0}")]
    ParseInt(#[from] std::num::ParseIntError),
}
```

应用示例：

```
fn read_number(path: &str) -> Result<i32, MyError> {
    let content = std::fs::read_to_string(path)?;
    let number = content.trim().parse::<i32>()?;
    Ok(number)
}
```

通过`#[from]`属性，Rust自动将底层错误（如`std::io::Error`）转换为自定义错误类型，与`?`无缝结合。

使用：

```
		#[test]
    fn test_thiserror(){
        println!("{:?}",read_number(""));
        let ret = read_number("lorem_ipsum.txt");
        println!("{:?}",ret);
        println!("this error test");
    }
```



### 快速开发：`anyhow`

对于应用程序或命令行工具，`anyhow`库提供了一种轻量级的错误处理方案。它允许开发者跳过自定义错误类型的定义，直接返回`anyhow::Result<T>`。

**Cargo.toml依赖配置**：

```
[dependencies]
anyhow = "1.0"
```

**示例代码**：

```
use anyhow::{Context, Result};

fn read_number(path: &str) -> Result<i32> {
    let content = std::fs::read_to_string(path)
        .context("文件读取失败")?;
    let number = content.trim().parse::<i32>()
        .context("数值解析失败")?;
    Ok(number)
}
```

通过`.context()`方法，可为错误添加描述性信息，便于调试时定位问题根源。

使用：

```
#[cfg(test)]
mod tests {
    use anyhow::{Context, Result};

    fn read_number(path: &str) -> Result<i32> {
        let content = std::fs::read_to_string(path).context("文件读取失败")?;
        let number = content.trim().parse::<i32>().context("数值解析失败")?;
        Ok(number)
    }

    fn process_file(path: &str) {
        match read_number(path) {
            Ok(n) => println!("数值: {n}"),
            Err(e) => println!("处理文件{path}失败: {e}"),
        }
    }

    #[test]
    fn test_anyhow() {
        println!("{:?}",read_number(""));
        let ret = read_number("lorem_ipsum.txt");
        println!("{:?}",ret);
        println!("any how test")
    }
}
```



### 总结

1. **优先使用`?`**：在多数场景下，`?`是错误传播的最简洁方案。
2.  **库开发选择`thiserror`**：定义结构化的错误类型，增强API的清晰度。
3.  **应用开发选择`anyhow`**：快速实现错误处理，减少样板代码。
4.  **谨慎使用`unwrap`**：仅在测试或原型代码中临时使用，避免生产环境崩溃风险。
5.  **合理添加错误上下文**：通过`.context()`或自定义错误消息，提升可调试性。
