## 记一次Sql注入 解决方案

老大反馈代码里面存在sql注入，这个漏洞会导致系统遭受攻击，定位到对应的代码，如下图所示

![](https://raw.githubusercontent.com/mengck/pic/master/img/8D7C6C95-0D95-43d6-85AE-0128D12FF1BB.png)

like 进行了一个字符串拼接，正常的情况下，前端传一个 cxk 过来，那么执行的sql就是


```
select * from test where name like '%cxk%';
```

好像没有什么问题，但是，如果被攻击了传了个 cxk%'; DELETE FROM test WHERE name like '%cxk

那么 这条sql 将会拼接成


```

select * from test where name like '%cxk%'; DELETE FROM test WHERE name like '%cxk%';

```
执行不会报错，结果是 name  like cxk 的数据全部删除，这个还是比较温柔的sql注入，如果是 drop table ，那不是要原地爆炸?

既然Sql 注入危害这么大，那么怎么防范呢？

采用sql语句预编译和绑定变量，是最简单，也是最有效的方案.

那什么是预编译呢？

like this 

```
 select  * from test where name like ?  args: WrappedArray(JdbcValue(%cxk%))
 
```

先是 select  * from test where name like ? 这样预编译好，然后传进来的数据以参数化的形式执行sql,就可以防止sql 注入。

为什么这样可以防止sql 注入呢？

分别给两种场景测试

1.likeString 

代码如下


```

def likeString(name:String) ={
    dataSource.rows[Test](sql" select  * from test where name like  "+s"'%${name}%'")
  }
  
```

输入 cxk%'; DELETE FROM test WHERE name like '%cxk

打断点可以看到

![](https://raw.githubusercontent.com/mengck/pic/master/img/sql18.png)


将statement 中的值复制出来 到navicat 中，可以看到 


![](https://raw.githubusercontent.com/mengck/pic/master/img/sql16.png)


那么jdbc 执行就会 直接执行，然后把cxk 删了。


2.like

代码如下

```
 def like(name:String) ={
    // cxu
    // %cku%
    dataSource.rows[Test](sql" select  * from test where name like ${name.likeSql}")
  }
  
  
  implicit class StringBuildSqlLikeImplicit(s:String){


    def likeSql: String ={
      s"%${s}%"
    }

    def likeLeftSql: String = {
       s"%${s}"
    }

    def likeRightSql: String ={
      s"${s}%"
    }

  }
  


```


输入 cxk%'; DELETE FROM test WHERE name like '%cxk


打断点可以看到

![](https://raw.githubusercontent.com/mengck/pic/master/img/sql13.png)


![](https://raw.githubusercontent.com/mengck/pic/master/img/sql14.png)




```

可以看到 statement =   select  * from test where name like ** NOT SPECIFIED **

```



```

PreparedStatement.setString(1,'cxk%'; DELETE FROM test WHERE name like '%cxk')

```


之后，

```
statement = 
select  * from test where name like '%cxk\'; DELETE FROM test WHERE name like \'%cxk%'

```




将statement 中的值复制出来 到navicat 中，可以看到 

![](https://raw.githubusercontent.com/mengck/pic/master/img/sql15.png)


string 内部的; % 被格式化， 这样执行的话，内部的sql 就以字符串的形式存在，这样避免了Sql 注入。



那 PreparedStatement 是这么做到的呢， 主要的原因是 PreparedStatement.setString(int parameterIndex, String x)

这里面会对x 进行一个格式化

判断是否需要格式化

![](https://raw.githubusercontent.com/mengck/pic/master/img/20191114195945.png)


贴上源码  


```

    private boolean isEscapeNeededForString(String x, int stringLength) {
        boolean needsHexEscape = false;

        for (int i = 0; i < stringLength; ++i) {
            char c = x.charAt(i);

            switch (c) {
                case 0: /* Must be escaped for 'mysql' */

                    needsHexEscape = true;
                    break;

                case '\n': /* Must be escaped for logs */
                    needsHexEscape = true;

                    break;

                case '\r':
                    needsHexEscape = true;
                    break;

                case '\\':
                    needsHexEscape = true;

                    break;

                case '\'':
                    needsHexEscape = true;

                    break;

                case '"': /* Better safe than sorry */
                    needsHexEscape = true;

                    break;

                case '\032': /* This gives problems on Win32 */
                    needsHexEscape = true;
                    break;
            }

            if (needsHexEscape) {
                break; // no need to scan more
            }
        }
        return needsHexEscape;
    }


```


可以看到 它会对传进来的参数判断，如果含有一些非法字符会判断传过来的值需要格式化， 那它是怎么格式化的呢？  我们看下源码 


```
// setString  里的部分源码

  if (this.isLoadDataQuery || isEscapeNeededForString(x, stringLength)) {
                    needsQuoted = false; // saves an allocation later

                    StringBuilder buf = new StringBuilder((int) (x.length() * 1.1));

                    buf.append('\'');

                    //
                    // Note: buf.append(char) is _faster_ than appending in blocks, because the block append requires a System.arraycopy().... go figure...
                    //

                    for (int i = 0; i < stringLength; ++i) {
                        char c = x.charAt(i);

                        switch (c) {
                            case 0: /* Must be escaped for 'mysql' */
                                buf.append('\\');
                                buf.append('0');

                                break;

                            case '\n': /* Must be escaped for logs */
                                buf.append('\\');
                                buf.append('n');

                                break;

                            case '\r':
                                buf.append('\\');
                                buf.append('r');

                                break;

                            case '\\':
                                buf.append('\\');
                                buf.append('\\');

                                break;

                            case '\'':
                                buf.append('\\');
                                buf.append('\'');

                                break;

                            case '"': /* Better safe than sorry */
                                if (this.usingAnsiMode) {
                                    buf.append('\\');
                                }

                                buf.append('"');

                                break;

                            case '\032': /* This gives problems on Win32 */
                                buf.append('\\');
                                buf.append('Z');

                                break;

                            case '\u00a5':
                            case '\u20a9':
                                // escape characters interpreted as backslash by mysql
                                if (this.charsetEncoder != null) {
                                    CharBuffer cbuf = CharBuffer.allocate(1);
                                    ByteBuffer bbuf = ByteBuffer.allocate(1);
                                    cbuf.put(c);
                                    cbuf.position(0);
                                    this.charsetEncoder.encode(cbuf, bbuf, true);
                                    if (bbuf.get(0) == '\\') {
                                        buf.append('\\');
                                    }
                                }
                                // fall through

                            default:
                                buf.append(c);
                        }
                    }

                    buf.append('\'');

                    parameterAsString = buf.toString();
                }
```


从这里可以看出，会讲' 加一个'\'  那么原传入的Sring 就会被格式化成上文所说。
打断点我们可以看到

![](https://raw.githubusercontent.com/mengck/pic/master/img/sql21.png)


这一块是jdbc PreparedStatement  对SQL注入的防范。

从网上我还看到了一些这样的 


那么，什么是所谓的“precompiled SQL statement”呢？

回答这个问题之前需要先了解下一个SQL文在DB中执行的具体步骤：


```

1.Convert given SQL query into DB format -- 将SQL语句转化为DB形式（语法树结构）
2.Check for syntax -- 检查语法
3.Check for semantics -- 检查语义
4.Prepare execution plan -- 准备执行计划（也是优化的过程，这个步骤比较重要，关系到你SQL文的效率，准备在后续文章介绍）
5.Set the run-time values into the query -- 设置运行时的参数
6.Run the query and fetch the output -- 执行查询并取得结果

```
出自  [是如何防止SQL注入的](https://www.cnblogs.com/roostinghawk/p/9703806.html)

打断点调试的时候看到，PreparedStatement 最终还是会转化成statement 然后执行，
jdbc 这么做应该是做应该是为了 mysql 的缓存机制，我们知道，mysql 进行select 查询的时候，会有一个缓存机制，如果执行语句一致的话，就会拿mysql 的缓存直接获取数据，如果以参数形式传到mysql 的话，这样就没有办法命中缓存了（个人看法，错误请佐证）。


 综上所述，SQL注入，用PreparedStatement 防治是可以防治的，代码中也尽量用 PreparedStatement 这种形式。

 题外话，那么这个是jdbc 的做法，那其他的框架是怎么解决SQL 注入的呢?

#### PHP 防治Sql注入

  1.通过函数去对一些特殊字符进行处理  例如 addslashes($str) ，mysql_escape_string($str)

  2.预编译的做法

####   Node 防治SQL 注入

 1.使用escape()对传入参数进行编码，

 2.使用connection.query()的查询参数占位符：（预编译）

 3.使用escapeId()编码SQL查询标识符：

 4.使用mysql.format()转义参数：

参考文章 [node-mysql中防止SQL注入](https://blog.csdn.net/lin_tuer/article/details/54809330)



@by 蒙初开

 







