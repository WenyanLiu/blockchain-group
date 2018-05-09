# 数据库访问工具

Plyvel是访问LevelDB的Python接口。

#### 使用Plyvel访问LevelDB

```python
import plyvel
import binascii

db = plyvel.DB('/home/wyliu/Documents/go-ethereum/data/geth/chaindata')

# print(db.get(b'480f95db58f1c988a990d36bc35383997f34be690ab6b0c61cbf4855fd7f93de79'))

for number, record in enumerate(db):
    print(number)
    (key, value) = record
    print("key = " + binascii.hexlify(key))
    print("value = " + binascii.hexlify(value))
    print

db.close()
```

其中，基本操作有：

* 通过创建新的`DB`实例来打开新的数据库`db = plyvel.DB(path/to/database)`，关闭数据库：`db.close()`
* 使用`for`循环遍历所有数据，按照字典序key顺序返回所有key/value对

```python
for key, value in db:
    print(key)
    print(value)
```

* 获取数据`db.get(b'key')`

:warning: 注意编码格式的转换！

