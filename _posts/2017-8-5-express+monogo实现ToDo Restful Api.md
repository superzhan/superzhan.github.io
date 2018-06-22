---
layout: post
title:  "express+monogo实现ToDo Restful Api"
date:   2017-8-5 15:31:54 +0800
categories: Blog
header-img: "img/tag-bg.jpg"
catalog: true
tags:
    - node.js
---

## Begin

>REST即表述性状态传递（英文：Representational State Transfer，简称REST）是Roy Fielding博士在2000年他的博士论文中提出来的一种软件架构风格。
表述性状态转移是一组架构约束条件和原则。满足这些约束条件和原则的应用程序或设计就是RESTful。需要注意的是，REST是设计风格而不是标准。REST通常基于使用HTTP，URI，和XML（标准通用标记语言下的一个子集）以及HTML（标准通用标记语言下的一个应用）这些现有的广泛流行的协议和标准。REST 通常使用 JSON 数据格式。

本文主要阐述使用node.js的express 框架实现一个ToDo List（任务列表）的Restful 接口。基本的功能包括用户的登录注册、添加任务、完成任务、查看任务等。 最终所有的Api可以在Postman中进行测试。

完整的项目地址： [https://github.com/superzhan/ToDo](https://github.com/superzhan/ToDo)

## 开发环境

mac, webstom, monogoDB , node.js 6.10.1 ,express 4.01

安装、创建express项目还挺简单的，官方还提供一个项目生成器。

```
npm install express-generator -g
```

执行命令就可以全局安装一个express的项目生成器。新建项目的时候执行命令

```
express todo -e
```

新建一个名称为 todo 的express 项目，使用ejs 模板。

执行 `npm install` ,安装相应的模块。

执行 `npm start` , 运行工程。可以通过 http://localhost:3000 访问。

## 基本的Restful 接口示例

新建的工程中会有一些基本的代码。express 中实现Restful 接口是一件相当简单的事情。

添加一个简单的api接口 testAPI. 在 routes/index.js 添加代码

```js
router.get('/testApi',function(req, res, next) {

    res.json({result:'hi ,this is api result'});
});
```
重新启动工程后，可以在浏览器通过http://localhost:3000/testApi 访问这个接口。这个接口使用了GET方法。

在添加一个Post 方法的接口。

```js
router.post('/testApi',function(req, res, next) {

    res.json({result:'hi ,this is api result'});
});
```

这个接口可以再Postman中，通过POST 方法来访问。

![img](/img/2017-8-5-restful/Screen Shot 2017-08-05 at 23.24.20.png)


## 简单添加和获取任务

添加一个addItem 的接口，把客户端发送过来的数据处理之后，插入到数据库中。


```js
router.post('/addItem',function(req,res,next){

  var item = {};
  item.note =req.body.note;
  item.completed=false;
  item.updated_at =timeTool.getCurDate();	
  todoSchema.create(item,function(err,post){
  	if(err)
  	{
  		next(err);
  	}else
  	{
       res.redirect('/');
  	}
  });

});
```

添加一个finishItem 接口，返回已经完成的任务。接口的实现也是简单的数据库查询。

```js
router.post('/finishItem',function(req,res,next){

  var item = {};
  item._id =req.body._id;
  item.completed=true;
  item.updated_at =timeTool.getCurDate();	
  todoSchema.findByIdAndUpdate(item._id,item,function(err,post){
  	if(err)
  	{
  		next(err);
  	}else
  	{
       res.redirect('/');
  	}
  });

});
```



## 数据库mongoose的使用

这个工程使用monogoDb 作为数据库，相对地使用monogoose作为数据库连接模块。monogoose的使用也相当简单。[monogoose Api 文档](http://www.nodeclass.com/api/mongoose.html)

 数据库连接 
 
 新建一个数据库连接模块mongoDB.js ,管理数据库连接。数据库的配置文件是根目录下的 mongoConfig.json, 在这个文件中配置相应的数据库地址和端口号。
 
 具体的可以参考github上的项目[https://github.com/superzhan/ToDo](https://github.com/superzhan/ToDo)
 
 
```js
var config = require('../../mongoConfig.json');

var connectStr = '';
if(config.isAuth)
{
  connectStr='mongodb://'+config.username+':'+config.password+'@'+config.host$
}else
{
  connectStr='mongodb://'+config.host+':'+config.port+'/'+config.database;
}
console.log(connectStr);

var mongoose = require('mongoose');
mongoose.Promise = global.Promise;
mongoose.connect(connectStr)
    .then(function () {console.log('connection succesful')})
    .catch(function (err) {console.error(err)});

module.exports = mongoose;
 ```

数据模型

这个工程用到的数据模型也比较简单，只有任务数据模型和用户数据模型。通过monogoose 预先定义好这两个数据模型。

任务数据模型

```js
var mongoose = require('mongoose');
var timeTool = require('./timeTool');

var ToDoSchema = new mongoose.Schema({
  completed: Boolean,
  note: String,
  userId : mongoose.Schema.Types.ObjectId, //表明任务所属的用户
  updated_at: { type: Date, default: timeTool.getCurDate()},
});
module.exports = mongoose.model('Todo', ToDoSchema);

```

用户数据模型

```js
var mongoose = require('mongoose');
var timeTool = require('./timeTool');

var UserSchema = new mongoose.Schema({

    name : String,
    password :String,
    updated_at: { type: Date, default: timeTool.getCurDate()}
});
module.exports = mongoose.model('User', UserSchema);
```


## 用户注册和登录

注册

用户注册的接口实现需要验证用户的密码是否相同，然后对密码进行哈希加密，把密文存放在数据中。 

而且还需要查找是否有相同名称的用户，若有相同名称的用户，返回注册失败的结果。

```js
router.post('/register',function (req, res, next) {

    var name = req.body.name;
    var password = req.body.password;
    var comfirmPassword = req.body.comfirmPassword;

    if( password != comfirmPassword)
    {
        res.json({code:500,msg:'password is not same'});
        return;
    }

    UserSchema.findOne({'name':name} , function (err, data) {
        if(err)
        {
            res.json({code:500,msg:'check error'});
            return;
        }

        if(data != null)
        {
            console.log(data);
            res.json({code:500,msg:'userName is exist'});
            return;
        }

        var hash = crypto.createHash('sha1');
        hash.update(password);
        password = hash.digest('hex');

        var userInfo={};
        userInfo.name = name;
        userInfo.password= password;

        UserSchema.create( userInfo,function (err,resData) {
            if(err)
            {
                res.json({code:500,msg:'data base error'});
            }else
            {
                res.json({code:200,msg:'success'});
            }
        });

    });

});
```

登录

用户登录时需要把请求的明文密码用哈希算法加密后，再进行数据库查询。

```js
router.post('/login',function (req, res, next) {

    var name = req.body.name;
    var password = req.body.password;

    UserSchema.findOne({'name':name} , function (err, data) {
        if(err)
        {
            res.json({code:500,msg:'data base error'});
            return;
        }

        if(data == null)
        {
            res.json({code:500,msg:'user not exist'});
            return;
        }

        var hash = crypto.createHash('sha1');
        hash.update(password);
        password = hash.digest('hex');
        UserSchema.findOne({'name':name ,'password':password},function (err, data) {
            if(err)
            {
                res.json({code:500,msg:'data base error'});
                return;
            }
            if(data==null)
            {
                res.json({code:500,msg:'password not right'});
                return;
            }


            UserSchema.findOneAndUpdate({name:name ,password:password},
                {updated_at:timeTool.getCurDate()},
                function (err, data) {
                    if(err)
                    {
                        res.json({code:500,msg:'data base error'});
                        return;
                    }
                    res.json({code:200,msg:'success',_id:data._id});
                }
            );

        });

    });

});
```

## API验证 Basic Auth

这些API有一个安全问题，API没有安全验证，任何一台设备都可以通过url进行访问。这个时候就需要对API的访问者进行身份认证。

这里使用最基本的 http Basic Auth 认证，所有的访问请求都需要携带用户的帐号密码信息。

![img](/img/2017-8-5-restful/Screen Shot 2017-08-06 at 21.01.34.png)

这里使用passport 模块，[http://passportjs.org/](http://passportjs.org/) 。passport 是一个node.js的Http验证模块，可以快速开发http验证功能。 这里使用简单的basic auth 验证。

```js
/*导入包*/
var passport = require('passport');
var Strategy = require('passport-http').BasicStrategy;

/*设置验证函数 验证帐号密码*/
passport.use(new Strategy(
    function(username, password, cb) {

        var hash = crypto.createHash('sha1');
        hash.update(password);
        var cryPassword = hash.digest('hex');

        UserSchema.findOne({name:username,password:cryPassword}, function(err, user) {
            if (err) { return cb(err); }
            if (!user) { return cb(null, false); }
            return cb(null, user);
        });
    }));

var authenti=passport.authenticate('basic', { session: false });

```

最后在路由上添加 authenti 进行API验证。

```
router.post('/getItem', authenti,function(req, res, next) {});
```




## 完整的任务接口

这个工程总共有8个任务接口，包括查看任务、添加任务、更新任务和删除任务。具体代码可以参考github 代码仓库。 [https://github.com/superzhan/ToDo](https://github.com/superzhan/ToDo)

```js
router.post('/getItem', authenti,function(req, res, next) {

    TodoSchema.findOne({"_id":req.body._id},function(err, data){
    	if(err){
    		next(err);
    	}else
    	{
    		res.json(data);
    	}
    });

});

/*返回未完成的Item*/
router.post('/getUnDoItem', authenti,function(req, res, next) {

    if(req.body.userId==null)
    {
        res.json({code:500,msg:"request error"});
        return;
    }

    var userId =mongoose.Types.ObjectId(req.body.userId);
    TodoSchema.find({"completed":false ,"userId":userId},function(err, data){
    if(err){
      next(err);
    }else
    {
      res.json(data);
    }
  });

});
```
