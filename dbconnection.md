#ADP上存取数据库技巧

在大数据平台上我们经常需要访问外部系统，比如将计算结果写入外部数据库，那么按常规逻辑，
我对数据库建立一个连接，然后写入数据就行了，但是因为ADP是一个分布式系统，事情并不像看
上去那么简单。

1. 我们来看代码示例，比如我们要访问mongodb(此处用的是mongodb的Java驱动)
```scala
    val mongoClient = new MongoClient()
    val db = mongoClient.getDatabase("test")
    val coll = db.getCollection("testCollection")

    myDataRdd.foreach {
        data =>
            //计算数据
            ...
            //写入数据
            coll.insertOne(doc)  //Runtime Error！
    }
```
代码很常见，计算了数据写到数据库，但是运行却报错：
```console
Caused by: java.io.NotSerializableException: com.mongodb.MongoCollectionImpl
Serialization stack:
	- object not serializable (class: com.mongodb.MongoCollectionImpl, value: com.mongodb.MongoCollectionImpl@60ab7329)
	- field (class: ...$anonfun$main$1, name: afColl$1, type: interface com.mongodb.client.MongoCollection)
	- object (class ...$$anonfun$main$1, <function1>)
	...
	... 26 more

```
为什么呢？
错误原因在于变量`db`的数据库连接是在驱动(driver)端建立，`foreach`计算却是在工作(worker)端，
那么`coll`需要从驱动端传到工作端，报错是`coll`的序列化出错，即使传输不出错，
在工作端上与`coll`相对应的数据库连接也是不存在的。

2. 怎么解决这个问题，当然是在工作端建立连接而不是在驱动端建立连接传过去
代码这样：
```scala
myDataRdd.foreach {
    data =>
        val mongoClient = new MongoClient()
        val db = mongoClient.getDatabase("test")
        val coll = db.getCollection("testCollection")
        //计算数据
        ...
        //写入数据
        coll.insertOne(doc)
}
```
这样程序运行就不会报错了。可是，这样其实是在每插入一条记录的时候都建立一个链接，效率太低。
能否一个工作端建立一个连接而不是一个数据建立一个连接？

3. 使用方法`foreachPartition`,可以做到
```scala
myDataRdd.foreachPartition {
    partitionData =>
        val mongoClient = new MongoClient()
        val db = mongoClient.getDatabase("test")
        val coll = db.getCollection("testCollection")

        partitionData.foreach {
            data =>
                //计算数据
                ...
                //写入数据
                coll.insertOne(doc)
        }
}
```

当然程序还可以再进一步优化，使用连接池，那么在一个工作端连接就可以重用，代码就不罗列了。
