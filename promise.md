# Then/Promise 源码解析
Then/Promise 是ES6 Promise的一个polyfill, 学习其源码有助于我们深入理解Promise

1. Promise构造函数
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
|_state|Number|0|Promise状态,0-pending,1-fulfilled且_value是promise,2-rejected,3-fulfilled且_value不是promise|
|_deferredState|Number|0|deferred状态|
|_value|Any|null|Promise 内部值，resolve 或者 reject返回的值|
|_deferreds|Array|null|存放 Handle 实例对象的数组，缓存 then 方法传入的回调|
- 调用excute函数

2. doResolve方法
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

3. resove方法
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

4. reject方法
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
