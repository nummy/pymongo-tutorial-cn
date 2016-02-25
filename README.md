## pymongo中文教程

这篇教程将为我们讲解如何通过pymongo操作MongoDB。

### 准备

首先，需要安装PyMongo，安装成功之后，就可以正常导入了：
```
>>> import pymongo
```

我们也假设你的开发环境已经安装好了MongoDB，并且运行在默认的主机与端口上。运行mongo的命令如下：
```
$ mongod
```

### 创建MongoClient

使用PyMongo的第一步就是创建一个MongoClient来运行mongod实例：

```Python
>>> from pymongo import MongoClient
>>> client = MongoClient()
```

上面的代码会连接到MongoDB的默认主机与端口，当然，也可以明确指定主机与端口：
```Python
>>> client = MongoClient('localhost', 27017)
```

或者使用URL格式：
```Python
>>> client = MongoClient('mongodb://localhost:27017/')
```

### 获取数据库

一个MongoDB的实例可以操作多个独立的数据库，当我们使用PyMongo的时候可以通过MongoClient的属性来获取不同的数据库。
```Python
>>> db = client.test_database
```

如果数据库名比较特殊，直接使用属性不能获取到，比如"test-databse", 则可以通过字典形式来获取：
```Python
>>> db = client("test-database")
```

### 获取集合

MongoDB中的集合用来保存一组文档，相当于关系型数据库中的数据表，获取集合的方式如下：
```Python
>>> collection = db.test_collection
```

或者：
```Python
>>> collection = db["test_collection"]
```

要注意的是，数据库以及集合都是延迟创建的，也就是说执行上面的命令实际上不会在MongoDB的服务器端进行任何操作，只有当第一个文档插进去的时候，它们才会被创建。

### 文档

MongoDB中的数据采用BSON格式来表示和存储，BSON格式是一种二进制的JSON格式。在PyMongo中采用字典来表示文档，例如，下面的字典可以表示一篇博客信息：
```Python
>>> import datetime
>>> post = {"author": "Mike",
...         "text": "My first blog post!",
...         "tags": ["mongodb", "python", "pymongo"],
...         "date": datetime.datetime.utcnow()}
```

注意文档中可以包含一些Python原生的数据类型，比如datetime.datetime对象，它将会被转换成合适的BSON类型。

### 插入文档

我们使用insert_one()方法来插入文档：
```Python
>>> posts = db.posts
>>> post_id = posts.insert_one(post).inserted_id
>>> post_id
ObjectId('...')
```

如果文档中没有`_id`这个属性，那么当它插入集合中的时候，会自动给它赋予这样一个属性，同一个集合中每个文档的`_id`属性值必须具有唯一性。`insert_one()`返回一个`InsertOneResutl`实例。

插入第一个文档之后，集合posts就被创建了，我们可以查看数据库中已经创建好的集合：
```Python
>>> db.collection_names(include_system_collections=False)
[u'posts']
```

### 使用find_one()获取第一个匹配的文档

MongoDB中最常用的基本操作是`find_one()`, 这个方法返回查询匹配到的第一个文档，如果没有则返回None。下面我们使用find_one()来获取posts集合中的第一个文档：
```Python
>>> posts.find_one()
{u'date': datetime.datetime(...), u'text': u'My first blog post!', u'_id': ObjectId('...'), u'author': u'Mike', u'tags': [u'mongodb', u'python', u'pymongo']}
```

查询结果是一个字典，内容正是我们之前插入的博客内容。

我们也可以使用`find_one()`来查找满足匹配要求的文档。例如，下面的例子查找作者名为"Mike"的文档。
```Python
>>> posts.find_one({"author": "Mike"})
{u'date': datetime.datetime(...), u'text': u'My first blog post!', u'_id': ObjectId('...'), u'author': u'Mike', u'tags': [u'mongodb', u'python', u'pymongo']}
```

如果我们尝试配置作者名"Eliot"，返回结果为空：
```Python
>>> posts.find_one({"author": "Eliot"})
>>>
```

### 通过ObjectId进行查询

我们也可以通过ObjectId属性来进行查找，例如：
```Python
>>> post_id
ObjectId(...)
>>> posts.find_one({"_id": post_id})
{u'date': datetime.datetime(...), u'text': u'My first blog post!', u'_id': ObjectId('...'), u'author': u'Mike', u'tags': [u'mongodb', u'python', u'pymongo']}
```

注意ObjectId与它的字符串表示形式是不一样的：
```Python
>>> post_id_as_str = str(post_id)
>>> posts.find_one({"_id": post_id_as_str}) # No result
>>>
```

在Web应用开发过程中，经常面临的一个任务就是从URL请求中获取ObjectId，然后根据这个ID查找匹配的文档。所以有必要在把ID传递给`find_one()`之前，对于字符串形式的id做一个转换，将其转换成ObjectID对象。
```Python
from bson.objectid import ObjectId

# The web framework gets post_id from the URL and passes it as a string
def get(post_id):
    # Convert from string to ObjectId:
    document = client.db.collection.find_one({'_id': ObjectId(post_id)})
```

### 关于Unicode字符串

你可能注意到我们从服务器端获取的数据与我们之前定义的数据格式并不一样， 例如存到数据库之前为"Mike",从服务器获取之后，结果为"u'Mike",这是为什么？

这是因为，MongoDB采用BSON格式来存储数据，BSON是采用utf-8进行编码的，所以PyMongo必须确保要存储的字符串是合法的utf-8字符串，正常的字符串一般都可存储，但是Unicode字符串不行，所以Unicode字符串在存储之前都必须先转码成utf-8格式，PyMongo在获取数据之后，会将utf-8格式的数据转码成unicode字符串。这也就是为什么会出现编码不一样的问题，

### 批量插入

除了插入单个文档，我们还可以批量插入多个文档，通过给`insert_many()`方法传递一个列表，它会插入列表中的每个文档。
```Python
>>> new_posts = [{"author": "Mike",
...               "text": "Another post!",
...               "tags": ["bulk", "insert"],
...               "date": datetime.datetime(2009, 11, 12, 11, 14)},
...              {"author": "Eliot",
...               "title": "MongoDB is fun",
...               "text": "and pretty easy too!",
...               "date": datetime.datetime(2009, 11, 10, 10, 45)}]
>>> result = posts.insert_many(new_posts)
>>> result.inserted_ids
[ObjectId('...'), ObjectId('...')]
```

### 查询多个文档

为了查询多个文档，我们可以使用`find()`方法，`find()`方法返回一个`Cursor`对象，使用这个对象可以遍历所有匹配的文档。例如下面的例子：
```Python
>>> for post in posts.find():
...   post
...
{u'date': datetime.datetime(...), u'text': u'My first blog post!', u'_id': ObjectId('...'), u'author': u'Mike', u'tags': [u'mongodb', u'python', u'pymongo']}
{u'date': datetime.datetime(2009, 11, 12, 11, 14), u'text': u'Another post!', u'_id': ObjectId('...'), u'author': u'Mike', u'tags': [u'bulk', u'insert']}
{u'date': datetime.datetime(2009, 11, 10, 10, 45), u'text': u'and pretty easy too!', u'_id': ObjectId('...'), u'author': u'Eliot', u'title': u'MongoDB is fun'}
```

跟`find_one()`一样，我们也可以给`find()`方法设置匹配条件，例如，查询作者名为"Mike"的文档：
```Python
>>> for post in posts.find({"author": "Mike"}):
...   post
...
{u'date': datetime.datetime(...), u'text': u'My first blog post!', u'_id': ObjectId('...'), u'author': u'Mike', u'tags': [u'mongodb', u'python', u'pymongo']}
{u'date': datetime.datetime(2009, 11, 12, 11, 14), u'text': u'Another post!', u'_id': ObjectId('...'), u'author': u'Mike', u'tags': [u'bulk', u'insert']}
```

### 统计

如果我们想获取匹配到文档个数，那么可以使用`count()`方法，使用这个方法还可以获取集合中文档的个数：
```Python
>>> posts.count()
3
```

获取获取匹配到的文档个数：
```Python
>>> posts.find({"author": "Mike"}).count()
2
```

### 范围查询

MongoDB支持多种高级查询，比如范围查询，请看下面这个例子：
```Python
>>> d = datetime.datetime(2009, 11, 12, 12)
>>> for post in posts.find({"date": {"$lt": d}}).sort("author"):
...   print post
...
{u'date': datetime.datetime(2009, 11, 10, 10, 45), u'text': u'and pretty easy too!', u'_id': ObjectId('...'), u'author': u'Eliot', u'title': u'MongoDB is fun'}
{u'date': datetime.datetime(2009, 11, 12, 11, 14), u'text': u'Another post!', u'_id': ObjectId('...'), u'author': u'Mike', u'tags': [u'bulk', u'insert']}
```

这个我们使用了`$lt`操作符实现范围查询，还使用`sort()`方法对查询到的结果进行排序。

### 索引

添加索引可以加快查询的速度，在这个例子中，将展示如何创建一个唯一索引。
首先，创建索引:
```Python
>>> result = db.profiles.create_index([('user_id', pymongo.ASCENDING)],
...                                   unique=True)
>>> list(db.profiles.index_information())
[u'user_id_1', u'_id_']
```

注意我们现在有两个索引了，一个在_id上，它由MongoDB自动创建，另外一个就是刚刚创建的索引了。

然后添加几个文档：
```Python
>>> user_profiles = [
...     {'user_id': 211, 'name': 'Luke'},
...     {'user_id': 212, 'name': 'Ziltoid'}]
>>> result = db.profiles.insert_many(user_profiles)
```

这个索引会阻止我们插入已经存在的user_id：
```Python
>>> new_profile = {'user_id': 213, 'name': 'Drew'}
>>> duplicate_profile = {'user_id': 212, 'name': 'Tommy'}
>>> result = db.profiles.insert_one(new_profile)  # This is fine.
>>> result = db.profiles.insert_one(duplicate_profile)
Traceback (most recent call last):
pymongo.errors.DuplicateKeyError: E11000 duplicate key error index: test_database.profiles.$user_id_1 dup key: { : 212 }


```
