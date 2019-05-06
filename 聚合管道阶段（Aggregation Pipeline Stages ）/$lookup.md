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
    as: <聚合操作后输出结果的字段名>
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
    from: <外连接文档>,
    // let 可有可无，与 ES6 let 相似，为管道声明变量。
    let: { 
      // 等同于 a = $a ，其中 a 为管道中要用到的变量，$ 表示对被连接文档字段的引用指令，
      // $a 则对被连接文档字段 a 的引用。
      <变量名 1>: <引用 1>, 
      ...
      <变量名 n>: <引用 n> },
    // 管道中有很多处理阶段，此处的管道为嵌套管道，因为 $lookup 本身就是管道中的一个阶段
    pipeline: [
      {
        // 匹配阶段，指定一些匹配的规则
        // $match: { <query> }
        $match: {
          // 通过 $expr 操作访问 let 中定义的变量
          $expr: {
            // $in 操作接受两个参数，第一个参数为表达式，第二个参数必须是数组，
            // 第一个参数在第二个参数中返回 true 否则返回 false，ES 语法下的支持正则表达式。
            // $filed_a 表示对连接文档字段 filed_a 的引用（管道不可以直接访问被连接文档的字段，
            // 但可以访问连接文档的字段）, $ 表示引用连接文档字段指令 $$ 表示引用 let 中定义的
            // 变量名，故 filed_z 必须是数组。
            $in: [ "$filed_a", "$$filed_z" ]
          }
        },
        // 该阶段聚合操作返回的元素个数最大值为 5
        {"$limit": 5 },
        // 该阶段聚合操作返回的元素个数是从第 8 个元素开始的，跳过了前 7 个
        {"$skip": 7 },
      },
    ],
    as: <聚合操作后输出结果的字段名>
  }
}
```

> 官方文档指出：
> 1. 管道无法访问被连接文档的字段，因此使用 ```let``` 声明管道需要用到哪些被连接文档的字段。
> 2. 除 ```$out``` 和 ```$geoNear``` 阶段之外的所有阶段都可以在管道中多次出现。
> 3.  ```let``` 定义的变量可以被 ```pipeline``` 中其他阶段访问到，即使是嵌套在 ```pipeline``` 中的 ```$lookup`` 阶段。
> 4.  ```$match``` 阶段越早执行越好，即放在管道的最前面。因为 ```$match``` 限制了聚合管道中的文档总数，因此越早执行 ```$match``` 阶段可以最大限度地减少管道的处理量。 ```$match``` 如果放在了管道的开始，那么 ```$match``` 阶段中 ```query``` 就会利用索引的优势查询，类似于 db.collection.find() 或 db.collection.findOne()。


**people 文档**
```
{_id: 1, name: "a", likes: [{_id: 10001}, {_id: 10002}, {_id: 10003}, ..., {_id: 10999}]}
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
...
{_id: 10999, name: "道德经", peice: 100}
```

**代码**
```
db.people.aggregate([
  {
    $lookup: {
      from: "book",
      let: {
        // $likes 表示对被连接文档 likes 字段的引用， $ifNull 表示被连接文档的所有元素
        // 可能有的无 likes 字段，有的有 likes 字段，如果没有则用空数组代替，否则引用它
        likes: { "$ifNull": [ "$likes", [] ] }
      },
      pipeline: [
        "$match": {
          "$expr": {
            // $$likes 表示对 let 中定义的 likes 变量的引用
            "$in": [ "$_id", "$$likes" ]
          }
        },
        // 从第 2 个开始
        {"$skip": 1 },
        // 最多返回 3 个结果
        {"$limit": 3 },
      ],
      as: "likes"
    }
  }
])
```

**结果**
```
[
  {_id: 1, name: "a", likes: [
      {_id: 10002, name: "论语", peice: 100},
      {_id: 10003, name: "资治通鉴", peice: 100},
      {_id: 10004, name: "春秋", peice: 100}
    ]
  },
  {_id: 2, name: "b", likes: [
      {_id: 10005, name: "山海经", peice: 100}
    ]
  },
  {_id: 3, name: "c", likes: []
  }
]
```

[返回目录](#目录)
