文章来源：https://github.com/Zeb-D/my-review ，请star 强力支持。

------



按照不同的分类方式，可以将把流分为不同的类型。常用的分类有三种：

- 按流的流向划分
- 按操作单元划分
- 按流的角色划分

#### 按流的流向划分，可以分为输入流和输出流

**输入流**：将数据从外设或外村（如键盘、鼠标、文件等）传递到应用程序的称为输入流（Input Stream）；

**输出流**：将数据从应用程序传递到外设或外存（如屏幕、打印机、文件等）的流（OutputStream）。

#### 按操作单元划分，可以划分为字节流和字符流

在介绍字节流和字符流之前，我们需要知道字节和字符之间的关系：

- 1字符 = 2字节
- 1字节 = 8位
- 一个汉字占两个字节长度（因为汉字博大精深，所以有些汉字也会占到三个字节的长度）

**字节流**：处理字节数据。每次读取（写出）一个字节，当传输的资源文件中有中文时，就会出现乱码；

**字符流**：处理字符数据。每次读取（写出）两个字节时，有中文时使用该留就可以正确传输显示文字。

> 在Java中，字节流主要是由InputStream和OutputStream作为基类，而字符流则主要有Reader和Writer作为基类。

#### 按流的角色，可划分为节点流和处理流

**节点流**，就是可以从/向一个特定的IO设备（如磁盘、网络）读/写数据的流。

**处理流**：对一个已存在的流进行连接和封装，通过所封装的流的功能调用实现数据读写。

#### IO流的特性

Java的IO流涉及到了40多个类，但其实都是从如下4个抽象基类中派生出来的：

- **InputStream/Reader：所有的输入流的基类，前者是字节输入流，后者是字符输入流；**
- **OutputStream/Writer：所有的输出流的基类，前者是字节输出流，后者是字符输出流。**
- **先进先出**，最先写入输出流的数据最先被输入流读取；
- **顺序存取**，数据的获取和发送是沿着数据序列顺序进行，不能随机访问中间的数据。（但RandomAccessFile可以从文件的任意位置进行存取（输入输出）操作）；
- **只读或只写**，每个流只能是输入流或输出流的一种，不能同时具备两个功能。

JavaIO中常用的流的类型有以下这些（**粗体所标出的类代表节点流，必须直接与指定的物理节点关联；斜体标出的类代表抽象类，不能直接创建实例**）：

| **分类**   | **字节输入流**           | **字节输出流**            | **字符输入流**      | **字符输出流**      |
| ---------- | ------------------------ | ------------------------- | ------------------- | ------------------- |
| 抽象基类   | *InputStream*            | *OutputStream*            | *Reader*            | *Writer*            |
| 访问文件   | **FileInputStream**      | **FileOutputStream**      | **FileReader**      | **FileWriter**      |
| 访问数组   | **ByteArrayInputStream** | **ByteArrayOutputStream** | **CharArrayReader** | **CharArrayWriter** |
| 访问管道   | **PipedInputStream**     | **PipedOutputStream**     | **PipedReader**     | **PipedWriter**     |
| 访问字符串 |                          |                           | **StringReader**    | **StringWriter**    |
| 缓冲流     | BufferedInputStream      | BufferedOutputStream      | BufferedReader      | BufferedWriter      |
| 转换流     |                          |                           | InputStreamReader   | OutputStreamWriter  |
| 对象流     | ObjectInputStream        | ObjectOutputStream        |                     |                     |
| 抽象基类   | *FilterInputStream*      | *FileOutputStream*        | *FilterReader*      | *FilterWriter*      |
| 打印流     |                          | PrintStream               |                     | PrintWriter         |
| 推回输入流 | PushbackInputStream      |                           | PushbackReader      |                     |
| 特殊流     | DataInputStream          | DataOutputStream          |                     |                     |