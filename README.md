# 手写 Promise 时遇到的问题总结

#### 关于 Promise 中异步问题

```javascript
const fn = () =>
  new Promise((resolve) => {
    setTimeOut(() => {
      resolve(100);
    });
  });

fn().then((res) => console.log(res));
```

- 首先明白一点的是 new Promise 中代码是一个同步执行的过程
- 也就是说在执行 fn()后调用 then() 此时的 setTimeOut 还未执行也就没有进行 resolve，此时的状态任然为 pending
- 所以此时需要将 then 的方法保存起来等待 resolve 执行的时候进行调用

#### 关于 resolvePromise(promise, x, resolve, reject)定义，主要用于处理 then 链式调用时的各种情况

x 与 promise 相等

> - 如果 promise 和 x 指向同一对象(造成死循环)，以 TypeError 为据因拒绝执行 promise

```javascript
const p = promise.then(function () {
  return p;
});
```

x 为 Promise

> - 如果 x 处于等待态， promise 需保持为等待态直至 x 被执行或拒绝
> - 如果 x 处于执行态，用相同的值执行 promise
> - 如果 x 处于拒绝态，用相同的据因拒绝 promise

x 为对象或函数

> - 把 x.then 赋值给 then（**防止恶意修改 Promise**如下）

```javascript
Object.defineProperty(Promise, "then", {
  get: function () {
    throw Error();
  },
});
```

> - 如果取 x.then 的值时抛出错误 e ，则以 e 为据因拒绝 promise
> - 如果 then 是函数，将 x 作为函数的作用域 this 调用之。传递两个回调函数作为参数，第一个参数叫做 resolvePromise ，第二个参数叫做 rejectPromise。（then 返回的也可能是 Promise ，需要递归解决）

```javascript
p1.then((res) => {
  return new Promise((resolve, reject) => {
    resolve(
      new Promise((resolve, reject) => {
        resolve(...);
      })
    );
  });
});
```

> - 如果 resolvePromise 以值 y 为参数被调用，则运行 [[Resolve]](promise, y)
> - 如果 rejectPromise 以据因 r 为参数被调用，则以据因 r 拒绝 promise
> - 如果 resolvePromise 和 rejectPromise 均被调用，或者被同一参数调用了多次，则优先采用首次调用并忽略剩下的调用(**这里主要针对 thenable，promise 的状态一旦更改就不会再改变**)

```javascript
//thenable可能指如下情况
let thenable = {
  then: function (resolve, reject) {
    resolve({
      then: function (resolve, reject) {
        resolve({
          then: function (resolve, reject) {
            resolve(42);
          },
        });
      },
    });
  },
};

var promise1 = new Promise((resolve, reject) => {
  resolve(thenable);
});

promise1.then(function (value) {
  console.log(value);
});
```

> - 如果 then 不是函数，以 x 为参数执行 promise
> - 如果 x 不为对象或者函数，以 x 为参数执行 promise
