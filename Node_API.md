# ChainSQL 的 JS API 环境建立及其使用

[ChainSQL项目地址](https://github.com/ChainSQL/chainsqld)

[Node Chainsql API 项目地址](https://github.com/ChainSQL/node-chainsql-api)

## 1.准备工作

#### ChainSQL 安装

安装 MySQL, 配置chainsqld; 参照 [mysql-doc](https://github.com/ycfung/ChainSQL-Docs/blob/master/README.md) 及 [官方文档](https://github.com/ChainSQL/chainsqld/blob/master/doc/manual/deploy.md)

#### 配置

为 ChainSQL API 连接使用的地址, 端口和协议
```
[port_ws_admin_local]
port = 6006
ip = 0.0.0.0
admin = 127.0.0.1
protocol = ws
```

JS API 安装
新建 JS 项目
在 JS 项目下添加包管理文件 `package.json`
文件内容为

```JSON
{
    "chainsql":"^0.6.20"
}
```
终端进入当前目录使用`npm install chainsql --save`安装需要的 node_module, 安装成功后即可开始
使用API

## 2.创建和使用操作对象

使用JS API前先启动 chainsqld, 参照 [mysql-doc](https://github.com/ycfung/ChainSQL-Docs/blob/master/README.md) 及 [官方文档](https://github.com/ChainSQL/chainsqld/blob/master/doc/manual/deploy.md)

本示例在 Node.js 终端运行

#### 1) 引入 ChainSQL
```JS
const ChainsqlAPI = require('chainsql').ChainsqlAPI;
```
#### 2) 创建操作对象

```JS
const c = new ChainsqlAPI();
```


#### 3) 连接

```JS
c.connect("ws://127.0.0.1:6006");
```

#### 4) 填入操作使用的身份

```JS
c.as({
	"secret": "xnoPBzXtMeMyMHUVTgbuqAfg1SUTb",
	"address": "zHb9CJAWyB4zj91VRWn96DkukG4bwdtyTh"
});
```

（此账户是初始的根账户, 拥有最高权限）
#### 5) 递交操作 submit()

使用 submit(option) 函数将操作进行递交, 获取操作结果

可以通过参数 options 来指示操作执行的结果（如发送成功, 入库成功, 验证成功）等于预期的值时才返回, 超时则会引发相应异常, 查询操作带 options 参数时不起作用;
option 的格式为{expect:"...."};例如:

```JS
{expect:"send_success"}
{expect:"validate_success"}
{expect:"db_success"}
```

使用 then(), 可以对 submit() 返回的结果进行操作, 例如:
```JS
submit().then(function(result){
  console.log(result.status);
  console.log(result.tx_hash);
});
```

使用 catch() 可以捕获 submit() 触发的异常, 例如:
```JS
submit().then().catch(function(e) {

	console.log(e.error);
	console.log(e.tx_hash);

};
```

实际用例:

删除“marvel”表
```js
c.dropTable("marvel").submit()
.then(function(result) {

	console.log(result.status);
	console.log(result.tx_hash);

})
.catch(function(e) {

	console.log(e.error);
	console.log(e.tx_hash);

};
```


##### (在下面介绍的几种操作中将省略 then() 和 catch() 处理结果, 以便更简单直接了解其他接口的使用, 在实际使用中请根据需要添加)


## 3. 表操作
使用具有权限的操作对象对表进行更改
#### 1) 创建表 createTable()
使用 create 函数创建表, 其中首参为表名, 第二个参数定义记录的格式, 如:

```JS
c.createTable("tableName", [
	{
		'field':'id',
		'type':'int',
		'length':11,
		'PK':1,
		'NN':1,
		'UQ':1
	},
	{
		'field':'name',
		'type':'varchar',
		'length':50,
		'default':null
	},
	{
		'field':'age',
		'type':'int'
	}]
).submit();
```

参数说明:

| field      | 表字段名                                                     |
| ---------- | ------------------------------------------------------------ |
| type       | 字段名类型, 支持int / float / double / decimal/ varchar / blob / text / datetime |
| length     | 字段值的字节长度                                             |
| PK         | 值为1表示字段为主键                                          |
| NN         | 值为1表示值不能为空,NOT NULL                                 |
| UQ         | 值为1表示值唯一                                              |
| index      | 值为1表示字段为索引                                          |
| FK         | 值为1表示字段为某表的外键, 必须配合REFERENCES使用            |
| REFERENCES | 值的格式为 {'table':'user','field':'id'}                     |

#### 2) 重命名

```JS
c.renameTable(tableName, tableNewName).submit();
```

#### 3) 删除表

```JS
c.dropTable(tableName).submit();
```


## 4. 数据操作
#### 1) 插入数据 insert

```js
c.table(tableName).insert(raw_json).submit();
```

其中 raw_json 必须严格遵守 json 格式
（!!!在官方文档中给出的格式是错误的,在对数据进行操作时, raw 参数需严格遵守下面例子给出的格式, 否则操作失败, 而且在不同操作中使用的格式不同! 为了便于区分, 这里使用 raw_json 和 raw 区别）

例:

```JS
c.table("tableName").insert({'id':1,'name':'peera','age': 22}).submit();
```
插入多个记录时使用数组

```JS
c.table("tableName").insert([{'id':1,'name':'peera','age': 22},{'id':2,'name':'peerb','age': 33}]).submit();
```


#### 2)  获取数据 get

```JS
c.table(tableName).get(raw).submit().submit();
```
例如
```JS
c.table("tableName").get({name: 'peerab'}).submit();
```
Node.js 操作实例

```JS
> var data;
> c.table("tableName").get({name:"peera"}).submit().then(function(result){data=result;});
Promise {
  <pending>,
  domain:
   Domain {
     domain: null,
     _events: { error: [Function: debugDomainError] },
     _eventsCount: 1,
     _maxListeners: undefined,
     members: [] } }
> data;
{ diff: 0,
  lines:
   [ { age: 22, id: 1, name: 'peera' },
     { age: 22, id: 2, name: 'peera' },
     { age: 22, id: 3, name: 'peera' },
     { age: 22, id: 4, name: 'peera' } ] }
```


另外对已有数据进行操作, 首先要用get函数找到数据, 如下面的更新和删除数据

#### 3) 更新数据 update

```JS
c.table(tableName).get(raw).update(raw_json).submit();
```

#### 4)  删除数据 delete

```JS
c.table(tableName).get(raw).delete().submit();
```
