## MySQL之join分析

https://www.cnblogs.com/BeginMan/p/3754322.html



SELECT A.*,B.channel_name FROM `sms_task` A LEFT JOIN sys_channel B FORCE INDEX (PRIMARY) ON A.channel=B.id; 

**FORCE INDEX (PRIMARY)**强制走索引；



### 语法

join 用于多表中字段之间的联系，语法如下：

... FROM table1 INNER|LEFT|RIGHT JOIN table2 ON conditiona

table1:左表；table2:右表。

JOIN 按照功能大致分为如下三类：

INNER JOIN（内连接,或等值连接）：取得两个表中存在连接匹配关系的记录。

LEFT JOIN（左连接）：取得左表（table1）完全记录，即是右表（table2）并无对应匹配记录。

RIGHT JOIN（右连接）：与 LEFT JOIN 相反，取得右表（table2）完全记录，即是左表（table1）并无匹配对应记录。

注意：**mysql不支持Full join**,不过可以通过UNION 关键字来合并 LEFT JOIN 与 RIGHT JOIN来模拟FULL join.





