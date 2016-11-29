## RxJS 阅读笔记

> RP 是针对异步数据流的编程。
一定程度而言，RP并不算新的概念。Event Bus、点击事件都是异步流。开发者可以观测这些异步流，并调用特定的逻辑对它们进行处理。使用Reactive如同开挂：你可以创建点击、悬停之类的任意流。通常流廉价(点击一下就出来一个)而无处不在，种类丰富多样：变量，用户输入，属性，缓存，数据结构等等都可以产生流。举例来说：微博回文(译者注：比如你关注的微博更新了)和点击事件都是流：你可以监听流并调用特定的逻辑对它们进行处理。

基于流的概念，RP赋予了你一系列神奇的函数工具集，使用他们可以合并、创建、过滤这些流。 一个流或者一系列流可以作为另一个流的输入。你可以 合并 两个流，从一堆流中 过滤 你真正感兴趣的那一些，将值从一个流 映射 到另一个流。

### 流事件 map、scan、filter
- map(fn) 是对原始得的流通过 fn 方法进行转换成新的流。
- scan((value, current) => value + current, 0) 对原始的流进行累加，只有发射过的才会进行累加。
- filter 过滤，和数组的过滤方法类似

### 创建一个单值流 Rx.Observable.just('xxx')

``` js
var requestStream = Rx.Observable.just('https://api.github.com/users');
```

### 订约一个 Observable

```js
var requestStream.subscribe(function(url) {
  // 获得单值流里面的数据
});
```

### 自定义流 Rx.Observable.create()

jquery 的 ajax 操作返回的是一个 Promise。而 Promise 也是属于客观察对象。

> 可观察对象(Observable)是超级Promise(原文Promise++，可以对比C，C++，C++在兼容C的同时引入了面向对象等特性)。 在Rx环境中，你可以简单的通过var stream = Rx.Observable.fromPromise(promise)将Promise转换为可观察对象

``` js
requestStream.subscribe(function(url) {
  var responseStream = Rx.Observable.create(function(observer) {
    $.getJSON(url).done(function(resp) {
      observer.onNext(resp);
    }).fail(function(jqXHR, status, error) {
      observer.onError(jqXHR, status, error);
    }).always(function() {
      observer.onCompleted();
    });
  });

  responseStream.subscribe(function(resp) {
    // TODO 获取异步请求数据
  }, function() {
    // 请求失败
  }, function() {
    // 请求完成
  });
});
```

### 使用 map 函数转化流

我们使用 map 函数将请求 URL 的流转化成为 ajax 响应请求的流

``` js
var responseStream = requestStream.map(function(url) {
  return Rx.Observable.fromPromise($.getJSON(url));
});
```

我们把上面代码执行后的返回结果称为 metastream (译者注：按字面可以翻译为“元流”，即包含流的流。类似概念例如：元编程——用于生成程序的编程方法；元知识——获取知识的知识)：包含其他流的流。没什么吓人的， 一个metastream会在执行后发射一个流。 你可以把它看做一个指针 指针)： 每一个发射的值是指向另外一个流的 指针 。在我们的例子中，每一个URL被映射为一个指向Promise流的指针，每一个Promise流中包含了相应的响应信息。

对于包含流的流的解析方式如下：

``` js
responseStream.subscribe(function(streamPromise) {
  // 展开 metastream,获取内部的流
  streamPromise.subscribe(function(responseJsonObject) {
    // 返回内部流发射的值
    return responseJsonObject;
  });
});
```

### flatMap() 函数将枝干的流发射到主干上

```js
var responseStream = requestStream.flatMap(function(url) {
  return Rx.Observable.fromPromise($.getJSON(url));
});
```
``` html
请求流:  --a-----b--c------------|->
响应流:  -----A--------B-----C---|->

(小写字母表示请求, 大写字母代表响应)
```

总结下代码：

```js 
var requestStream = Rx.Observable.just('https://api.github.com/users');

var responseStream = requestStream
  .flatMap(function(requestUrl) {
    return Rx.Observable.fromPromise(jQuery.getJSON(requestUrl));
  });

responseStream.subscribe(function(response) {
  // 在浏览器中渲染响应数据的逻辑
});
```

### 将事件监听器转话为可观察的流对象

```js
var refreshButton = document.querySelector('.refresh');
var refreshClickStream = Rx.Observable.fromEvent(refreshButton, 'click');
var requestStream = refreshClickStream.map(function() {
    var randomOffset = Math.floor(Math.random()*500);
    return 'https://api.github.com/users?since=' + randomOffset;
  });
```

由于如果点击，就不会发生流，所以我们需要一个默认的初始得了流

``` js
var startupRequestStream = Rx.Observable.just('https://api.github.com/users');
```

### merge() 合并流

```js
var requestOnRefreshStream = refreshClickStream
  .map(function() {
    var randomOffset = Math.floor(Math.random()*500);
    return 'https://api.github.com/users?since=' + randomOffset;
  });
  
var startupRequestStream = Rx.Observable.just('https://api.github.com/users');

var requestStream = Rx.Observable.merge(
  requestOnRefreshStream, startupRequestStream
);
```

``` html
流 A: ---a--------e-----o----->
流 B: -----B---C-----D-------->
          vvvvvvvvv merge vvvvvvvvv
          ---a-B---C--e--D--o----->
```

### startWith(x) 默认将 x 作为这个流的起始输入

```js
var requestStream = refreshClickStream
  .map(function() {
    var randomOffset = Math.floor(Math.random()*500);
    return 'https://api.github.com/users?since=' + randomOffset;
  })
  .startWith('https://api.github.com/users');
```

### 参考资料
[https://segmentfault.com/a/1190000004293922](https://segmentfault.com/a/1190000004293922)
