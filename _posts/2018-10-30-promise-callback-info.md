---
layout: post
title: promise和callback在微信小程序异步调用中的应用
category : Linux
tags : [小程序,promise,callback,javascript]
---

# 背景
近期，因为工作需要在研究微信小程序。
微信小程序基本是通过Javascript+css+wxml(类html)组合而成。对于精通前端技术的人来说，javascript中的promise、callback应该是非常熟悉了，但是对于javascript小白来说，这类技术还是需要研究。本文即是对javascript中的这两个概念的具体应用进行举例分析。

# 需求
小程序中的主体逻辑是用js实现的，并且小程序中大部分网络交互的接口实现都是异步的，因此在写小程序时，不可避免的就必须和异步进行打交道。接下来举个简单的例子，来看看callback和promise在异步接口中的应用。

需求举例：
1. 首先我在数据库的medicine表中插入了3条药品信息记录，每条记录简单的包含药品的一些基础信息。
2. 在小程序中，我们需要从medicine表中查出所有药品记录进行展示

解决方案：
1. 传统的方案，我们写个同步接口，先查数据库再进行数据输出即可
2. 但是在小程序中，微信提供的数据库访问接口都是异步的，因此不能用简单的同步模式写代码，这时就需要使用callback或者promise

# 实现方案1：传统同步模式
![image.png](https://upload-images.jianshu.io/upload_images/13183512-6b03a9adfa335979.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


1. 按照同步模式的理解，以上代码第8行，调用this.getDbData函数查询数据库，在函数内将查询结果赋值给全局变量data中的medicine中，并输出查询结果
2. 第9行，在调用了getDbData函数之后，再输出变量data的内容，发现medicine变量并没有被赋值，如下图
![image.png](https://upload-images.jianshu.io/upload_images/13183512-7ffa3bca2fab57ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3. 从上边的执行结果可以看到，第9行先执行了，getDbData中的输出还后执行了
这就是因为getDbData中调用数据库的接口，在微信API中是异步实现的，因此要想实现查询获取了相关数据之后，再对数据做进一步处理就需要使用callback机制或者promise

# 实现方案2：callback模式改造
![image.png](https://upload-images.jianshu.io/upload_images/13183512-dfd84eb99eacded8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. 如上，在第8行增加了this.process_data作为callback传入getDbData函数中
2. 第12~14行，是callback函数的定义，并在该函数中输出相关全局变量的内容
3. 第25行，是执行完数据库查询并且完成全局变量赋值之后，将全局变量传给callback函数并调用，整体执行结果如下：
![image.png](https://upload-images.jianshu.io/upload_images/13183512-a54e600a114330cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 实现方案3：promise模式改造
![image.png](https://upload-images.jianshu.io/upload_images/13183512-c9e67601b5caaa59.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. 如上，第15行重新定义了promise模式的函数，getDataByPromise返回的是一个promise对象
2. 在第7行调用函数之后，在then中执行相关后置动作，具体执行效果如下：
![image.png](https://upload-images.jianshu.io/upload_images/13183512-3058eab0e0fe5f2d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以上，即为这几种模式的对比。
附，完整代码：
```

Page({
  data: {
    "medicine": "N/A",
  },
  onLoad: function (options) {
    // callback模式处理异步调用
    // var that = this;
    // this.getDbData("medicine", {}, "medicine", this.process_data)
    // console.log("in onLoad:", this.data)

    // promise模式处理异步调用
    this.getDataByPromise("medicine", {}).then((data) => { 
      this.setData({
        "medicine": data
      })
      console.log("in onLoad promise:", this.data)
    })
  },

  getDataByPromise: function (coll_name, search_cond) {
    var promise = new Promise((resolve, reject) => {
      var that = this;
      const db = wx.cloud.database();
      db.collection(coll_name).where(search_cond).get({
        success: function (res) {
          console.log("in promise info:", res.data)
          resolve(res.data)
        },
        error: function (e) {
          console.log(e)
          reject("查询数据库失败")
        }
      });
    });
    return promise;
  },

  process_data: function(data){
    console.log("in onLoad callback:", this.data)
  },
  getDbData: function (coll_name, search_cond, data_key, cb) {
    const db = wx.cloud.database()
    var that = this;
    var ready_data = {};
    db.collection(coll_name).where(search_cond).get({
      success: function (res) {
        ready_data[data_key] = res.data;
        that.setData(ready_data);
        console.log("查询数据库完成:", that.data, res.data);
        cb(that.data)
      }
    })
  }
})
```
