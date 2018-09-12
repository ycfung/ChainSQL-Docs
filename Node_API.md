# ChainSQL 的 JS API 环境建立及其使用

## 介绍
本文档是参照chainsql提供的[JS_API](http://www.chainsql.net/api_javascript.html)编写，与原文档内容大体相同，根据项目源代码和实际操作结果，加以个人理解，对文档内容进行修正和补充

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

#### 5）选择需要进行操作的库的所有者 use()
在对库操作之前我们需要先用use(adress)表明要使用谁创建的库，默认情况下，不使用use()函数时，将直接使用操作账户创建的库(即as中的adress)

例:
```JS
c.use("zP8Mum8xaGSkypRgDHKRbN8otJSzwgiJ9M")
```
(注意:要操作别的账户创建的库，首先要获得别的用户的授权，后面将会介绍授权操作grant())


#### 6) 递交操作 submit()

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
c.renameTable("tableName", tableNewName).submit();
```

#### 3) 删除表

```JS
c.dropTable("tableName").submit();
```


## 4. 数据操作
#### 1) 插入数据 insert

```js
c.table("tableName").insert(raw_json).submit();
```

其中 raw_json 必须严格遵守 json 格式
（在对数据进行操作时, raw 参数需严格遵守下面例子给出的格式, 否则操作失败, 而且在不同操作中使用的格式不同! 为了便于区分, 这里使用 raw_json 和 raw 区别）

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
c.table("tableName").get(raw).update(raw_json).submit();
```

#### 4)  删除数据 delete

```JS
c.table("tableName").get(raw).delete().submit();
```

## 5.用户管理
#### 1)创建新的账户 generateAddress()
使用操作对象的generateAddress()函数可以创建一个新的账户，函数返回得到新账户的密码，地址和公钥
```JS
>c.generateAddress();
{
	"secret":"xcUd996waZzyaPEmeFVp4q5S3FZYB",
	"address":"zP8Mum8xaGSkypRgDHKRbN8otJSzwgiJ9M",
	"publicKey":"02B2F836C47A36DE57C2AF2116B8E812B7C70E7F0FEB0906493B8476FC58692EBE"
}
```
创建了账户后我们就可以根据上面的方法的登陆该用户对数据库进行操作了，等等，账户是在本地创建的，要使用他，首先我们要对他进行激活，下面我们介绍用pay()方法来激活用户

#### 2）转账pay()
每个账户都有一个钱包用于储存金币，其中在系统默认的根账户中具有100000000000000000个金币，在创建新的账户的时候，需要拥有金币的的账户使用pay(adress,count)对其进行转账并成功才能激活用户,count表示转账的金币数，最少为5

例：

```JS
c.pay('znvVbBAJgLZg5scAN4kXBnMuTxU1VtyCvM',5000);
```
(注意:在使用pay函数时不要使用submit()函数)

#### 3) 授予对表操作的权限grant
上面提到了使用createTable()创建新的表，这时候我们可以使用表的创建者对这个表进行增删查改的操作，但是这个表对别的账户是不可视的，如果要对别的账户进行授权操作，我们需要用到grant("tableName",user,rightInfo)函数
操作的权限分为select insert update delete，授权的格式参照下面的示例

例如我在账户zHb9CJAWyB4zj91VRWn96DkukG4bwdtyTh下创建了表，想对znvVbBAJgLZg5scAN4kXBnMuTxU1VtyCvM授权查和写的操作
```JS
c.grant("tableName","znvVbBAJgLZg5scAN4kXBnMuTxU1VtyCvM",{select: true, insert: true}).submit();
```
下面的可以用as登陆新的账户，对该表进行操作了，别忘了使用
```JS
c.use("zHb9CJAWyB4zj91VRWn96DkukG4bwdtyTh");
```
如果要取消刚刚的授权，可以使用
```JS
c.grant("tableName","znvVbBAJgLZg5scAN4kXBnMuTxU1VtyCvM",{select: false, insert: false}).submit();
```
## 6.创建交易

上面提到的需要入链的操作，每执行一条命令就会对应生成一笔交易，然而在实际使用中，一笔交易往往涉及多个数据库的基本操作，它可能同时需要添加新数据，删除或更新旧的数据，这时候我们需要单独创建交易

#### 1)  开始交易beginTran()
使用
```JS
c.beginTran()
```
交易开始后，可以开始写入交易的操作内容
如：
```JS
c.table("tableName1").insert({'id':100,'name':'peersafe', 'age':20}, {'id':200,'name': 'zongxiang', 'age': 21});
c.table("tableName2").get({id: 1}).assert({age:22, name: 'peersafe'});
c.table("tableName2").get({id: 1}).update({'age':52, 'name': 'zongxiang'});
c.table("tableName1").delete({'id':1});
```

#### 2) 递交交易commit()

在上面写到的操作代码中，并没有使用submit函数对操作进行递交，因为所有的操作应该是作为一个交易一起递交的，在写入所有的操作后，使用commit()递交交易
与逐个操作递交的区别除了只生成单个交易以外，所有的操作都是交易的一部分，只有所有的操作都通过了验证，交易才能通过验证，只要其中一个操作失败，所有操作驳回
```JS
c.commit()
```
至于commit函数的具体用法，与submit()函数是相同的，所以这里不多作解释

#### 3) 断言操作assert()
一笔交易可能是复杂的，它能否执行除了需要正确的操作以外，可能还需要以数据库中的一些信息作为先决条件，这时候我们需要在交易的操作描述中加入断言操作assert()对数据库里的某些信息进行获取和判断，如果条件不符合，我们直接驳回整笔交易
例:
```JS
c.table("posts").get({id: 1}).assert({age:'52',name:'lisi'});
```

## 7.获取区块信息
直接对区块链及链上信息进行获取，不会创建交易，十分简单
为了查看结果我们需要用到一个回调函数，为便于操作，我们先定义一个简单的函数callback把结果log出来
```JS
var callback = function(err,data){console.log(err);console.log(data)}
```
下面是函数的含义及用法
```JS
c.getLedger(option,callback)                //获取区块链的相关信息
c.getLedgerVersion(callback)                //获取区块链的版本
c.getTransactions(adress,callback)          //根据地址查看某个账户进行的交易
c.getTransaction(hash,callback)             //根据交易哈希获取某个交易
```

其中，getLedger操作可以选择性地获取信息
如:
```JS
option={
    limit:12,                       //交易条数
    ledgerVersion :666,             //版本号
                                    //下面是返回信息中含有哪些内容
	includeAllData : false,
	includeTransactions : true,
	includeState : false
} 
```

## 8.订阅
```JS
c.subscribeTable(owner_adress, tablename, callback);        //对某张表进行订阅(callback函数参考第七部分的定义)
c.unsubcribeTable(owner, tablename);                        //取消订阅

c.subscribeTx(txid, callback);                              //对某个交易进行订阅(txid是指交易哈希)
c.unsubscribeTx(txid);                                      //取消订阅
```

