本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 背景

目前cargo在低版本会把全量依赖树下载下来，但在1.68版本以上提供sparse+registry 来解决这个问题，

原理是根据你的代码使用到依赖树进行下载，你可以理解就是编译时的依赖树，这样在增量依赖下载是非常快的，

特别是对一些使用docker容器进行CI打包是非常快的；



### 操作

找到国内／内网的代理，

然后在～./cargo/config 中进行编辑

```
[source.crates-io]
replace-with = 'tuna'

[source.tuna]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"
[source.tuna-sparse]
registry = "sparse+https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"

```

配置比较简单，就一个基于source.xxx加了对应的source.xxx-sparse 以及它的链接前缀也加了。



### 测试

基于这边的https://tokio.rs/tokio/tutorial/hello-tokio

写个minigrep demo来测试；

每次测试前需要把 ~/.cargo/registry的目录删除，防止有缓存；

```
加了sparse的配置的测试
➜  minigrep git:(main) ✗ time cargo build
    Updating tuna index
warning: spurious network error (2 tries remaining): [28] Timeout was reached (download of async-stream v0.3.5 failed to transfer more than 10 bytes in 30s)
warning: spurious network error (2 tries remaining): [28] Timeout was reached (failed to download any data for ansi_term v0.12.1 within 30s)
  Downloaded async-stream-impl v0.3.5 (registry tuna)
  Downloaded atty v0.2.14 (registry tuna)
  Downloaded autocfg v1.1.0 (registry tuna)
  Downloaded bitflags v1.3.2 (registry tuna)
  Downloaded bytes v1.4.0 (registry tuna)
  Downloaded cfg-if v1.0.0 (registry tuna)
  Downloaded chrono v0.4.24 (registry tuna)
  Downloaded clap v2.34.0 (registry tuna)
  Downloaded core-foundation-sys v0.8.4 (registry tuna)
  Downloaded futures-core v0.3.28 (registry tuna)
  Downloaded heck v0.3.3 (registry tuna)
  Downloaded iana-time-zone v0.1.56 (registry tuna)
  Downloaded itoa v1.0.6 (registry tuna)
  Downloaded lazy_static v1.4.0 (registry tuna)
  Downloaded libc v0.2.142 (registry tuna)
  Downloaded lock_api v0.4.9 (registry tuna)
  Downloaded log v0.4.17 (registry tuna)
  Downloaded matchers v0.0.1 (registry tuna)
  Downloaded mini-redis v0.4.1 (registry tuna)
  Downloaded mio v0.8.6 (registry tuna)
  Downloaded num-integer v0.1.45 (registry tuna)
  Downloaded num-traits v0.2.15 (registry tuna)
  Downloaded num_cpus v1.15.0 (registry tuna)
  Downloaded once_cell v1.17.1 (registry tuna)
  Downloaded parking_lot v0.12.1 (registry tuna)
  Downloaded parking_lot_core v0.9.7 (registry tuna)
  Downloaded pin-project v1.0.12 (registry tuna)
  Downloaded pin-project-internal v1.0.12 (registry tuna)
  Downloaded pin-project-lite v0.2.9 (registry tuna)
  Downloaded proc-macro-error v1.0.4 (registry tuna)
  Downloaded proc-macro-error-attr v1.0.4 (registry tuna)
  Downloaded proc-macro2 v1.0.56 (registry tuna)
  Downloaded quote v1.0.26 (registry tuna)
  Downloaded regex v1.8.1 (registry tuna)
  Downloaded regex-automata v0.1.10 (registry tuna)
  Downloaded regex-syntax v0.6.29 (registry tuna)
  Downloaded regex-syntax v0.7.1 (registry tuna)
  Downloaded ryu v1.0.13 (registry tuna)
  Downloaded serde v1.0.162 (registry tuna)
  Downloaded serde_json v1.0.96 (registry tuna)
  Downloaded sharded-slab v0.1.4 (registry tuna)
  Downloaded signal-hook-registry v1.4.1 (registry tuna)
  Downloaded smallvec v1.10.0 (registry tuna)
  Downloaded socket2 v0.4.9 (registry tuna)
  Downloaded strsim v0.8.0 (registry tuna)
  Downloaded structopt v0.3.26 (registry tuna)
  Downloaded structopt-derive v0.4.18 (registry tuna)
  Downloaded syn v1.0.109 (registry tuna)
  Downloaded syn v2.0.15 (registry tuna)
  Downloaded textwrap v0.11.0 (registry tuna)
  Downloaded thread_local v1.1.7 (registry tuna)
  Downloaded tokio v1.28.0 (registry tuna)
  Downloaded tokio-macros v2.1.0 (registry tuna)
  Downloaded tokio-stream v0.1.14 (registry tuna)
  Downloaded tracing v0.1.37 (registry tuna)
  Downloaded tracing-attributes v0.1.24 (registry tuna)
  Downloaded tracing-core v0.1.30 (registry tuna)
  Downloaded tracing-futures v0.2.5 (registry tuna)
  Downloaded tracing-log v0.1.3 (registry tuna)
  Downloaded tracing-serde v0.1.3 (registry tuna)
  Downloaded tracing-subscriber v0.2.25 (registry tuna)
  Downloaded unicode-ident v1.0.8 (registry tuna)
  Downloaded unicode-segmentation v1.10.1 (registry tuna)
  Downloaded unicode-width v0.1.10 (registry tuna)
  Downloaded vec_map v0.8.2 (registry tuna)
  Downloaded version_check v0.9.4 (registry tuna)
  Downloaded async-stream v0.3.5 (registry tuna)
  Downloaded ansi_term v0.12.1 (registry tuna)
warning: spurious network error (2 tries remaining): [28] Timeout was reached (download of scopeguard v1.1.0 failed to transfer more than 10 bytes in 30s)
warning: spurious network error (2 tries remaining): [28] Timeout was reached (failed to download any data for atoi v0.3.3 within 30s)
  Downloaded atoi v0.3.3 (registry tuna)
  Downloaded scopeguard v1.1.0 (registry tuna)
  Downloaded 70 crates (5.2 MB) in 1m 17s
    Finished dev [unoptimized + debuginfo] target(s) in 3m 09s
cargo build  73.47s user 18.02s system 48% cpu 3:10.06 total

```

```
//无sparse的测试数据
minigrep git:(main) ✗ time cargo build
    Updating tuna index
  Downloaded async-stream v0.3.5 (registry tuna)
  Downloaded atoi v0.3.3 (registry tuna)
  Downloaded atty v0.2.14 (registry tuna)
  Downloaded autocfg v1.1.0 (registry tuna)
  Downloaded bitflags v1.3.2 (registry tuna)
  Downloaded async-stream-impl v0.3.5 (registry tuna)
  Downloaded cfg-if v1.0.0 (registry tuna)
  Downloaded bytes v1.4.0 (registry tuna)
  Downloaded chrono v0.4.24 (registry tuna)
  Downloaded core-foundation-sys v0.8.4 (registry tuna)
  Downloaded futures-core v0.3.28 (registry tuna)
  Downloaded heck v0.3.3 (registry tuna)
  Downloaded iana-time-zone v0.1.56 (registry tuna)
  Downloaded itoa v1.0.6 (registry tuna)
  Downloaded lazy_static v1.4.0 (registry tuna)
  Downloaded clap v2.34.0 (registry tuna)
  Downloaded lock_api v0.4.9 (registry tuna)
  Downloaded log v0.4.17 (registry tuna)
  Downloaded matchers v0.0.1 (registry tuna)
  Downloaded mini-redis v0.4.1 (registry tuna)
  Downloaded mio v0.8.6 (registry tuna)
  Downloaded num-integer v0.1.45 (registry tuna)
  Downloaded num-traits v0.2.15 (registry tuna)
  Downloaded num_cpus v1.15.0 (registry tuna)
  Downloaded once_cell v1.17.1 (registry tuna)
  Downloaded parking_lot v0.12.1 (registry tuna)
  Downloaded parking_lot_core v0.9.7 (registry tuna)
  Downloaded pin-project v1.0.12 (registry tuna)
  Downloaded pin-project-internal v1.0.12 (registry tuna)
  Downloaded pin-project-lite v0.2.9 (registry tuna)
  Downloaded proc-macro-error v1.0.4 (registry tuna)
  Downloaded proc-macro-error-attr v1.0.4 (registry tuna)
  Downloaded libc v0.2.142 (registry tuna)
  Downloaded proc-macro2 v1.0.56 (registry tuna)
  Downloaded quote v1.0.26 (registry tuna)
  Downloaded regex v1.8.1 (registry tuna)
  Downloaded regex-automata v0.1.10 (registry tuna)
  Downloaded regex-syntax v0.6.29 (registry tuna)
  Downloaded ryu v1.0.13 (registry tuna)
  Downloaded scopeguard v1.1.0 (registry tuna)
  Downloaded serde v1.0.162 (registry tuna)
  Downloaded serde_json v1.0.96 (registry tuna)
  Downloaded sharded-slab v0.1.4 (registry tuna)
  Downloaded signal-hook-registry v1.4.1 (registry tuna)
  Downloaded smallvec v1.10.0 (registry tuna)
  Downloaded regex-syntax v0.7.1 (registry tuna)
  Downloaded socket2 v0.4.9 (registry tuna)
  Downloaded strsim v0.8.0 (registry tuna)
  Downloaded structopt v0.3.26 (registry tuna)
  Downloaded structopt-derive v0.4.18 (registry tuna)
  Downloaded syn v1.0.109 (registry tuna)
  Downloaded textwrap v0.11.0 (registry tuna)
  Downloaded thread_local v1.1.7 (registry tuna)
  Downloaded syn v2.0.15 (registry tuna)
  Downloaded tokio-macros v2.1.0 (registry tuna)
  Downloaded tokio-stream v0.1.14 (registry tuna)
  Downloaded tracing v0.1.37 (registry tuna)
  Downloaded tracing-attributes v0.1.24 (registry tuna)
  Downloaded tracing-core v0.1.30 (registry tuna)
  Downloaded tokio v1.28.0 (registry tuna)
  Downloaded tracing-futures v0.2.5 (registry tuna)
  Downloaded tracing-serde v0.1.3 (registry tuna)
  Downloaded tracing-log v0.1.3 (registry tuna)
  Downloaded unicode-ident v1.0.8 (registry tuna)
  Downloaded unicode-segmentation v1.10.1 (registry tuna)
  Downloaded unicode-width v0.1.10 (registry tuna)
  Downloaded vec_map v0.8.2 (registry tuna)
  Downloaded version_check v0.9.4 (registry tuna)
  Downloaded ansi_term v0.12.1 (registry tuna)
  Downloaded tracing-subscriber v0.2.25 (registry tuna)
  Downloaded 70 crates (5.2 MB) in 4m 56s
    Blocking waiting for file lock on package cache
   Compiling minigrep v0.1.0 (/Users/yongdongzou/rust/my-rust/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 6m 50s
cargo build  73.01s user 18.80s system 22% cpu 6:50.50 total
```

可以看出3m 09s 是6m 50s 快了一倍的。