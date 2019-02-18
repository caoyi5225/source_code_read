# dva源码解析

## dva-core

### 工具

1. checkModel()
检测model是否合法, 实现代码简单,无需展开解释,具体规则如下:
|项目|规则|
|:---:|:---:|
|namespace|被定义且字符串且唯一|
|state|任意值|
|reducers|空或PlainObject或(数组且[Object, Function])|
|effects|空或PlainObject|
|subscriptions|空或PlainObject且每个值必须为函数|

2. prefixNamespace()
给model的reducers和effects添加'namespace/'前缀,
并检测用户在定义model时,是否自行添加了'namespace/'前缀,
若有则发出警告

3. filterHooks()
过滤合法的hooks, 所有合法的hooks如下:
|hook|意义|
|:---:|:---:|
|onError|effect 执行错误或 subscription 通过 done 主动抛错时触发|
|onStateChange|state 改变时触发|
|onAction|在 action 被dispatch 时触发|
|onHmr|热替换触发|
|onReducer|封装 reducer 执行|
|onEffet|封装 effect 执行|
|extraReducers|指定额外的 reducers |
|extraEnhancers|指定额外的 [StoreEnhancer](https://github.com/reduxjs/redux/blob/master/docs/Glossary.md#store-enhancer) |

4. Plugin
