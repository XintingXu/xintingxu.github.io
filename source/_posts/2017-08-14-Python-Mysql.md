title: Python中的MySQLdb操作模式性能分析
date: 2017-08-14 20:35:17
tags: [Python,Mysql]
categories: [Blog]
---
# 不同的MySQLdb操作方式性能分析

要在数据库中插入一条数据，该有哪些操作？

1. **连接数据库**
2. **获取数据库游标**
3. **执行SQL语句**
4. **提交**
5. **关闭数据库连接**

然而，在实际的操作中，不可避免涉及到对数据库的多次操作，那么，不同的执行组合到底有什么样的性能表现呢？

在这里，我设定了几种操作方法，并进行了相关的性能测试，用来判断各自的性能差异。

### 测试基准

数据库中创建一个4字段的表，字段属性均为int(32)，对表进行1000000次插入操作。

每次测试完成后，清空表中数据。

采用Python的**time.process_time()**作为性能评价标准

### 一：数据库保持连接，频繁commit

{% codeblock lang=Python %}
def form1():
    db = MySQLdb.connect(db_url,db_user,db_pass,db_name,charset='utf8')
    cursor = db.cursor()
    for i in range (1,1000000):
        sql = "INSERT INTO %s.%s VALUES(%d,%d,%d,%d)" %\
              (db_name,table_name,i,i+1,i-1,i/2)
        cursor.execute(sql)
        db.commit()
    db.close()
{% endcodeblock %}

在该种方法中，数据库的连接处于保持状态，每次执行插入语句，都会进行commit操作提交修改。

### 二：数据库保持连接，最后一次commit

{% codeblock lang=Python %}
def form2():
    db = MySQLdb.connect(db_url,db_user,db_pass,db_name,charset='utf8')
    cursor = db.cursor()
    for i in range (1,1000000):
        sql = "INSERT INTO %s.%s VALUES(%d,%d,%d,%d)" %\
              (db_name,table_name,i,i+1,i-1,i/2)
        cursor.execute(sql)
    db.commit()
    db.close()
{% endcodeblock %}

该种方法需要保持数据库的连接，但并不是每次执行SQL语句都会进行数据库的提交，而是最后一次提交。

### 三：在数据库操作前连接，操作完成后关闭

{% codeblock lang=Python %}
def form3():
    for i in range (1,1000000):
        db = MySQLdb.connect(db_url,db_user,db_pass,db_name,charset='utf8')
        cursor = db.cursor()
        sql = "INSERT INTO %s.%s VALUES(%d,%d,%d,%d)" %\
              (db_name,table_name,i,i+1,i-1,i/2)
        cursor.execute(sql)
        db.commit()
        db.close()
{% endcodeblock %}

该种方法在在每次数据库操作前均需要建立与数据库的连接，但并不会保持下去，而是在对数据库的操作完成后进行连接释放。当下次再进行数据库操作的时候再创建连接。

## 测试结果

```
form1 :  116.677019931
form2 :  82.13445935399999
form3 :  328.88110945000005
```

理论上分析，每次对数据库操作都需要建立连接，一定是时间消耗最大的；而最后一次全部提交，一定是时间消耗最少的。

从最后的时间统计中，也体现了这一规律。form3这种每次都进行连接建立的方式，耗费的时间达到了form2的4倍。

而form1这种每次数据库操作都进行提交操作的方式，比form2增加了42%的时间。

**但是，这三种方法在不同的场景中有不同的应用意义。**

**对于数据库服务器连接数量有限制的场景中，采用form1和form2既有可能导致部分终端无法连接到数据库。**

**对于数据操作敏感的场景，form2这种一次提交的方式极有可能导致失败，需要大范围的数据回滚。**

**所以，各种方法利弊共存，还是要考虑实际情况来操作。**





> 
以上内容均为本人学习过程中的测试和分析，如果有任何问题，欢迎批评指正。