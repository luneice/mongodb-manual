# $lookup 阶段 [官方文档](https://docs.mongodb.com/manual/reference/operator/aggregation/lookup/)

## 目录
- [定义](#定义)
- [基本语法](#基本语法)
- [复杂语法1](#复杂语法1)
- [更多](#更多)
----------

### 定义

对同一数据库中的未整数集合执行左外连接，以从“已连接”集合中过滤文档以进行处理。
对于每个输入文档， ```$lookup``` 阶段添加一个新的数组字段，该数组元素是来自“已连接”集合的匹配文档。 ```$lookup``` 阶段将处理后的文档传递给下一个阶段。

> 考虑这样一个场景，文档 ```people``` 中有一个数组字段 ```likes``` ，其值是对文档 ```book``` 中 ```ObjectId``` 的引用，即 ```people``` 文档的单个元素可以喜欢多个 ```book``` ，那么聚合操作时如何得到 ```people``` 中每个元素所喜欢 ```book``` 的详细信息？ ```$lookup``` 阶段便可以实现这样的聚合，对 ```people``` 中每个元素的 ```likes``` 字段与 ```book``` 中每个元素的 ```ObjectId``` 外连接做聚合操作便可得到 ```people``` 中每个元素所喜欢 ```book``` 的详细信息。

[返回目录](#目录)

### 基本语法

```
{
  $lookup: {
    from: <外连接文档>,
    localField: <输入文档字段>,
    foreignField: <外连接文档字段>,
    as: <聚合操作后输出字段名>
  }
}
```

根据上面的场景，具体聚合操作为

**people 文档**
```
{_id: 1, name: "a", likes: [{_id: 10001}, {_id: 10002}, {_id: 10003}]}
{_id: 2, name: "b", likes: [{_id: 10004}, {_id: 10005}]}
{_id: 3, name: "c", likes: [{_id: 10001}]}
```

**book 文档**
```
{_id: 10001, name: "孙子兵法", peice: 100}
{_id: 10002, name: "论语", peice: 100}
{_id: 10003, name: "资治通鉴", peice: 100}
{_id: 10004, name: "春秋", peice: 100}
{_id: 10005, name: "山海经", peice: 100}
```

**代码**
```
db.people.aggregate([
  {
    $lookup: {
      from: "book",
      localField: "likes",
      foreignField: "_id",
      as: "likes"
    }
  }
])
```

**结果**
```
[
  {_id: 1, name: "a", likes: [
      {_id: 10001, name: "孙子兵法", peice: 100}, 
      {_id: 10002, name: "论语", peice: 100}, 
      {_id: 10003, name: "资治通鉴", peice: 100}
    ]
  },
  {_id: 2, name: "b", likes: [
      {_id: 10004, name: "春秋", peice: 100},
      {_id: 10005, name: "山海经", peice: 100}
    ]
  },
  {_id: 3, name: "c", likes: [
      {_id: 10001, name: "孙子兵法", peice: 100}
    ]
  }
]
```

[返回目录](#目录)

> 你一定会想到，如果 ```people``` 中 ```likes``` 字段元素太多，那么聚合后的 ```likes``` 结果会很大，但我只想得到指定的大小，该怎么操作？

### 复杂语法1
```
{  
  $lookup: {
    from: <collection to join>,
    let: { <var_1>: <expression>, …, <var_n>: <expression> },
    pipeline: [ <pipeline to execute on the collection to join> ],
    as: <output array field>
  }
}
```




























