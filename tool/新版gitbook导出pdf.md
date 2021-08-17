本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

## 新版gitbook导出pdf

### 背景

最近想把自己写的一个gitbook转成pdf分享出去，突然发现最新的gitbook版本已经不支持导出PDF了。于是在网上找了好久终于被我发现了三个将gitbook转换成pdf的方式，现分享给大家。我使用的是mac系统，如果是其他系统大家可以查找相应的方案。



### gitbook自带的npm模块gitbook

npm gitbook的最新版本是3.2.3，最新更新时间是1年前，官方估计已经放弃这个模块了。不过还好，这个模块还能够使用。 具体步骤如下：

1. 安装npm

   通常来说，安装好nodejs后会自动安装相应的npm。

   ```shell
   brew install nodejs
   ```

2. 安装gitbook

   ```shell
   npm install gitbook -g
   npm install gitbook-cli -g
   ```

3. 安装calibre

   直接到官网下载： https://download.calibre-ebook.com/

   安装好calibre之后，需要将 /Applications/calibre.app/Contents/MacOS/ebook-convert 链接到/usr/local/bin/ebook-convert

   ```shell
   ln -s /Applications/calibre.app/Contents/MacOS/ebook-convert  /usr/local/bin/ebook-convert
   ```

4. 生成PDF

   在所有的一切都准备好之后就可以运行下面的命令生成pdf了。

   ```shell
   gitbook pdf
   ```



### 演示截图

```
➜  me git clone https://github.com/KaiserY/trpl-zh-cn.git
Cloning into 'trpl-zh-cn'...
remote: Enumerating objects: 6782, done.
remote: Counting objects: 100% (364/364), done.
remote: Compressing objects: 100% (206/206), done.
remote: Total 6782 (delta 193), reused 216 (delta 158), pack-reused 6418
Receiving objects: 100% (6782/6782), 7.98 MiB | 1.02 MiB/s, done.
Resolving deltas: 100% (4880/4880), done.
```



```
➜  trpl-zh-cn git:(master) gitbook serve
Live reload server started on port: 35729
Press CTRL+C to quit ...
```



```
➜  trpl-zh-cn git:(master) ln -s /Applications/calibre.app/Contents/MacOS/ebook-convert  /usr/local/bin/ebook-convert
```

```
➜  trpl-zh-cn git:(master) gitbook pdf
info: 7 plugins are installed
info: 6 explicitly listed
info: loading plugin "highlight"... OK
info: loading plugin "search"... OK
info: loading plugin "lunr"... OK
info: loading plugin "sharing"... OK
info: loading plugin "fontsettings"... OK
info: loading plugin "theme-default"... OK
info: found 102 pages
info: found 26 asset files

info: >> generation finished with success in 60.8s !
info: >> 1 file(s) generated
```

```
➜  trpl-zh-cn git:(master) ✗ ls
LICENSE             book.json           custom-template.tex src
README.md           book.pdf            ferris.css          vuepress_page.png
_book               book.toml           ferris.js
```

