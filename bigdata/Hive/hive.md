### 一、常用命令

#### 1.启动beeline客户端

~~~ 
beeline -u jdbc:hive2://spark01:10000 -n hx
~~~

#### 2.常用的交互命令

~~~
bin/hive -help
bin/hive 
-e 不进入hive的交互窗口执行sql语句;
-f 执行脚本中sql语句
~~~

#### 3.参数配置方式

* 默认配置文件 hive-default.xml
* 用户自定义 hive-site.xml
* 命令行bin/hive -hiveconf.param=value
* 参数说明的方式 set mapred.reduce.tasks=100; 查看所有配置：set;

### 二、知识点

#### 1.复杂数据类型

* MAP
* ARRAY
* STRUCT

**类型强转**：CAST(  '1' AS INT  )

#### 2.操作

**建表语句**

~~~sql
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name 
[(col_name data_type [COMMENT col_comment], ...)] 
[COMMENT table_comment] 
[PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)] 
[CLUSTERED BY (col_name, col_name, ...) 
[SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS] 

[ROW FORMAT row_format]DELIMITED [FIELDS TERMINATED BY char] [COLLECTION ITEMS TERMINATED BY char][MAP KEYS TERMINATED BY char] [LINES TERMINATED BY char] 

[STORED AS file_format] 
[LOCATION hdfs_path]
[TBLPROPERTIES (property_name=property_value, ...)]
[AS select_statement]
~~~

**向表中加载数据**

~~~sql
 #1.load
 load data [local] inpath '/opt/module/datas/student.txt' [overwrite] into table student [partition (partcol1=val1,…)];
 #2.Insert
insert into|overwrite table  student_par partition(month='201709') values(1,'wangwu'),(2,'zhaoliu');
insert into|overwrite table  student_par partition(month='201709') select ....
#3.查询语句中创建表并加载数据
create table if not exists student3 as select id, name from student;
#4.创建表时通过Location指定加载数据路径
dfs -put /opt/module/datas/student.txt /student;
create external table if not exists student5(....)...
location '/student;
#5.Import数据到指定Hive表中,先用export导出后，再将数据导入。
import table student2 partition(month='201709') from '/user/hive/warehouse/export/student';
~~~

**导出**

~~~sql
#1.insert
overwrite [local] directory '/opt/module/datas/export/student1' ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' select * from student;
#2.hive shell
bin/hive -e 'select * from default.student;' > /opt/module/datas/export/student4.txt;
#3.Hadoop命令导出到本地
dfs -get /user/hive/warehouse/student/month=201709/000000_0 /opt/module/datas/export/student3.txt;
#4.export     export和import主要用于两个Hadoop平台集群之间Hive表迁移。
export table default.student to '/user/hive/warehouse/export/student';
#5.sqoop
~~~

**排序**

* 全局排序 order by
* 每个reducer内部排序 sort by
* 分区排序 distribute by ,定义数据去哪个reducer，结合sort by使用
* Cluster By 当distribute by和sorts by字段相同时，可以使用cluster by方式

 **常用函数**

~~~sql
#1.空字段赋值
NVL( value，default_value)
#2.CASE WHEN
case sex when '男' then 1 else 0 end
#3.行转列
CONCAT(string A/col, string B/col…)
CONCAT_WS(separator, str1, str2,...)
COLLECT_SET(col)
#4.列转行
LATERAL VIEW EXPLODE(col) tmp as category_name
#5.窗口函数


~~~

