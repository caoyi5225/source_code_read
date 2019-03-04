# Then/Promise 源码解析
Then/Promise 是ES6 Promise的一个polyfill, 学习其源码有助于我们深入理解Promise

### Promise类
```javascript
function Promise(fn) {
  if (typeof this !== 'object') {
    throw new TypeError('Promises must be constructed via new');
  }
  if (typeof fn !== 'function') {
    throw new TypeError('Promise constructor\'s argument is not a function');
  }
  this._deferredState = 0;
  this._state = 0;
  this._value = null;
  this._deferreds = null;
  if (fn === noop) return;
  doResolve(fn, this);
}
```
我们可以看到, Promise的构造函数很简单, 做了三件事情:
- 调用方式和参数检测
- 初始化属性
|名称|类型|默认值|描述|
|----|----|------|----|
|_state|Number|0|Promise状态,0-pending,1-fulfilled且_value不是promise,2-rejected,3-fulfilled且_value是promise|
|_deferredState|Number|0|deferred状态|
|_value|Any|null|Promise 内部值，resolve 或者 reject返回的值|
|_deferreds|Array|null|存放 Handle 实例对象的数组，缓存 then 方法传入的回调|
- 调用excute函数

### final方法
```javascript
function finale(self) {
  if (self._deferredState === 1) {
    handle(self, self._deferreds);
    self._deferreds = null;
  }
  if (self._deferredState === 2) {
    for (var i = 0; i < self._deferreds.length; i++) {
      handle(self, self._deferreds[i]);
    }
    self._deferreds = null;
  }
}
```
用来在resolve或reject时执行.then保存的异步任务
具体逻辑就是通过deferredState判断有几个异步任务,
如果有一个就直接handle, 两个及以上就遍历handle,
然后清空异步任务列表

### doResolve方法
```javascript
function doResolve(fn, promise) {
  var done = false;
  var res = tryCallTwo(fn, function (value) {
    if (done) return;
    done = true;
    resolve(promise, value);
  }, function (reason) {
    if (done) return;
    done = true;
    reject(promise, reason);
  });
  if (!done && res === IS_ERROR) {
    done = true;
    reject(promise, LAST_ERROR);
  }
}
```
本方法意在执行传入Promise的函数,
需要注意的地方是, 通过done字段来确保fn只执行一次,
另外,tryCallTwo的作用是将第二、三个参数作为参数调用第一个参数, 如果报错则返回错误信息

### resove方法
```javascript
function resolve(self, newValue) {
  // 不能把自身作为resolved的参数
  if (newValue === self) {
    return reject(
      self,
      new TypeError('A promise cannot be resolved with itself.')
    );
  }
  if (
    newValue &&
    (typeof newValue === 'object' || typeof newValue === 'function')
  ) {
    var then = getThen(newValue);
    if (then === IS_ERROR) {
      return reject(self, LAST_ERROR);
    }
    // resolve新的promise
    if (
      then === self.then &&
      newValue instanceof Promise
    ) {
      self._state = 3;
      self._value = newValue;
      finale(self);
      return;
    // 不是promise但.then是一个function  
    } else if (typeof then === 'function') {
      doResolve(then.bind(newValue), self);
      return;
    }
  }
  // 参数是一个普通值
  self._state = 1;
  self._value = newValue;
  finale(self);
}
```
我们可以看到,resolve方法对不同的参数处理方式是不同的
  1. 如果是本promise对象本身,则报错;
  2. 如果是新promise, 则将_state置为3, 赋值_value, 执行final;
  3. 如果不是新promise但是.then是function, 则将.then作为新的一个处理函数通过doResolve调用;
  4. 如果以上均不匹配(普通值), \_state置为1, 赋值_value, 执行final;

### reject方法
```javascript
function reject(self, newValue) {
  self._state = 2;
  self._value = newValue;
  if (Promise._onReject) {
    Promise._onReject(self, newValue);
  }
  finale(self);
}
```
reject方法相对简单, state置为2, 赋值_value, 执行Promise._onReject, 执行finale

### then方法
```javascript
Promise.prototype.then = function(onFulfilled, onRejected) {
  if (this.constructor !== Promise) {
    return safeThen(this, onFulfilled, onRejected);
  }
  var res = new Promise(noop);
  handle(this, new Handler(onFulfilled, onRejected, res));
  return res;
};
```
then方法是实现链式调用的核心, 实现原理就是每次都返回一个新的promise.另外, then将生成一个异步任务在resolve之后执行

### Handler类
```javascript
function Handler(onFulfilled, onRejected, promise){
  this.onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : null;
  this.onRejected = typeof onRejected === 'function' ? onRejected : null;
  this.promise = promise;
}
```
该类的属性:
|名称|类型|默认值|描述|
|----|----|------|----|
|onFulfilled|Function|null|resolve的回调|
|onRejected|Function|null|reject的回调|
|promise|Function|null|要返回的新promise|

### handle方法
```javascript
function handle(self, deferred) {
  while (self._state === 3) {
    self = self._value;
  }
  if (Promise._onHandle) {
    Promise._onHandle(self);
  }
  if (self._state === 0) {
    if (self._deferredState === 0) {
      self._deferredState = 1;
      self._deferreds = deferred;
      return;
    }
    if (self._deferredState === 1) {
      self._deferredState = 2;
      self._deferreds = [self._deferreds, deferred];
      return;
    }
    self._deferreds.push(deferred);
    return;
  }
  handleResolved(self, deferred);
}
```
这个方法是处理异步任务的核心方法之一, 我们看下处理的逻辑:
  1. 如果在handle执行的时候, promise的状态已经变为了fulfilled, 且返回的值是新的promise, 那么可以跳过当前promise, 直接处理下一个promise
  2. 执行全局绑定的Prmomise._onHanlde方法
  3. 如果promise处于pendding状态, 则将异步任务放入defferreds数组, deferredState记录异步任务数量, 超过1个都记录为2
  4. 调用handleResolved调用异步任务

### handleResolved方法
```javascript
function handleResolved(self, deferred) {
  asap(function() {
    var cb = self._state === 1 ? deferred.onFulfilled : deferred.onRejected;
    if (cb === null) {
      if (self._state === 1) {
        resolve(deferred.promise, self._value);
      } else {
        reject(deferred.promise, self._value);
      }
      return;
    }
    var ret = tryCallOne(cb, self._value);
    if (ret === IS_ERROR) {
      reject(deferred.promise, LAST_ERROR);
    } else {
      resolve(deferred.promise, ret);
    }
  });
}
```
顾名思义,本方法的作用是处理已经resolved的promise通过.then创建的任务
本方法用到了一个工具[asap](https://github.com/kriskowal/asap),官方介绍是:
> executes a task as soon as possible but not before it returns, waiting only for the completion of the current event and previously scheduled tasks 
> 即: 在当前事件和先前计划的任务完成后尽快执行 

接下来看逻辑:
  1. 如果state是1, 则说明该promise是fulfilled, 如果不是1, 则说明该promise是rejected(因为在handle方法里会跳过所有state为3的promise, 所以fulfilled的promise只有state为1的)
  2. 果未传入对应onFulfilled和onFejected的方法, 直接resolve/reject handler
  3. 否则, 先执行对应的方法再resolve/reject handler并将对应处理方法返回的数据作为value传入

### 总结
1. 能够链式调用的原因是每个then都会返回一个新的Promise(state = 3)
2. 能够实现异步执行的原因是把每个异步任务都放在最后一个Promise里执行的
