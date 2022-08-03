先从index.js这个入口文件开始，入口文件导出了几个在当前文件中引入的文件
```javascript
import createStore from './createStore'
import combineReducers from './combineReducers'
import bindActionCreators from './bindActionCreators'
import applyMiddleware from './applyMiddleware'
import compose from './compose'
import warning from './utils/warning'
...
function isCrushed() {}
...
export {
  createStore,
  combineReducers,
  bindActionCreators,
  applyMiddleware,
  compose
}
```
1.createStore

根据导出的顺序，我们先看createStore函数，该函数接收3个参数，返回4个方法

```javascript
function createStore(reducer, preloadedState, enhancer) {
  return  {
    dispatch,
    subscribe,
    getState,
    replaceReducer
  }
}

```
reducer: reducer是个一函数，该函数返回一个全新的包含了所有数据的state
```javascript
// 一个常见的 reducer 函数
function counter(state, action) {
  if (typeof state === 'undefined') {
    return 0
  }
  switch (action.type) {
    case 'INCREMENT':
      return state + 1
    case 'DECREMENT':
      return state - 1
    default:
      return state
  }
}
```

preloadedState：预置的state
enhancer: enhance参数如果存在的话，则会把createStore函数作为参数传给enhancer函数，在enhancer函数执行完后会得到一个新函数，这个新函数再接受reducer, preloadedState作为参数。enhancer就是applyMiddleware函数的返回值

```javascript
// 函数已进入会先进行参数的检查，
// 首先进行参数的校验 如果全都是函数的话，则会被认为传递了多个enhancer，会被要求先compose后再传入一个单一的函数
if (
    (typeof preloadedState === 'function' && typeof enhancer === 'function') ||
    (typeof enhancer === 'function' && typeof arguments[3] === 'function')
) {
  throw new Error(
    'It looks like you are passing several store enhancers to ' +
     'createStore(). This is not supported. Instead, compose them ' +
     'together to a single function.'
  )
}
// 如果preloadedState是函数，而未传入enhancer，会将preloadedeState当作enhancer
if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
  enhancer = preloadedState
  preloadedState = undefined
}

if (typeof enhancer !== 'undefined') {
  // enhancer参数必须是函数，否则进行错误提示
  if (typeof enhancer !== 'function') {
    throw new Error('Expected the enhancer to be a function.')
  }
  // 执行enhancer函数，会得到一个增强版的createStore函数
  return enhancer(createStore)(reducer, preloadedState)
}

// 定义了几个变量
var currentReducer = reducer; // 当前的reducer
var currentState = preloadedState; // 当前的state
var currentListeners = []; // 监听函数的队列
var nextListeners = currentListeners; // 下一次监听变化后的函数队列
var isDispatching = false; // 是否正在dispatch一个action

// 确保nextListeners 与 currentListeners 不是同一个引用
function ensureCanMutateNextListeners() {
  if (nextListeners === currentListeners) {
    nextListeners = currentListeners.slice();
  }
}

// 返回上述变量currentState,
function getState() {
  return currentState
}


function subscribe(listener) {
  // 若listener 不是函数则直接抛出错误
  if (typeof listener !== 'function') {
    throw new Error('Expected the listener to be a function.');
  }
  if (isDispatching) {
    throw new Error('You may not call store.subscribe() while the reducer is executing. ' + 'If you would like to be notified after the store has been updated, subscribe from a ' + 'component and invoke store.getState() in the callback to access the latest state. ' + 'See https://redux.js.org/api-reference/store#subscribelistener for more details.');
  }
  // isSubscribed 标志
  var isSubscribed = true;

  // 调用ensureCanMutateNextListeners函数，将currentListeners 进行浅拷贝复制给nextListeners
  ensureCanMutateNextListeners();

  // 将listener放入队列尾部
  nextListeners.push(listener);

  // 返回unsubscribe函数，并在nextListeners队列中找到listener并将其删除
  return function unsubscribe() {
    if (!isSubscribed) {
      return;
    }

    if (isDispatching) {
      throw new Error('You may not unsubscribe from a store listener while the reducer is executing. ' + 'See https://redux.js.org/api-reference/store#subscribelistener for more details.');
    }

    isSubscribed = false;
    ensureCanMutateNextListeners();
    var index = nextListeners.indexOf(listener);
    nextListeners.splice(index, 1);
    currentListeners = null;
  };
}

function dispatch(action) {
  // 如果传入的action不是一个普通对象则抛出错误
  if (!isPlainObject(action)) {
    throw new Error('Actions must be plain objects. ' + 'Use custom middleware for async actions.');
  }
  // 如果action没有type属性则抛出错误
  if (typeof action.type === 'undefined') {
    throw new Error('Actions may not have an undefined "type" property. ' + 'Have you misspelled a constant?');
  }
  // 当前正在触发另一个action
  if (isDispatching) {
    throw new Error('Reducers may not dispatch actions.');
  }

  try {
    // 将isDispatching标识置为true
    isDispatching = true;
    // 执行reducer 并将得到结果赋值给currentState
    currentState = currentReducer(currentState, action);
  } finally {
    // 执行完后将isDispatching标志为置为false
    isDispatching = false;
  }
  // 在dispatch一个action后，需要将监听函数都执行一遍
  // nextListeners 赋值给currentListeners 进行遍历执行
  var listeners = currentListeners = nextListeners;
  for (var i = 0; i < listeners.length; i++) {
    var listener = listeners[i];
    listener();
  }
  // 返回action
  return action;
}

```

2.combineReducers
当应用复杂后，为了降低复杂性，需要将庞大的reducer拆分成一个个独立的单元，然后将这些一个个小单元组合来完成复杂的应用逻辑。和React的组合思想相似
```javascript
function combineReducers(reducers) {
  var reducerKeys = Object.keys(reducers);
  var finalReducers = {};

  for (var i = 0; i < reducerKeys.length; i++) {
    var key = reducerKeys[i];

    {
      if (typeof reducers[key] === 'undefined') {
        warning("No reducer provided for key \"" + key + "\"");
      }
    }
    // 判断reducer是函数时 才放到finalReducers对象中
    if (typeof reducers[key] === 'function') {
      finalReducers[key] = reducers[key];
    }
  }

  // 取出过滤后有效的keys 
  var finalReducerKeys = Object.keys(finalReducers); 

  ...

  return function combination(state, action) {
    ...

    var hasChanged = false;
    // 定义一个新的nextState
    var nextState = {};
    // 1.遍历过滤后的reducers,
    // 2.执行执行该key 对应的reducer函数，得到其对应的state
    // 3.将这个新的state, 赋值给nextState
    for (var _i = 0; _i < finalReducerKeys.length; _i++) {
      var _key = finalReducerKeys[_i];
      var reducer = finalReducers[_key];
      var previousStateForKey = state[_key];
      var nextStateForKey = reducer(previousStateForKey, action);
      if (typeof nextStateForKey === 'undefined') {
        var errorMessage = getUndefinedStateErrorMessage(_key, action);
        throw new Error(errorMessage);
      }

      nextState[_key] = nextStateForKey;
      // 判断state是否改变了，上一次的state与这次的不同即为改变了
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey;
    }
    // 判断过滤后有效的keys 是否与 state内包含的reducer的keys 长度是否一致，不一致则认为改变了
    hasChanged = hasChanged || finalReducerKeys.length !== Object.keys(state).length;
    // state改变了就返回新的nextState,否则直接返回旧的state
   return hasChanged ? nextState : state;
  };
}
```

3.bindActionCreators
```javascript
function bindActionCreators(actionCreators, dispatch) {
  // 传入的是函数则返回函数
  if (typeof actionCreators === 'function') {
    return bindActionCreator(actionCreators, dispatch);
  }
  ...
  // 传入的是对象则返回对象
  var boundActionCreators = {};

  for (var key in actionCreators) {
    var actionCreator = actionCreators[key];

    if (typeof actionCreator === 'function') {
      boundActionCreators[key] = bindActionCreator(actionCreator, dispatch);
    }
  }
  return boundActionCreators;
}

// creators
const actionCreators = {
  increase: function() {
    return (todo) {
      type: 'increase',
      todo
    }
  },
  decrease: function() {
    return (todo) {
      type: 'decrease',
      todo
    }
  }
}

// creators为对象时
let boundActions = bindActionCreators(actionCreators, store.dispatch)
boundActions.increase({
  content: '增加'
})

// creators为函数时
let boundIncrease = bindActionCreators(actionCreators.decrease, store.dispatch)
boundIncrease({
  content: '降低'
})

从上面的例子可以看出，bindActionCreators 是对调用的封装，即不用每次都dispatch
```

4.compose
```javascript
export default function compose(...funcs) {
  // 参数长度===0，则直接返回传入的这个函数
  if (funcs.length === 0) {
    return arg => arg
  }

  // 参数长度===1，则将参数中的第一个函数作为返回值
  if (funcs.length === 1) {
    return funcs[0]
  }
  // 这里的reduce就是数组的reduce方法，即从后往前执行，将上一个函数的返回值作为下一个函数的入参
  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```

5.applyMiddleware
我们先来看一个中间件
```javascript
function logger1({ getState, dispatch }) {
  return next => action => {
    console.log('logger1 next before state', getState())
    const result = next(action)
    console.log('logger1 next after state', getState())
    return result
  }
}

function logger2({ getState }) {
  return function (next){
    return function (action){
      console.log('logger2 next before state', getState())
      const returnValue = next(action)
      console.log('logger2 next after state', getState())
      return returnValue
    }
  }
}
```

```javascript

function applyMiddleware() {
  for (var _len = arguments.length, middlewares = new Array(_len), _key = 0; _key < _len; _key++) {
    middlewares[_key] = arguments[_key];
  }
  // 将createStore函数作为参数传入
  return function (createStore) {
    return function () {
      // 根据createStore执行生成一个store
      var store = createStore.apply(void 0, arguments);

      var _dispatch = function dispatch() {
        throw new Error('Dispatching while constructing your middleware is not allowed. ' + 'Other middleware would not be applied to this dispatch.');
      };
      // 向中间件注入 getState和Dispatch这两个api
      var middlewareAPI = {
        getState: store.getState,
        dispatch: function dispatch() {
          return _dispatch.apply(void 0, arguments);
        }
      };
      // 例：
      // function logger1({ getState, dispatch }) {
      // return next => action => {
      //    ...
      // }
    }
    // 执行中间件函数，在此处将上面的middlewareAPI传入，由变量chain接收其执行返回值
      var chain = middlewares.map(function (middleware) {
        return middleware(middlewareAPI);
      });
      // 在此处开始执行每一个中间件，这里用到了compose函数，从最后开始往前执行，并且将前一个函数的返回值作为下一个函数的入惨，这里将store.dispatch 作为中间件的第一个入惨，生成一个新的dispatch
      _dispatch = compose.apply(void 0, chain)(store.dispatch);
      return _objectSpread2(_objectSpread2({}, store), {}, {
        dispatch: _dispatch
      });
    };
  };
```


