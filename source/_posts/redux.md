---
title: redux源码解析
date: 2018-11-30 13:11:33
tags:
---

前几天写完了react-redux的源码解析,接下来看一下redux的源码,继续了解react常用套件的底层原理

## API
redux向我们暴露了6个api,分别是:
1. createStore
2. combineReducers
3. bindActionCreators
4. applyMiddleware
5. compose
5. _____DO_NOT_USE_____ActionTypes

下面就逐个分析这些api的实现:

## createStore
creteStore无疑是redux的核心实现,应重点关注

首先createStore接受三个参数:
1. reducer: 计算下一个状态树的纯函数
2. preloadedState: 初始状态树,由用户定义
3. enhancer: 增强函数,可以通过这里新增中间件等功能

createStore的返回值是一个功能完整的状态容器:
1. state状态树
2. getState: 阅读状态的方法
3. dispatch: (唯一)改变状态的方法
4. subscription: 订阅状态改变的方法

下面看源码:
```javascript
export default function createStore(reducer, preloadedState, enhancer) {
  // 处理没有初始状态,第二个参数传入enhancer的情况
  if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
    enhancer = preloadedState
    preloadedState = undefined
  }

  // 若enhancer是函数则调用并作为返回值返回调用结果
  // 非函数则抛出错误
  // tip: 这里暂时先考虑没有enhancer的情况
  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }

    return enhancer(createStore)(reducer, preloadedState)
  }

  // reducer非函数抛出错误
  if (typeof reducer !== 'function') {
    throw new Error('Expected the reducer to be a function.')
  }

  let currentReducer = reducer
  let currentState = preloadedState
  let currentListeners = []
  let nextListeners = currentListeners
  let isDispatching = false

  function ensureCanMutateNextListeners() {
    if (nextListeners === currentListeners) {
      nextListeners = currentListeners.slice()
    }
  }

  // 读取状态树
  // 如果正在dispatch:
  // 那么不可以获取,并抛出错误
  // 不能保证此时状态是“你想要的”那个状态,因为状态正在改变
  // 为确保取到的状态正确可以从最后一个reducer传入的位置取
  function getState() {
    if (isDispatching) {
      throw new Error(
        'You may not call store.getState() while the reducer is executing. ' +
          'The reducer has already received the state as an argument. ' +
          'Pass it down from the top reducer instead of reading it from the store.'
      )
    }

    return currentState
  }

  // 注册订阅
  // 这里本来有很大一段注释,大致意思是说:
  // 每次dispatch后触发的订阅是dispatch前保存的快照,
  // 在本次dispatch途中或者之后注册的订阅不会被本次dispatch触发
  // tip: 1.注意快照的实现,2.注意对reducer执行中的控制
  function subscribe(listener) {
    // listener不是函数:你没有资格订阅!
    if (typeof listener !== 'function') {
      throw new Error('Expected the listener to be a function.')
    }

    // 在reducer执行是无法注册订阅
    // 想看本次dispatch的结果请getState
    if (isDispatching) {
      throw new Error(
        'You may not call store.subscribe() while the reducer is executing. ' +
          'If you would like to be notified after the store has been updated, subscribe from a ' +
          'component and invoke store.getState() in the callback to access the latest state. ' +
          'See http://redux.js.org/docs/api/Store.html#subscribe for more details.'
      )
    }

    // 闭包保存是否已订阅的状态
    let isSubscribed = true

    // 确保nextListener和currentListener不是指向同一个数组
    // 以通过currenListener保留快照
    ensureCanMutateNextListeners()
    // 注册订阅成功!!!
    nextListeners.push(listener)

    // 注册订阅的返回值是一个注销订阅的函数
    return function unsubscribe() {
      // 未订阅就不需要注销订阅了,直接返回
      if (!isSubscribed) {
        return
      }

      // 还是老问题,reducer执行中无法注销订阅
      if (isDispatching) {
        throw new Error(
          'You may not unsubscribe from a store listener while the reducer is executing. ' +
            'See http://redux.js.org/docs/api/Store.html#subscribe for more details.'
        )
      }

      // 置是否已订阅状态为false
      isSubscribed = false

      // 保留快照
      ensureCanMutateNextListeners()
      // 从订阅列表中删除本订阅
      // 注销成功!!!
      onst index = nextListeners.indexOf(listener)
      nextListeners.splice(index, 1)
    }
  }

  // 触发action
  // 通过reducer和当前状态树计算并返回下一个状态树
  // 通知监听
  function dispatch(action) {
    // 原生dispatch只支持普通对象作为action
    // plainobject 是指字面量或者new Object()创建的对象
    if (!isPlainObject(action)) {
      throw new Error(
        'Actions must be plain objects. ' +
          'Use custom middleware for async actions.'
      )
    }

    // action必须有type
    if (typeof action.type === 'undefined') {
      throw new Error(
        'Actions may not have an undefined "type" property. ' +
          'Have you misspelled a constant?'
      )
    }

    // 还是老问题,正在执行reducer当然不能执行另一个dispatch了
    if (isDispatching) {
      throw new Error('Reducers may not dispatch actions.')
    }

    // 执行reducer,然后更新currentState
    // 终于看到掌控一切的isDispatching状态了,...也很简单=.=
    try {
      isDispatching = true
      currentState = currentReducer(currentState, action)
    } finally {
      isDispatching = false
    }

    // 获取最新的订阅列表快照,并通知它们状态改变
    // 看到这里感觉说订阅列表的快照是保存在本次dispatch完成且通知订阅前时比较合适?
    // 实际上和dispatch前是一样的,因为在dispatch时无法继续注册或注销订阅,那么dispatch
    // 前的快照就是当前位置的快照
    // 要理解订阅列表快照的作用是保证在"通知订阅状态改变"时本次使用的订阅列表不变
    // 而执行dispatch时的订阅列表不变是由isDispatching变量控制的
    const listeners = (currentListeners = nextListeners)
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener()
    }

    // 返回值是本次执行的action
    return action
  }

  // 用于动态修改reducer, 实现方法是替换reducer后触发内置`REPLACE`action
  // `REPLACE`的key是一个随机字符串防止用户“误触”
  function replaceReducer(nextReducer) {
    if (typeof nextReducer !== 'function') {
      throw new Error('Expected the nextReducer to be a function.')
    }

    currentReducer = nextReducer
    dispatch({ type: ActionTypes.REPLACE })
  }

  // 为observable/reactive库预留的交互接口
  // 需要看了observable等库的原理才能解析,不过不影响大局,跳过
  function observable() {
    const outerSubscribe = subscribe
    return {
      subscribe(observer) {
        if (typeof observer !== 'object' || observer === null) {
          throw new TypeError('Expected the observer to be an object.')
        }

        function observeState() {
          if (observer.next) {
            observer.next(getState())
          }
        }

        observeState()
        const unsubscribe = outerSubscribe(observeState)
        return { unsubscribe }
      },

      [$$observable]() {
        return this
      }
    }
  }

  // 创建状态树时会先执行内置`INIT`action
  // 目的是在没有提供preloadedState时创建一个空树
  // 注意:因为`INIT`这个action实际不存在,所以返回的是reducer里的默认返回值
  // 如果reducer里没有默认返回值,那么state将是undefined
  // 所以所有reducer都必须默认在匹配不到type时返回上一个state,
  // 并且没有perloadedState时,需为state设定缺省值
  dispatch({ type: ActionTypes.INIT })

  // 都给你们用!
  return {
    dispatch,
    subscribe,
    getState,
    replaceReducer,
    [$$observable]: observable
  }
}
```

## combineReducers
将多个reducers组合成一个的工具

```javascript
// 输出reducer返回undefined时的错误信息
function getUndefinedStateErrorMessage(key, action) {
  const actionType = action && action.type
  const actionDescription =
    (actionType && `action "${String(actionType)}"`) || 'an action'

  return (
    `Given ${actionDescription}, reducer "${key}" returned undefined. ` +
    `To ignore an action, you must explicitly return the previous state. ` +
    `If you want this reducer to hold no value, you can return null instead of undefined.`
  )
}

// 执行新的reducer过程中的一些警告
function getUnexpectedStateShapeWarningMessage(
  inputState,
  reducers,
  action,
  unexpectedKeyCache
) {
  const reducerKeys = Object.keys(reducers)
  // 因为组合后的新reducer有state的缺省值,所以如果`INIT`action
  // 的state有问题,那么肯定是传了preloadedState的,因为如果不传,
  // 新reducer的state缺省值是{}是没有问题的
  const argumentName =
    action && action.type === ActionTypes.INIT
      ? 'preloadedState argument passed to createStore'
      : 'previous state received by the reducer'

  // 生成的新reducer用的reducers没有合法的
  if (reducerKeys.length === 0) {
    return (
      'Store does not have a valid reducer. Make sure the argument passed ' +
      'to combineReducers is an object whose values are reducers.'
    )
  }

  // 输入的state是否是基本对象
  if (!isPlainObject(inputState)) {
    return (
      `The ${argumentName} has unexpected type of "` +
      {}.toString.call(inputState).match(/\s([a-z|A-Z]+)/)[1] +
      `". Expected argument to be an object with the following ` +
      `keys: "${reducerKeys.join('", "')}"`
    )
  }

  // 检测state中是否存在与reducer不匹配的key
  // 并将其置入缓存
  const unexpectedKeys = Object.keys(inputState).filter(
    key => !reducers.hasOwnProperty(key) && !unexpectedKeyCache[key]
  )

  unexpectedKeys.forEach(key => {
    unexpectedKeyCache[key] = true
  })

  if (action && action.type === ActionTypes.REPLACE) return

  if (unexpectedKeys.length > 0) {
    return (
      `Unexpected ${unexpectedKeys.length > 1 ? 'keys' : 'key'} ` +
      `"${unexpectedKeys.join('", "')}" found in ${argumentName}. ` +
      `Expected to find one of the known reducer keys instead: ` +
      `"${reducerKeys.join('", "')}". Unexpected keys will be ignored.`
    )
  }
}

// 确定reducers是否是合法的
function assertReducerShape(reducers) {
  // 遍历reducers
  Object.keys(reducers).forEach(key => {
    const reducer = reducers[key]
    // 执行`INIT`action的返回值
    // 若是undefined就抛出错误
    // 这里就如上文提到的
    // 1. 需要有state的缺省值
    // 2. 需要在actiontype匹配不到时,返回上一个state(若为undefined则返回缺省值)
    // !!! 这里看到,如果使用combineReducers的话,
    // 因为在检测是传入的state参数是undefined,
    // 所以不论有没有perloadedState, reducer都要遵从上边的原则
    const initialState = reducer(undefined, { type: ActionTypes.INIT })

    if (typeof initialState === 'undefined') {
      throw new Error(
        `Reducer "${key}" returned undefined during initialization. ` +
          `If the state passed to the reducer is undefined, you must ` +
          `explicitly return the initial state. The initial state may ` +
          `not be undefined. If you don't want to set a value for this reducer, ` +
          `you can use null instead of undefined.`
      )
    }

    // 防止某些用户,通过处理ActionsTypes.INIT绕过此检测
    const type =
      '@@redux/PROBE_UNKNOWN_ACTION_' +
      Math.random()
        .toString(36)
        .substring(7)
        .split('')
        .join('.')
    if (typeof reducer(undefined, { type }) === 'undefined') {
      throw new Error(
        `Reducer "${key}" returned undefined when probed with a random type. ` +
          `Don't try to handle ${
            ActionTypes.INIT
          } or other actions in "redux/*" ` +
          `namespace. They are considered private. Instead, you must return the ` +
          `current state for any unknown actions, unless it is undefined, ` +
          `in which case you must return the initial state, regardless of the ` +
          `action type. The initial state may not be undefined, but can be null.`
      )
    }
  })
}

// 将多个不同的reducers合并为一个
export default function combineReducers(reducers) {
  // 获取所有reducers的key
  const reducerKeys = Object.keys(reducers)
  // 排除掉非函数的reducers的合集
  const finalReducers = {}
  for (let i = 0; i < reducerKeys.length; i++) {
    const key = reducerKeys[i]

    // 不存在对应key的reducer,警告
    if (process.env.NODE_ENV !== 'production') {
      if (typeof reducers[key] === 'undefined') {
        warning(`No reducer provided for key "${key}"`)
      }
    }

    // 确认是函数的reducer,添加到finalReducers里
    if (typeof reducers[key] === 'function') {
      finalReducers[key] = reducers[key]
    }
  }
  const finalReducerKeys = Object.keys(finalReducers)

  let unexpectedKeyCache
  if (process.env.NODE_ENV !== 'production') {
    unexpectedKeyCache = {}
  }

  // 检测reducers是否合法(触发匹配不到的actionType不会返回undefined)
  // 如果其中有不和法的reducer,记录错误
  let shapeAssertionError
  try {
    assertReducerShape(finalReducers)
  } catch (e) {
    shapeAssertionError = e
  }

  // 生成新的reducer
  return function combination(state = {}, action) {
    // 存在不合法的reducer,直接抛出错误
    if (shapeAssertionError) {
      throw shapeAssertionError
    }

    // 开发环境,提示警告
    // 1. 检测生成的新reducer用的reducers是否没有可用的
    // 2. 检测输入的state是否是基本对象
    // 3. 检测state的key是否有不与reducer对应的
    if (process.env.NODE_ENV !== 'production') {
      const warningMessage = getUnexpectedStateShapeWarningMessage(
        state,
        finalReducers,
        action,
        unexpectedKeyCache
      )
      if (warningMessage) {
        warning(warningMessage)
      }
    }

    let hasChanged = false
    const nextState = {}
    // 这里新的reducer生成的state是用reducer的key划分的state
    // reducer执行的时候也是改变对应的state[key]
    // 为匹配到的state[key]不会发生改变!!!默认返回上一个state的重要性
    for (let i = 0; i < finalReducerKeys.length; i++) {
      const key = finalReducerKeys[i]
      const reducer = finalReducers[key]
      const previousStateForKey = state[key]
      const nextStateForKey = reducer(previousStateForKey, action)
      if (typeof nextStateForKey === 'undefined') {
        const errorMessage = getUndefinedStateErrorMessage(key, action)
        throw new Error(errorMessage)
      }
      nextState[key] = nextStateForKey
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey
    }
    return hasChanged ? nextState : state
  }
}

```

## bindActionCreators
> actionCreator: 一个创建 action 的函数
> bindActionCreator: 把一个 value 为不同 action creator 的对象，转成拥有同名 key 的对象。同时使用 dispatch 对每个 action creator 进行包装，以便可以直接调用它们

```javascript
function bindActionCreator(actionCreator, dispatch) {
  // 返回一个函数, 我们调用返回的函数时,就会触发dispatch
  return function() {
    return dispatch(actionCreator.apply(this, arguments))
  }
}

export default function bindActionCreators(actionCreators, dispatch) {
  // 如果是一个actionCreator那么直接包装dispatch
  if (typeof actionCreators === 'function') {
    return bindActionCreator(actionCreators, dispatch)
  }

  if (typeof actionCreators !== 'object' || actionCreators === null) {
    throw new Error(
      `bindActionCreators expected an object or a function, instead received ${
        actionCreators === null ? 'null' : typeof actionCreators
      }. ` +
        `Did you write "import ActionCreators from" instead of "import * as ActionCreators from"?`
    )
  }

  // 为每个actionCreator包装dispatch,并用key分割
  const keys = Object.keys(actionCreators)
  const boundActionCreators = {}
  for (let i = 0; i < keys.length; i++) {
    const key = keys[i]
    const actionCreator = actionCreators[key]
    if (typeof actionCreator === 'function') {
      boundActionCreators[key] = bindActionCreator(actionCreator, dispatch)
    }
  }
  return boundActionCreators
}

```
## compose
compose是函数式编程的一个很重要的工具
作用是生成一个把作为参数的函数从左到右地依次把执行结果传入下一个函数的新函数
```javascript
export default function compose(...funcs) {
  // 如果没有参数,那么返回返回原值的函数
  if (funcs.length === 0) {
    return arg => arg
  }

  // 如果只有一个参数,那么直接返回参数
  if (funcs.length === 1) {
    return funcs[0]
  }

  // 如果有一个以上的参数,那么递归生成新函数
  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```

## applyMiddleware
中间件是很重要的一部分,是帮助扩展dispatch的唯一途径
使用方法createStore(reducers, preloadState, applayMiddleWare(...midlewares))
同时这一部分用到了函数时编程的思想,所以比较难理解
> ...middleware (arguments): 遵循 Redux middleware API 的函数。每个 middleware 接受 Store 的 dispatch 和 getState 函数作为命名参数，并返回一个函数。该函数会被传入被称为 next 的下一个 middleware 的 dispatch 方法，并返回一个接收 action 的新函数，这个函数可以直接调用 next(action)，或者在其他需要的时刻调用，甚至根本不去调用它。调用链中最后一个 middleware 会接受真实的 store 的 dispatch 方法作为 next 参数，并借此结束调用链。所以，middleware 的函数签名是 ({ getState, dispatch }) => next => action

```javascript
export default function applyMiddleware(...middlewares) {
  return createStore => (...args) => {
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
        `Dispatching while constructing your middleware is not allowed. ` +
          `Other middleware would not be applied to this dispatch.`
      )
    }
    let chain = []

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    // 这部分比较难理解,不妨假设有三个中间件,分别是f1, f2, f3
    // 为了方便c1 = f1(middlewareAPI), c2 = f2(middlewareAPI), c3 = f3(middlewareAPI)
    // 这里chain = [c1, c2 , c3]
    chain = middlewares.map(middleware => middleware(middlewareAPI))
    // 这里comopse
    // dispatch = c1(c2(c3(store.dispatch)))
    // 那么就是c3最先执行,以真实的store.dispatch传入,作为next
    // next => action => {}
    // 所以返回的是一个action => {}, 这其实是一个新的dispatch
    // 那么c2接受的next就是c3返回的新的dispatch
    // 以此类推,c1最后会返回它自己生成的一个dispatch
    // 所以最后得到的就是一个dispatch
    dispatch = compose(...chain)(store.dispatch)

    // 最后真正执行dispatch(action)的时候,还是c1的next,c2的next,c3的next这样的顺序执行的,最终执行真正的store.dispatch:
    return {
      ...store,
      dispatch
    }
  }
}
```
