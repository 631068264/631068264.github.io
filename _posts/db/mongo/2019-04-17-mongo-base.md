---
layout:     post
rewards: false
title:      mongo 概念
categories:
    - mongo
---

# 概念

![](https://tva3.sinaimg.cn/large/006tNc79ly1g25u6yxv5vj31bc0iaaan.jpg)

![](https://tva3.sinaimg.cn/large/006tNc79ly1g25ui32etaj31am0hy3zt.jpg)

# 集合

集合就是 MongoDB 文档组(**类似于table**) 集合存在于数据库中，集合没有固定的结构，通常情况下我们插入集合的数据都会有一定的关联性。

```
{"site":"www.baidu.com"}
{"site":"www.google.com","name":"Google"}
{"site":"www.runoob.com","name":"菜鸟教程","num":5}
```

## capped collections
MongoDB 的操作日志文件

**固定大小**的collection,按照**插入顺序来保存**（提高增添数据的效率）,必须要显式的创建一个capped collection，**指定一个 collection 的大小**，
单位是字节。collection 的数据存储空间值提前分配的。更新后的文档不可以超过之前文档的大小，这样话就可以确保所有文档在磁盘上的位置一直保持不变。

# 元数据
数据库的信息是存储在集合中。`dbname.system.*`


# 文档(Document)
文档是一组键值 MongoDB 的文档不需要设置相同的字段，并且相同的字段不需要相同的数据类型

- 文档中的键/值对是有序的。
- 文档中的值不仅可以是在双引号里面的字符串，还可以是其他几种数据类型（甚至可以是整个嵌入的文档)。
- MongoDB**区分类型和大小写**。
- MongoDB的文档**不能有重复的键**。
- **文档的键是字符串**。除了少数例外情况，键可以使用任意UTF-8字符


# 数据类型

![](https://tva4.sinaimg.cn/large/006tNc79ly1g25v6k3nlsj312w0u0mz8.jpg)

ObjectId 类似唯一主键，可以很快的去生成和排序，MongoDB 中存储的**文档必须有一个 _id 键**。这个键的值**可以是任何类型**的，默认是个 ObjectId 对象

# 连接

`mongodb://username:password@hostname/dbname`

# 操作

## database

数据库不存在，则创建数据库，否则切换到指定数据库
`use DATABASE_NAME`

查看所有数据库
`show dbs`

删除数据库
`db.dropDatabase()`

## 集合

删除
`db.[collection].drop()`

查看所有集合
`show tables / show collections`

建立
`db.createCollection(name, options)`
![](https://tva1.sinaimg.cn/large/006tNc79ly1g25zy2dq74j31ao0msjss.jpg)


# select

## 条件比较 

|  操作 |	格式   |  范例  |  RDBMS中的类似语句  | 
|---|---| ---| ---| 
| 等于  | `{<key>:<value>}` |	db.col.find({"by":"菜鸟教程"}).pretty() |	where by = '菜鸟教程' |
|   小于	| `{<key>:{$lt:<value>}}` |	db.col.find({"likes":{$lt:50}}).pretty() |	where likes < 50    | 
|   小于或等于|	`{<key>:{$lte:<value>}}` |	db.col.find({"likes":{$lte:50}}).pretty() |	where likes <= 50    | 
|   大于	| `{<key>:{$gt:<value>}}` |	db.col.find({"likes":{$gt:50}}).pretty() |	where likes > 50    | 


## and

每个键(key)以逗号隔开，即常规 SQL 的 AND 条件。

```
db.col.find({key1:value1, key2:value2}).pretty()
```

50<qty<80
```
db.posts.find( {  qty: { $gt: 50 ,$lt: 80}} )
```


## or

```
>db.col.find(
   {
      $or: [
         {key1: value1}, {key2:value2}
      ]
   }
).pretty()
```

## 模糊

查询 title 包含"教"字的文档：
`db.col.find({title:/教/})`

查询 title 字段以"教"字开头的文档：
`db.col.find({title:/^教/})`

查询 titl e字段以"教"字结尾的文档：
`db.col.find({title:/教$/})`


# 索引

```
db.collection.createIndex(keys, options)
```

1 为指定按升序创建索引，如果你想按降序来创建索引指定为 -1

```
db.col.createIndex({"title":1,"description":-1})
```

![](https://tva4.sinaimg.cn/large/006tNc79ly1g26m47yby1j311k0u0ad3.jpg)

查看集合索引
`db.col.getIndexes()`

查看集合索引大小
`db.col.totalIndexSize()`

删除集合所有索引
`db.col.dropIndexes()`

删除集合指定索引
`db.col.dropIndex("索引名称")`

# pymongo

## connect
```python
import pymongo
from bson import ObjectId, SON

mongo_config = {
    'host': 'localhost',
    'port': 27017,
}

db = pymongo.MongoClient(**mongo_config)
# dblist = client.list_database_names()
# print(dblist)
site = db.my.site
inventory = db.my.inventory
```

## insert

```python
d = {"name": "RUNOOB", "alexa": "10000", "url": "https://www.runoob.com"}
res = site.insert_one(d)
print(res.inserted_id)

data_list = [
    {"name": "Taobao", "alexa": "100", "url": "https://www.taobao.com"},
    {"name": "QQ", "alexa": "101", "url": "https://www.qq.com"},
    {"name": "Facebook", "alexa": "10", "url": "https://www.facebook.com"},
    {"name": "知乎", "alexa": "103", "url": "https://www.zhihu.com"},
    {"name": "Github", "alexa": "109", "url": "https://www.github.com"}
]
res = site.insert_many(data_list)
print(res.inserted_ids)
```
## update

```python
query = {'alexa': '10880'}
value = {"$set": {"alexa": "12345"}}
value = {'$inc': {'age': 3}}
unvalue = {"$unset": {"name": "12345"}}
# res = db.update_one(query,value)
# res = db.update_many(query, value, upsert=True)
res = site.update_one(query, unvalue)
print(res)
site.replace_one({'_id': ObjectId('5cb735b4efdf11198ebb92ac')},
                 {"name": "Taobao12323", "url": "https://www.taobao.com"}, )
```

## delete

```python
site.delete_one({'_id': ObjectId('5cb73399efdf11190e096729')})
site.delete_many({'_id': ObjectId('5cb73399efdf11190e096729')})
```
## bulk

```python
site.delete_one({'_id': ObjectId('5cb73399efdf11190e096729')})
site.delete_many({'_id': ObjectId('5cb73399efdf11190e096729')})
```

## select

```python
# 查询  文档中的第一条数据
print(site.find_one())

# 查询集合中所有数据
for s in site.find():
    print(s)

# where
output(site.find_one({'_id': ObjectId('5cb73399efdf11190e096721')}))

# 嵌入doc
cursor = inventory.find(
    {"size": SON(
        [("h", 14), ("w", 21), ("uom", "cm")]
    )}
)

inventory.find({"size.uom": "in"})

# and
output(site.find_one({'name': 'RUNOOB', 'alexa': '100003', }))

query = {"name": {"$lt": "H"}}
mydoc = site.find(query)
for x in mydoc:
    print(x)

# 字段限制 or select xx .limit(1).skip(2) => limit(2,1)
query = {"$or": [{'alexa': '101'}, {'alexa': '12345'}]}
output(site.find(query, {'_id': 0}).limit(1).skip(2))

# order by  1 为升序排列，而 -1 是用于降序排列。
for x in site.find().sort([
    ('alexa', pymongo.DESCENDING), ('name', pymongo.ASCENDING)
]):
    print(x)
# in
query = {'name': {'$in': ['Facebook', 'Taobao']}}
for x in site.find(query):
    print(x)

# array

# 严格 有 顺序一样
inventory.find({"tags": ["red", "blank"]})
# 包含这两个元素 不考虑数组中的顺序或其他元素
inventory.find({"tags": {"$all": ["red", "blank"]}})
# 包含
inventory.find({"tags": "red"})
# 数组位置2
cursor = db.inventory.find({"dim_cm.1": {"$gt": 25}})
# 长度
cursor = db.inventory.find({"tags": {"$size": 3}})

# 数组嵌入
cursor = db.inventory.find({'instock.0.qty': {"$lte": 20}})
cursor = db.inventory.find({"instock.qty": {"$gt": 10, "$lte": 20}})


cursor = inventory.find(
    {"size": SON(
        [("h", 14), ("w", 21), ("uom", "cm")]
    )}
)

inventory.find_one({"size.uom": "in"})
output(inventory.find({"size.uom": "cm"}))



# 聚合
query = [
    # {'$match': {'a': 1}},
    {
        '$group': {
            '_id': "$name",
            'a': {'$sum': 1},
            'b': {'$max': '$alexa'},
        }
    },
    {'$sort': {'a': pymongo.ASCENDING}},
    {'$sort': {'b': pymongo.DESCENDING}},
    {'$limit': 20},
    {'$match': {'a': 1}},

]
# select _id, sum(*) as a, max(alexa) as b  from xx group by name as _id having a=1 order by a asce, b desc
for x in site.aggregate(query):
    print(x)
    
    



```

