#ADP平台上列存储数据的自动分区

大数据时代我们通常都会把原始数据存储在分布式文件系统上，如果需要处理非常大量的数据
就会遇到扫描数据量太大的问题，那么首先我们会把数据进行分区，在扫描数据时只扫描我们需要
处理的数据，大大缩短处理时间。
ADP支持使用parquet作为列存储方式，可以压缩数据占用空间，做列修剪(column pruning)，即
只读取所需要的列。
另外ADP还支持parquet文件的数据分区，下面用具体的例子解释。

##1.数据的分区写入与发现
比如我们有1到20的数据，下面例子分别写入到'year=2015'和'year=2016'两个两个目录
```scala
val df1= sc.makeRDD(1 to 10).toDF("value")
df1.write.parquet(basePath+"/year=2015")

val df2= sc.makeRDD(11 to 20).toDF("value")
df2.write.parquet(basePath+"/year=2016")

```
这样文件系统上得到如下的数据文件
```console
 basePath
    ├── year=2016
    │      └── data.parquet
    └── year=2015
           └── data.parquet
```
当我们读入数据时，会发现系统自动增加了一个`year`的虚拟列
```console
scala> val df= sqlContext.read.parquet(basePath)
scala> df.printSchema
root
 |-- value: integer (nullable = true)
 |-- year: integer (nullable = true)

scala> df.show
+-----+----+
|value|year|
+-----+----+
|   16|2016|
|   17|2016|
|   18|2016|
|   19|2016|
|   20|2016|
|   11|2016|
|   12|2016|
|   13|2016|
|   14|2016|
|   15|2016|
|    8|2015|
|    9|2015|
|   10|2015|
|    6|2015|
|    7|2015|
|    3|2015|
|    4|2015|
|    5|2015|
|    1|2015|
|    2|2015|
+-----+----+
```
当然如果你的数据中就包含了可以用来分区的列，那么ADP也支持显式调用`partitionBy(key)`
来做数据分区

##2.数据读取与分区的修剪(pruning)
虽然我们的数据中并不含有年份的列，但我们仍然可以使用常规的过滤方法来查找数据，例如
我们只需要查找比如2015年的数据，那么我们可以直接使用年份进行过滤。
```console
scala> df.filter(df("year")===2015).show
+-----+----+
|value|year|
+-----+----+
|    8|2015|
|    9|2015|
|   10|2015|
|    6|2015|
|    7|2015|
|    3|2015|
|    4|2015|
|    5|2015|
|    1|2015|
|    2|2015|
+-----+----+
```
甚至也可以在SQL中使用时间过滤
```console
scala> sqlContext.sql("select value from simpleData where year='2015'").show
+-----+
|value|
+-----+
|    8|
|    9|
|   10|
|    6|
|    7|
|    3|
|    4|
|    5|
|    1|
|    2|
+-----+
```

这时ADP会自动修剪分区(partition pruning)，过滤掉2016年的分区，在访问文件系统的时候
会下推(pushdown)到具体的目录，也就是说不会去访问2016目录下的文件，那么扫描的文件数
大大缩小，在处理大量数据的时候非常有性能优势