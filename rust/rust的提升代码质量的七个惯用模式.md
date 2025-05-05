本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 背景

Rust以其对内存安全和零成本抽象的设计闻名，但编写符合语言习惯的代码往往是开发者面临的挑战。本文将通过具体场景和完整代码示例，解析七个能显著提升代码质量的惯用模式。

------



### 模式匹配取代显式检查

新手常通过`is_ok()`和`unwrap()`组合处理`Result`类型：

```
if result.is_ok() {
    let val = result.unwrap();
    do_something(val);
}
```

此写法存在潜在风险：`unwrap()`可能导致程序崩溃。更安全的方式是使用模式匹配：

```
if let Ok(val) = result {
    do_something(val);
}
```

或完整处理所有分支：

```
match result {
    Ok(val) => do_something(val),
    Err(e) => eprintln!("Error: {}", e),
}
```

**优势**：

- 避免`unwrap()`引发的运行时崩溃
- 强制处理所有可能的分支（在`match`场景下）
- 代码意图更清晰

------



### 迭代器优先于手动循环

传统循环方式需要显式管理容器：

```
let mut doubled = Vec::new();
for x in &numbers {
    doubled.push(x * 2);
}
```

使用迭代器链可简化代码并提高可读性：

```
let doubled: Vec<_> = numbers.iter().map(|x| x * 2).collect();
```

**优势**：

- 消除临时变量和可变状态
- 支持惰性求值（如`filter`和`take`组合）
- 更易并行化（通过`rayon`等库）

------



### 错误传播的`?`操作符

传统错误处理需要显式`match`分支：

```
let file = match File::open("config.txt") {
    Ok(f) => f,
    Err(e) => return Err(e),
};
```

使用`?`操作符可简化为：

```
let file = File::open("config.txt")?;
```

**扩展应用**：
在自定义错误类型中实现`From`特征后，可实现跨错误类型的自动转换：

```
impl From<io::Error> for MyError {
    fn from(e: io::Error) -> Self {
        MyError::Io(e)
    }
}
```

------



### Cow类型的灵活所有权管理

当函数可能修改输入数据时，直接使用`String`会导致不必要的克隆。通过`Cow`（Clone On Write）优化：

```
use std::borrow::Cow;

fn process(input: Cow<str>) -> Cow<str> {
    if input.len() > 10 {
        input
    } else {
        Cow::Owned(format!("{}:!!", input))
    }
}
```

**使用场景**：

- 输入可能为字面量（`&'static str`）或动态字符串
- 需要避免无谓的堆内存分配
- 函数返回值可能为原始引用或新对象

------



### derive宏消除样板代码

手动实现特征易出错且冗长：

```
impl Debug for MyStruct {
    fn fmt(&self, f: &mut Formatter) -> Result {
        // 手动实现字段输出
    }
}
```

使用`#[derive]`自动生成：

```
#[derive(Debug, Clone, PartialEq)]
struct MyStruct {
    id: u32,
    name: String,
}
```

**支持的特征**：

- 基础特征：`Debug`, `Clone`, `Copy`, `Default`
- 比较特征：`PartialEq`, `Eq`, `PartialOrd`, `Ord`
- 序列化：`Serialize`, `Deserialize`（需`serde`支持）

------



### 线程安全的共享状态管理

跨线程共享可变状态需结合`Arc`和`Mutex`：

```
use std::sync::{Arc, Mutex};

let data = Arc::new(Mutex::new(Vec::new()));
let data_clone = Arc::clone(&data);

thread::spawn(move || {
    let mut vec = data_clone.lock().unwrap();
    vec.push(42);
});
```

**注意要点**：

- `Arc`提供原子引用计数
- `Mutex`确保独占访问
- 优先考虑无锁数据结构（如`RwLock`或通道）

------



### 单一职责的小型函数

庞大函数难以维护和测试：

```
fn handle_request() -> Result<(), Error> {
    // 解析、验证、处理、日志... 全部混在一起
}
```

拆分为独立功能单元：

```
fn parse_input(input: &str) -> Result<Request, Error> { /* ... */ }
fn validate(req: &Request) -> Result<(), Error> { /* ... */ }
fn process(req: Request) -> Result<Response, Error> { /* ... */ }
```

**优势**：

- 每个函数可独立测试
- 错误处理更聚焦
- 组合式开发提升可维护性

------



### 总结

编写符合Rust习惯的代码并非追求奇技淫巧，而是通过语言特性实现以下目标：

1. **安全性**：利用所有权系统和类型检查避免内存错误
2. **清晰性**：通过模式匹配和迭代器链提升可读性
3. **高效性**：零成本抽象保障运行时性能

这些模式不仅使代码更健壮，还能降低后续维护成本。建议在代码审查时重点关注这些实践，逐步培养符合Rust哲学的编码风格。
