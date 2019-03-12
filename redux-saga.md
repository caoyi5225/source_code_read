# Redux-Saga 源码解析
看完了redux源码,对于中间件很感兴趣,正好项目中用到了redux-saga,所以看一下redux-saga的源码

## 介绍
> redux-saga 是一个用于管理应用程序 Side Effect（副作用，例如异步获取数据，访问浏览器缓存等）的 library，它的目标是让副作用管理更容易，执行更高效，测试更简单，在处理故障时更容易。 可以想像为，一个 saga 就像是应用程序中一个单独的线程，它独自负责处理副作用。 redux-saga 是一个 redux 中间件，意味着这个线程可以通过正常的 redux action 从主应用程序启动，暂停和取消，它能访问完整的 redux state，也可以 dispatch redux action。

## API
1. createSagaMiddleware
2. effects
3. utils

## createSagaMiddleware
创建saga中间件,用于和redux建立连接,是redux-saga的核心代码
这个方法是一个工厂函数用于创建saga: sagaMiddlewareFactory
```javascript
import { is, asap, isDev, check, warnDeprecated } from './utils'
import proc from './proc'
import emitter from './emitter'
import { MONITOR_ACTION } from './monitorActions'
import SagaCancellationException from './SagaCancellationException'

// That's it!
export default function sagaMiddlewareFactory(...sagas) {
  let runSagaDynamically

  // 遍历所有传入的saga, 检测有不是function的抛出错误
  sagas.forEach((saga, idx) => check(saga, is.func, sagaArgError('createSagaMiddleware', idx, saga)))

  // 要创建的sagamiddleware
  function sagaMiddleware({getState, dispatch}) {
    // sagaEmitter是一个触发器,有两个api:
    // subscribe: 添加一个函数,@param function, @return 移除该函数的方法
    // emit: 触发所有函数并传入参数
    const sagaEmitter = emitter()
    const monitor = isDev ? action => asap(() => dispatch(action)) : undefined

    const getStateDeprecated = () => {
      warnDeprecated(GET_STATE_DEPRECATED_WARNING)
      return getState()
    }

    function runSaga(saga, ...args) {
      return proc(
        saga(getStateDeprecated, ...args),
        sagaEmitter.subscribe,
        dispatch,
        getState,
        monitor,
        0,
        saga.name
      )
    }

    runSagaDynamically = runSaga

    sagas.forEach(runSaga)

    return next => action => {
      const result = next(action) // hit reducers
      // filter out monitor actions to avoid endless loops
      // see https://github.com/yelouafi/redux-saga/issues/61
      if(!action[MONITOR_ACTION])
        sagaEmitter.emit(action)
      return result;
    }
  }

  sagaMiddleware.run = (saga, ...args) => {
    if(!runSagaDynamically) {
      throw new Error(RUN_SAGA_DYNAMIC_ERROR)
    }
    check(saga, is.func, sagaArgError('sagaMiddleware.run', 0, saga))

    const task = runSagaDynamically(saga, ...args)
    task.done.catch(err => {
      if(!(err instanceof SagaCancellationException))
        throw err
    })
    return task
  }

  return sagaMiddleware
}
```

## effects
1. take
2. put
3. race
4. call
5. apply
6. cps
7. fork
8. join
9. cancel
10. select

### take

