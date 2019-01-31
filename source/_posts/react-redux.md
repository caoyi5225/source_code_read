---
title: react-redux 源码解析
date: 2019-01-31 12:40:10
tags:
---

## api

``` javascript
export { Provider, createProvider, connectAdvanced, connect }
```

从 index.js 可以知道, react-redux对外暴露了4个api,接下来我们去看每个api的实现

## Provider

```javascript
class Provider extends Component {
  // Provider作为顶层组件,通过context向下分发store
  // 之后Connect组件可以获取store
  getChildContext() {
    return { [storeKey]: this[storeKey], [subscriptionKey]: null }
  }

  constructor(props, context) {
    super(props, context)
    this[storeKey] = props.store;
  }

  render() {
    // 返回children里仅有的子级。否则抛出异常
    // Provider必须有且仅有一个子组件
    return Children.only(this.props.children)
  }
}

Provider.propTypes = {
  store: storeShape.isRequired,
  children: PropTypes.element.isRequired,
}

Provider.childContextTypes = {
  [storeKey]: storeShape.isRequired,
  [subscriptionKey]: subscriptionShape,
}

// store的格式
export const storeShape = PropTypes.shape({
  subscribe: PropTypes.func.isRequired,
  dispatch: PropTypes.func.isRequired,
  getState: PropTypes.func.isRequired
})

// subscription的格式
export const subscriptionShape = PropTypes.shape({
  trySubscribe: PropTypes.func.isRequired,
  tryUnsubscribe: PropTypes.func.isRequired,
  notifyNestedSubs: PropTypes.func.isRequired,
  isSubscribed: PropTypes.func.isRequired,
})

```
这里我们可以看到,Provider组件的作用就是作为根组件,向整个react树通过context分发store

## createProvider

```javascript
export function createProvider(storeKey = 'store') {
  // createProvider为自定义storeKey的Provider提供了创建接口
  const subscriptionKey = `${storeKey}Subscription`
  // ...
  return Provider;
}
```

## connectAdvance

因为connectAdvance的返回值是一个HOC(高阶组件),所以不熟悉HOC的话可以移步[HOC](https://react.docschina.org/docs/higher-order-components.html)
```javascript

export default function connectAdvanced(

  /**
   * selectFactory 负责从state,props,dispath计算新的props
   * 定义方式如下:
   * options是传递给connectAdvance的第二个参数和WrappedComponent的组合
   * export default connectAdvanced((dispatch, options) => (state, props) => ({
   *   thing: state.things[props.thingId],
   *   saveThing: fields => dispatch(actionCreators.saveThing(props.thingId, fields)),
   * }))(YourComponent)
   */
  selectorFactory,
  {
  
    // 用来组织HOC的名字
    getDisplayName = name => `ConnectAdvanced(${name})`,

    // 错误提示时展示用的名字
    methodName = 'connectAdvanced',

    // 如果已定义,则指定对应的子组件的render次数
    // 用于在react-devtools观察不必要的render
    renderCountProp = undefined,

    // 表明HOC是否订阅state改变
    shouldHandleStateChanges = true,

    // 从context/props中获取store的key
    storeKey = 'store',

    // 是否需要ref,详见HOC介绍
    withRef = false,

    // 其余的配置
    ...connectOptions
  } = {}
) {
  const subscriptionKey = storeKey + 'Subscription'
  // 热加载版本
  const version = hotReloadingVersion++

  const contextTypes = {
    [storeKey]: storeShape,
    [subscriptionKey]: subscriptionShape,
  }
  const childContextTypes = {
    [subscriptionKey]: subscriptionShape,
  }

  return function wrapWithConnect(WrappedComponent) {
    // 如果不是生产环境,检测被HOC包裹的对象是否为合法的组件
    if (process.env.NODE_ENV !== 'production') {
      invariant(
        isValidElementType(WrappedComponent),
        `You must pass a component to the function returned by ` +
        `${methodName}. Instead received ${JSON.stringify(WrappedComponent)}`
      );
    }

    const wrappedComponentName = WrappedComponent.displayName
      || WrappedComponent.name
      || 'Component'

    const displayName = getDisplayName(wrappedComponentName)

    // 要传给selectorFactory的参数
    const selectorFactoryOptions = {
      // 用户要传入的参数
      ...connectOptions,
      getDisplayName,
      methodName,
      renderCountProp,
      shouldHandleStateChanges,
      storeKey,
      withRef,
      displayName,
      wrappedComponentName,
      WrappedComponent
    }
    class Connect extends Component {
      constructor(props, context) {
        super(props, context)

        this.version = version
        this.state = {}
        this.renderCount = 0
        this.store = props[storeKey] || context[storeKey]
        // 从props是否传入了store
        this.propsMode = Boolean(props[storeKey])
        // for ref
        this.setWrappedInstance = this.setWrappedInstance.bind(this)

        invariant(this.store,
          `Could not find "${storeKey}" in either the context or props of ` +
          `"${displayName}". Either wrap the root component in a <Provider>, ` +
          `or explicitly pass "${storeKey}" as a prop to "${displayName}".`
        )

        this.initSelector()
        this.initSubscription()
      }

      getChildContext() {
        const subscription = this.propsMode ? null : this.subscription
        return { [subscriptionKey]: subscription || this.context[subscriptionKey] }
      }

      componentDidMount() {
        if (!shouldHandleStateChanges) return
        this.subscription.trySubscribe()
        this.selector.run(this.props)
        if (this.selector.shouldComponentUpdate) this.forceUpdate()
      }

      componentWillReceiveProps(nextProps) {
        this.selector.run(nextProps)
      }

      shouldComponentUpdate() {
        return this.selector.shouldComponentUpdate
      }

      componentWillUnmount() {
        if (this.subscription) this.subscription.tryUnsubscribe()
        this.subscription = null
        this.notifyNestedSubs = noop
        this.store = null
        this.selector.run = noop
        this.selector.shouldComponentUpdate = false
      }

      getWrappedInstance() {
        invariant(withRef,
          `To access the wrapped instance, you need to specify ` +
          `{ withRef: true } in the options argument of the ${methodName}() call.`
        )
        return this.wrappedInstance
      }

      setWrappedInstance(ref) {
        this.wrappedInstance = ref
      }

      initSelector() {
        // 为selectorFactory注入dispatch
        const sourceSelector = selectorFactory(this.store.dispatch, selectorFactoryOptions)
        // 为selector注入state,并通过mapstate计算nextProps来标记是否需要更新组件
        // props和nextProps存储在selector中,在两次run时用来比较
        this.selector = makeSelectorStateful(sourceSelector, this.store)
        this.selector.run(this.props)
      }

      initSubscription() {
        // 不监听直接返回
        if (!shouldHandleStateChanges) return
        // 如果是props传来的store则从props订阅,如果不是则从context订阅
        const parentSub = (this.propsMode ? this.props : this.context)[subscriptionKey]
        // 为当前组件创建一个新的订阅
        this.subscription = new Subscription(this.store, parentSub, this.onStateChange.bind(this))
        this.notifyNestedSubs = this.subscription.notifyNestedSubs.bind(this.subscription)
      }

      // state改变的回调
      onStateChange() {
        // 比较props, 看是否需要更新组件
        this.selector.run(this.props)

        // 如果props没变,不需要更新组件,则通知下边的组件store改变,检测是否需要更新
        // 如果触发了更新,则通过setState强制更新,通过didupdate延后通知子组件检查更新
        // !!!在本组件update之后,子组件实际已经update,所以在didupdate后通知子组件检查更新
        // 并不会造成再次渲染子组件,不需要担心嵌套connect的性能问题
        if (!this.selector.shouldComponentUpdate) {
          this.notifyNestedSubs()
        } else {
          // 在didupdate延后通知子组件检查更新
          this.componentDidUpdate = this.notifyNestedSubsOnComponentDidUpdate
          this.setState(dummyState)
        }
      }

      notifyNestedSubsOnComponentDidUpdate() {
        this.componentDidUpdate = undefined
        this.notifyNestedSubs()
      }

      isSubscribed() {
        return Boolean(this.subscription) && this.subscription.isSubscribed()
      }

      addExtraProps(props) {
        if (!withRef && !renderCountProp && !(this.propsMode && this.subscription)) return props
        const withExtras = { ...props }
        if (withRef) withExtras.ref = this.setWrappedInstance
        if (renderCountProp) withExtras[renderCountProp] = this.renderCount++
        if (this.propsMode && this.subscription) withExtras[subscriptionKey] = this.subscription
        return withExtras
      }

      render() {
        const selector = this.selector
        selector.shouldComponentUpdate = false

        if (selector.error) {
          throw selector.error
        } else {
          return createElement(WrappedComponent, this.addExtraProps(selector.props))
        }
      }
    }

    Connect.WrappedComponent = WrappedComponent
    Connect.displayName = displayName
    Connect.childContextTypes = childContextTypes
    Connect.contextTypes = contextTypes
    Connect.propTypes = contextTypes

    if (process.env.NODE_ENV !== 'production') {
      Connect.prototype.componentWillUpdate = function componentWillUpdate() {
        // 处理热加载
        // 如果版本号改变则说明出发了热加载
        // 更新版本号
        // 重新初始化订阅,并解除订阅后重新订阅原有订阅
        if (this.version !== version) {
          this.version = version
          this.initSelector()
          let oldListeners = [];

          if (this.subscription) {
            oldListeners = this.subscription.listeners.get()
            this.subscription.tryUnsubscribe()
          }
          this.initSubscription()
          if (shouldHandleStateChanges) {
            this.subscription.trySubscribe()
            oldListeners.forEach(listener => this.subscription.listeners.subscribe(listener))
          }
        }
      }
    }

    return hoistStatics(Connect, WrappedComponent)
  }
}
```

## Subscription
订阅的实现是整个框架实现的核心部分之一, 它让每个Container管理自己的一个订阅状态, 并通过将子树叶的订阅集中在父树叶处理,来确保每次state更新造成的dom更新是自上而下的,保证了子树叶不会重复更新
```javascript
// encapsulates the subscription logic for connecting a component to the redux store, as
// well as nesting subscriptions of descendant components, so that we can ensure the
// ancestor components re-render before descendants

const CLEARED = null
const nullListeners = { notify() {} }

function createListenerCollection() {
  // the current/next pattern is copied from redux's createStore code.
  // TODO: refactor+expose that code to be reusable here?
  let current = []
  let next = []

  return {
    clear() {
      next = CLEARED
      current = CLEARED
    },

    notify() {
      const listeners = current = next
      for (let i = 0; i < listeners.length; i++) {
        listeners[i]()
      }
    },

    get() {
      return next
    },

    subscribe(listener) {
      let isSubscribed = true
      if (next === current) next = current.slice()
      next.push(listener)

      return function unsubscribe() {
        if (!isSubscribed || current === CLEARED) return
        isSubscribed = false

        if (next === current) next = current.slice()
        next.splice(next.indexOf(listener), 1)
      }
    }
  }
}

export default class Subscription {
  constructor(store, parentSub, onStateChange) {
    this.store = store
    this.parentSub = parentSub
    this.onStateChange = onStateChange
    this.unsubscribe = null
    this.listeners = nullListeners
  }

  addNestedSub(listener) {
    this.trySubscribe()
    return this.listeners.subscribe(listener)
  }

  notifyNestedSubs() {
    this.listeners.notify()
  }

  isSubscribed() {
    return Boolean(this.unsubscribe)
  }

  trySubscribe() {
    if (!this.unsubscribe) {
      // !!!划重点!敲黑板!
      // 没有父级观察者的话，直接监听store change
      // 有的话，添到父级下面，由父级传递变化
      this.unsubscribe = this.parentSub
        ? this.parentSub.addNestedSub(this.onStateChange)
        : this.store.subscribe(this.onStateChange)
 
      this.listeners = createListenerCollection()
    }
  }

  tryUnsubscribe() {
    if (this.unsubscribe) {
      this.unsubscribe()
      this.unsubscribe = null
      this.listeners.clear()
      this.listeners = nullListeners
    }
  }
}
```
