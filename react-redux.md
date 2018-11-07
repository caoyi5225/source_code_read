# React-redux 源码解析

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
  const version = hotReloadingVersion++

  const contextTypes = {
    [storeKey]: storeShape,
    [subscriptionKey]: subscriptionShape,
  }
  const childContextTypes = {
    [subscriptionKey]: subscriptionShape,
  }

  return function wrapWithConnect(WrappedComponent) {
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

    const selectorFactoryOptions = {
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
        this.propsMode = Boolean(props[storeKey])
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
        const sourceSelector = selectorFactory(this.store.dispatch, selectorFactoryOptions)
        this.selector = makeSelectorStateful(sourceSelector, this.store)
        this.selector.run(this.props)
      }

      initSubscription() {
        if (!shouldHandleStateChanges) return
        const parentSub = (this.propsMode ? this.props : this.context)[subscriptionKey]
        this.subscription = new Subscription(this.store, parentSub, this.onStateChange.bind(this))
        this.notifyNestedSubs = this.subscription.notifyNestedSubs.bind(this.subscription)
      }

      onStateChange() {
        this.selector.run(this.props)

        if (!this.selector.shouldComponentUpdate) {
          this.notifyNestedSubs()
        } else {
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
