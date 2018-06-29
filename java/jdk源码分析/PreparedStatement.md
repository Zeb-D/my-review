####Statement与PreparedStatement的区别

通过调用connection.preparedStatement(sql)方法可以获得PreparedStatment对象。数据库系统会对sql语句进行预编译处理（如果JDBC驱动支持的话），预处理语句将被预先编译好，这条预编译的sql查询语句能在将来的查询中重用，这样一来，它比Statement对象生成的查询速度更快。



#### PreparedStatement的优势

1. **PreparedStatement**可以写动态参数化的查询

   用PreparedStatement你可以写带参数的sql查询语句，通过使用相同的sql语句和不同的参数值来做查询比创建一个不同的查询语句要好，下面是一个参数化查询：

   `SELECT interest_rate FROM loan WHERE loan_type=?`

   ​

2. **PreparedStatement**比 **Statement** 更快

   使用 **PreparedStatement** 最重要的一点好处是它拥有更佳的性能优势，SQL语句会预编译在数据库系统中。执行计划同样会被缓存起来，它允许数据库做参数化查询。使用预处理语句比普通的查询更快，因为它做的工作更少（数据库对SQL语句的分析，编译，优化已经在第一次查询前完成了）。为了减少数据库的负载，生产环境中的JDBC代码你应该总是使用**PreparedStatement** 。值得注意的一点是：为了获得性能上的优势，应该使用参数化sql查询而不是字符串追加的方式。

   ​

3. **PreparedStatement**可以防止SQL注入式攻击

   在SQL注入攻击里，恶意用户通过SQL元数据绑定输入

   在使用参数化查询的情况下，数据库系统（eg:MySQL）不会将参数的内容视为SQL指令的一部分来处理，而是在数据库完成SQL指令的编译后，才套用参数运行，因此就算参数中含有破坏性的指令，也不会被数据库所运行。

   ​

4. 比起凌乱的字符串追加似的查询，PreparedStatement查询可读性更好、更安全。



#### PreparedStatement的局限性

1. 为了防止SQL注入攻击，PreparedStatement不允许一个占位符（？）有多个值，如**IN**
2. 不支持预编译SQL查询的JDBC驱动，在调用connection.prepareStatement(sql)的时候，它不会把SQL查询语句发送给数据库做预处理，而是等到执行查询动作的时候（调用executeQuery()方法时）才把查询语句发送个数据库，这种情况和使用Statement是一样的。





### 数据结构







### 实现方式





###主要方法