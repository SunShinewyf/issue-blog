继上次对`express`进行简单地了解和深入之后，又开始倒腾`koa`了。对于`koa`的印象是极好的，简洁而有表现力。和`express`相比它有几个比较明显的特征：
- 比较新潮，`koa1`中使用了`generator`，拥抱`es6`语法，使用同步语法来避免`callback hell`的层层嵌套。在`koa2`中又拥抱了`es7`的`async-await`语法，语法表现形式更为简洁。
- 变得更轻量化，相比于`express`，`koa`抽离了原先内置的中间件，一些比较重要的中间件都使用单独的中间件插件，使得开发者可以根据自己的实际需要使用中间件，更加灵活。就比如给开发者建造了一个简单的地基，之后的装修设计都由开发者自己决定，精简灵活。
关于更多更详细的两者以及和`hapi`的比较，读者可以移步[这里](https://www.airpair.com/node.js/posts/nodejs-framework-comparison-express-koa-hapi)

在源码方面，`koa`变得更加轻量化，但是还是很有特点的。目录结构如下：
```js
- lib/
    - application.js
    - context.js
    - request.js
    - response.js
```
从目录结构来看，只有四个文件，摒弃了`express`中的路由模块，显得简单而有表现力。四个文件中分别定义了四个对象，分别是`app`对象，`context`,`request`以及`response`。深入源码查看，你会发现更加简单，每个文件的代码行数也是很少，而且逻辑嵌套并不复杂。

### `application.js`
首先定义了一个构造函数，源码如下:
```js
function Application() {
  if (!(this instanceof Application)) return new Application;
  this.env = process.env.NODE_ENV || 'development';
  this.subdomainOffset = 2;
  this.middleware = [];
  this.proxy = false;
  this.context = Object.create(context);  //koa的上下文对象
  this.request = Object.create(request);  //koa.request
  this.response = Object.create(response);  //koa.request
}
```
在这段里面只是单纯地定义了一个实例化`app`的一些属性。如上下文，中间件数组等。
然后是注册了一些中间件，中间件的`use`源码很简单，就是将当前的中间件函数`push`到该应用实例的`middleware`数组中，在此不再赘述。
最后定义了开启服务的`lisen`函数，在这个函数里面没有什么特殊的，只是有一点需要注意：它将自身原型的一个`callback`作为参数传入：
```js
  var server = http.createServer(this.callback());
```
这一句很关键，它表示每次在开启`koa`服务的时候，就会执行传入的`callback`,而`callback`都干了些啥，具体看源码:
```js
app.callback = function(){
  if (this.experimental) {
    console.error('Experimental ES7 Async Function support is deprecated. Please look into Koa v2 as the middleware signature has changed.')
  }
  var fn = this.experimental
    ? compose_es7(this.middleware)
    : co.wrap(compose(this.middleware));
  var self = this;

  if (!this.listeners('error').length) this.on('error', this.onerror);

  return function handleRequest(req, res){
    res.statusCode = 404;
    var ctx = self.createContext(req, res);
    onFinished(res, ctx.onerror);
    fn.call(ctx).then(function handleResponse() {
      respond.call(ctx);
    }).catch(ctx.onerror);
  }
};
```
在这个函数里面，通过`experimental`参数作为是否使用`es7`的`async`的标准，当然了，这种折中处理的方式是`koa1`中的，在`koa2`中，由于完全摒弃了`generator`,转而拥抱`async-await`，所以直接使用的`const fn = compose(this.middleware);`就简单进行处理了。对于使用`es7`语法的情况，使用的`compose_es7`对`app`中的中间件数组进行处理：
```js
function compose(middleware) {
  return function (next) {
    next = next || new Wrap(noop);
    var i = middleware.length;
    while (i--) next = new Wrap(middleware[i], this, next);
    return next
  }
}
```
也就是将中间件进行遍历，`compose`函数的作用如下:
```js
compose([f1,f2,...,fn])(args)  =====>  f1(f2(f3(..(fn(args)))));  
```
也就是将数组里面的函数依次执行，通过一个next中间值不断将执行权进行传递。如果传入的中间件数组不是`generator`函数，那么应该是依次执行，但是`generator`有暂停执行的功能，所以一旦执行`yield next`的时候，就会去执行下一个函数。等下一个中间件执行完成时，再在原来中断的地方继续执行。这种执行方式导致形成了`koa`中著名的洋葱模型。
(![输入图片说明](https://raw.githubusercontent.com/SunShinewyf/issue-blog/master/assets/technical/5.png))

举例子如下：
```js
first step before
second step before
second step after
first step after
```
当使用`es7`语法时，处理也是一样的。

### context.js

