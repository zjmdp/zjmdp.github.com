---
layout: post
title: "用Node.js快速搭建后台"
description: ""
category: 
tags: []
---
{% include JB/setup %}


Node是近几年流行起来的后台开发技术，相信大家即使没用过也一定听过，Node的单线程异步IO框架和Javascript + V8的开发平台为其将来流行奠定了很好的基础。单线程模型让开发者摆脱多线程的困扰，异步IO决定了Node可以做到很高的纯IO并发，Javascript意味着拥有⼲大的群众基础以及较高的开发效率，⽽V8保证了的Node运行速度。

我所在的公司90%的后台使⽤了C++作为开发语言，C++作为后台开发语言久经考验，同时也积累了大量的后台框架和开源组件，但考虑到自己所在的团队后台开发人员⽐较紧缺，并且C++熟悉程度⼀般，因此我们决定寻找一种更适合我们的技术。选择C++以外的语言面临最大的问题是公司内部组件⼀般只提供了C++ API，这让我们在选型的时候要充分考虑到是否可以方便的封装这些C++接口。综合各种因素，我们决定选择Node作为我们的平台，从这⼤半年的使用效果来看，Node很好的符合了我们当时选型的⺫标。

由于本人开发Node的时间也不算太长，之前也没做过Javascript相关的开发，所以此⽂主要以交流为目的，表达⾃己对Node的⼀点理解和心得。

##Node的特点

###异步IO复

异步IO最大的好处就是提高了代码执行的效率，异步请求不需要等待结果，后续代码可以立即执⾏，请求结束后通过回调的形式进⾏通知，符合"Don't call me, I will call you"原则，以读取⽂件为例：

```
￼￼var fs = require('fs');
fs.readFile('/Path/To/File1', function(err, result){
	console.log('读取File1完成'); 
});
fs.readFile('/Path/To/File2', function(err, result){
	console.log('读取File2完成');
});
```

File1和File2的读取是并发执行的，整个程序的耗时取决于读取最慢的那个文件的耗时，如果使⽤同步IO的话，耗时就等于两者之和。

经常听到有⼈误把epoll和select归为异步IO，根据Richard Stevens的*“UNIX® Network Programming Volume 1， Third Edition： The Sockets Networking”*中*I/O Models*这一章的介绍，epoll和select属于IO复⽤模型，是同步IO的一种改进模式，如果你对Unix IO几种模型之间的区别不是很清楚的话，建议翻⼀翻上⾯提到的这本书。[这里1](http://blog.csdn.net/historyasamirror/article/details/5778378)，[这里2](http://miaoo.in/talk-about-unix-io-model.html)对Unix IO介绍得比较清楚。

###事件循环

传统的网络编程主要包括以下几种模型：

* 单线程处理所有请求
* 每个请求分配一个线程（进程）
* 分配多个线程，每个线程处理一组请求

以上几种模型都有缺点，第1种方式最大的问题就是性能瓶颈，第2、3种方式需要处理棘手的多线程问题，Node使用事件循环的方式来避免这些问题，它将所有的请求放入一个事件队列，用主线程去处理队列中的请求，而主线程对事件循环的处理速度是十分快的，它通过接收来自不同线程的请求，把需要长时间处理的操作交出去（服务器最多的操作就是IO，充分利用到了Node的异步IO），然后继续接收新的请求。下图展示了请求处理的大致模型：

![Node Process Model](http://infoqstatic.com/resource/articles/nodejs-weakness-cpu-intensive-tasks/zh/resources/0304003.png)


###单线程
这里单线程所指的线程即上面提到执行事件循环的主线程，单线程最⼤的好处就是避免了多线程带来的⼀些弊端，⽐如资源竞争，线程死锁，Debug困难等，简化了编程模型。但很多人会质疑单线程的性能，特别是⼤并发场景，单线程无法很好的利用多核CPU，然而Node的单线程只是针对用户的Javascript代码，Node的底层的、非Javascript代码是多线程的，Node通过线程池来实现异步IO，所以只要我们的代码不执行耗时的操作（所有IO操作都是异步的，Node干得就是这事）即可保证Node的高性能。

###闭包
严格意义上讲闭包是属于Javascript语言本身的特性，之前用C++写过⼿Q国际版的⽓泡翻译后台，这个经历让我明⽩闭包是多么的有用。先⼤概解释⼀下闭包的概念，这⾥的闭包区别于数学概念上的闭包，程序语⾔中的闭包表现为函数运行时可以访问其定义时的上下⽂环境，特别像Node这种异步回调编程模型中，回调的定义和运行处于不同的阶段，能保持上下文变得尤为重要，例如下面的⼀段代码：

```
var server = dgram.createSocket('udp4');
server.on('message', function(message, remote{     
	doAsyncTask(function(err, result){         
		server.send(result, 0, result.length, remote.port, remote.address);     
	});
});
```

```doAsyncTask```的回调函数里调⽤了```server.send```函数，其参数使⽤了```remote```变量，这⾥回调的定义和运行就处于不同的阶段，定义是在执行```doAsyncTask```之前，⽽执行是在 ```doAsyncTask```返回结果以后，但执⾏的时候我们仍然可以访问```remote```变量，⽽```remote```变量是在定义时的上下文中的，当回调执行的时候，这个上下文早已不存在，而闭包提供的就是访问```remote```变量的能⼒。

⼤家可以想象一下，假如```server.send```函数访问不了```remote```变量，我们不得不自己写⼀堆代码来保存这些信息，当并发请求量很⼤的情况下如何管理这些信息可不是⼀件很容易的事情，然而在Node中，这一切显得很自然。

闭包的概念在其他语⾔中也⽐较常⻅，⼀般来说函数式语言⼤多提供了闭包特性（闭包最先出现在Lisp的方言Scheme中），就连Java的匿名函数也有闭包的影子。不过滥用闭包可能会导致内存泄露，特别是回调函数定义时的上下文中存在占⽤内存较大的对象。

###Node适合干什么？

Node什么都能干，但是更擅⻓IO密集型，为什么这么说？Node的主线程通过事件循环处理不同事件的回调函数，这意味着你写的回调函数要尽可能避免做耗时的操作，⽐如你不太适合在异步读取图片的回调中对图片做耗时的图像处理，因为所有异步回调都在这个主线程中的事件队列里排队执行，从业务上来看就是无法及时响应⽤户的请求。

那Node是否不能处理CPU密集型业务呢？本文最开始提到了Node使用了Google的V8引擎，这让JS的执行速度几乎是所有脚本语言中最高的，同时Node还提供了child_process来应对这种业务场景，通过将复杂耗时的计算分发给child_process，通过进程间的通信来传递结果信息。


##一些经验分享


###Callback Hell

虽然异步IO可以⾼效处理并发IO，但这是以抛弃传统编程习惯为代价的，⼤量使⽤异步IO很容易导致回调嵌套过深，即所谓的Callback Hell，⽐如我们很容易写出以下风格的代码：

```
asyncFun1(param, function (err, data1) {
	if (err) return cb(err);
	asyncFun2(data1, function (err,data2) {
		if (err) return cb(err);
		asyncFun3(data2, function (err, data3) {
			if (err) return cb(err);
			cb(data3); 
		});
	})
})
```


这种风格的代码可读性和维护性都存在一定问题，并且写代码的⼈也很容易被绕进去，因此诞⽣了各种流程控制库来解决Callback Hell的问题，⽐较著名的有async，它提供各种函数来对异步流程来进⾏控制，⽐如使⽤async中的函数```waterfall(tasks, [callback])```来解决上⾯的回调嵌套问题：

```
async.waterfall([
     asyncFun1(param, callback){
         // do some stuff...
         callback(null, 'data1');
     },
     asyncFun2(data, callback){
         // do some more stuff...
         callback(null, 'data2');
	 },
	 asyncFun3(data, callback){
         // do some more stuff...
         callback(null, 'done');
     }
 ],
function(err, results){
     // results is now equal to ['data1', 'data2', 'data3']
});
```
 
async的使⽤能⼤大缓解Callback Hell现象，除了流程控制库，我们还可以使用事件发布/订阅的模式，这是⼀种常见的异步编程模式：

```
emitter.on('event1', function(result){
	asyncFun2(function(err, result){
		if(err) return;
		//do something...
       emitter.emit('event2', 'data2');
	}) 
})
emitter.on('event2', function(result){
	asyncFun3(function(err, result){
		if(err) return;
       //do something...
	})
})
asyncFun1(param, function(err, result){
	if(err) return;
	//do something...
   emitter.emit('event1', 'data1');
})
```

发布/订阅模式不仅能解决Callback Hell的问题，同时它还能解耦业务逻辑，事件发布者和事件监听者之间除了事件本身以外(⽐如上例中的事件名称event1和event2)，两者之间不需要有任何联系。

然而我们在解决Callback Hell的问题上并未采用上述的流程库以及事件发布/订阅，⽽使用了一种叫Promise/A⻛格的编程模式，相信做前端开发的同事应该对此很熟悉。⼀个Promise代表⼀个任务的执⾏结果，这个任务的状态可以是unfulfilled， fulfilled或者failed，Promise范式唯一需要的接口就是通过then⽅法来注册任务fulfilled和failed时的回调函数，这样的解释也许不太严谨，但⼤致表达了Promise范式的基本模型。这里将上面的例子写成Promise风格：￼

```
￼ promisefyAsyncFun1(param)
     .then(function(result){
         return promisefyAsyncFunc2(result);
     })
     .then(function(result){
         promisfyAsyncFunc3(result);
     })
     .catch(err){
         console.error("Error: " + err.message);
     }
```


**那Promise能带来什么?**
从上⾯的例⼦可以看出，我们通过then将原本嵌套回调的代码顺序串联在一起，这种风格和同步式的风格⾮常相似，Promise⼏乎做到了和同步⻛格代码⼀样的编写和阅读体验，下面再通过一个具体的业务代码片段来说明这个问题：

```
function doHandleReq(){
        verifySig(req)
            .then(getOpenId)//获取openid
            .then(buildRechargeUrl)//构造充值url
            .then(rechargeUseProxy)//发起充值请求
            .then(respondToMidas)//响应充值结果给midas server
            .then(notifyMP)//通知充值结果给公共账号
            .catch(function(e){
                res.json(404, {"ret": ErrorInfo.UNKNOWN_ERROR.code, "msg": ErrorInfo.UNKNOWN_ERROR.msg})
            });
    }
```

这段代码反映了整个充值过程的主体框架，我相信不用过多讲解⼤家就能明白这段代码的主要逻辑，如果使用传统的异步回调嵌套来实现估计没有10分钟是⽆法完全搞清整个代码的跳转流程的。这种⼀⽓呵成的coding体验和阅读体验不正是我们追求的么？大家是否注意到最后的catch，前面then中的方法一旦抛出异常，并且异常没有在方法中捕获，那所有的异常将最终在这里的catch中被捕获，这也方便了我们对异常做统一的处理。

###C++ API封装

在我司使用Node必然会面临封装C++的问题，当时我们选型的时候最担心的是有没有⼀个简单可靠的⽤于封装现有组件的Node库，最终我们选定了[node-ffi](https://github.com/node-ffi/node-ffi)，node-ffi的使⽤非常方便，下面通过封装某C++库的例⼦介绍下大致的流程：

* 编写C⻛格的接口(node-ffi只⽀持C接口的封装)

```
class L5ClientApiWrap
{
public:
     static int L5ApiGetRoute(uint32_t modid, uint32_t cmdid, uint32_t *out_ip, uint32_t *out_port, float tm_out);
};
//---------------------------------------------------
extern "C" {
     int getRoute(uint32_t modid, uint32_t cmdid, uint32_t *out_ip, uint32_t *out_port, float tm_out)
     {
         return L5ClientApiWrap::L5ApiGetRoute(modid, cmdid, out_ip, out_port, tm_out);
     }
}
#endif
```

* 实现接口(调⽤组件真正的API)。

```
￼#include "L5ClientApiWrap.h"
int L5ClientApiWrap::L5ApiGetRoute(uint32_t modid, uint32_t cmdid, uint32_t *out_ip, uint32_t *out_port, float tm_out)
{
    QOSREQUEST req;
    req._modid = modid;
    req._cmd = cmdid;
    std::string err_msg;
    int ret = ApiGetRoute(req, tm_out, err_msg);
    if(ret == 0){
        uint32_t ip;
        inet_pton(AF_INET, req._host_ip.c_str(), &ip);
        *out_ip = ntohl(ip);
        *out_port = req._host_port;
	}
	return ret; 
}
```

* 编写make脚本,将我们自己写的C风格的接口和实现，以及组件库编译成so文件￼

```
￼CXX = g++
CXXFLAGS = -g -Wall -fPIC -shared
OBJECTS = L5ClientApiWrap.o
TargetName = libL5ClientApiWrap.so
PROCESSOR = $(shell uname -p)
ifeq ($(PROCESSOR), x86_64)
LIB = ./libqos_client64.a
else
LIB = ./libqos_client32.a
endif
:.PHONY
.PHONY:all
all::$(TargetName)
$(TargetName):$(OBJECTS) $(LIB)
    $(CXX) $(CXXFLAGS) $^  -o $@
$(OBJECTS): %.o: %.cpp
    $(CXX) $(CXXFLAGS)  -c $^  -o $@
clean:
    rm -rf *.so *.o *.gch
```

* 使⽤node-ffi编写js⽂文件，load⽣成的so，暴露相应的js接⼝

```
var ref = require('ref');
var ffi = require('ffi');
var uint32Ptr = ref.refType('uint32');

var l5_api = ffi.Library(__dirname + '/../l5_lib/libL5ClientApiWrap',{
    'getRoute': ['int', ["uint32", "uint32", uint32Ptr, uint32Ptr, "float"]]]
});

function getRoute(modid, cmdid, tmOut){
    var ipInfo = {};
    var ip = require('ref').alloc('uint32');
    var port = require('ref').alloc('uint32');
    var ret = l5_api.getRoute(modid, cmdid, ip ,port, tmOut);
    if(ret == 0){
        ipInfo.ip = number2ipString(ip.deref());
        ipInfo.port = port.deref();
    }else{
        ipInfo.errCode = ret;
    }
    return ipInfo;
}

exports.getRoute = getRoute;
```


###其它⽤用到的库

除了上⾯提到的node-ffi，我们在开发过程还主要⽤到了下⾯的⼀些库：

* [bluebird](https://github.com/petkaantonov/bluebird)：Promise的一种实现
* [express](http://expressjs.com)：node上最流行的module之一，主要用来开发web应用，但我们的应用场景不涉及⻚⾯的render，所以[restirfy](http://mcavage.me/node-restify/)更适合我们
* [request](https://github.com/mikeal/request)：强⼤的http请求库，内置http module的替代 
* [mocha](http://visionmedia.github.io/mocha/)：⽤来写测试的
* [chai](http://chaijs.com)：BDD/TDD assertion库，内置assert的强化 
* [tracer](https://github.com/baryon/tracer)：log库，不过⽐较推荐winston
* [forever](https://github.com/nodejitsu/forever)：充当守护进程的角色，保证服务一直在后台，[pm2](https://github.com/Unitech/pm2)也不错。

##总结
Node是近几年流行起来的后台开发技术，其基于单线程事件驱动的异步IO模型赢得了不少喝彩，但其在关键性的后台的搭建上还缺乏一些时间考验，不过目前也有不少大公司已经开始使用Node，比如LinkedIn、Yahoo、阿里巴巴，[这里](https://github.com/joyent/node/wiki/Projects,-Applications,-and-Companies-Using-Node)有更详细的列表。我个人的建议是如果项目本身没什么历史包袱，而项目进度又比较赶，需要快速开发原型，那Node会是一个很不错的选择。

**最后欢迎大家一起交流，共同进步。**