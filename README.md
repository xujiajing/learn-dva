[TOC]


## 为什么需要dva？
 Redux的架构虽然已经比较成熟完善，但是仍然避免不了在项目中会暴露出一些问题。
 #### 1.文件切换问题
   redux 的项目通常要分 reducer , action , saga , component 等等，我们需要在这些文件之间来回切换。并且这些文件通常是分目录存放的：
 ```
 + src
  + sagas
    - user.js
  + reducers
    - user.js
  + actions
    - user.js
 ```
 所以通常我们需要在这三个 user.js 中来回切换。
 
 #### 2.saga 创建麻烦
  我们在 saga 里监听一个 action 通常需要这样写：
```
function *userCreate() {
  try {
    // Your logic here
  } catch(e) {}
}
function *userCreateWatcher() {
  takeEvery('user/create', userCreate);
}
function *rootSaga() {
  yield fork(userCreateWatcher);
}
```
 对于 redux-saga 来说，这样设计可以让实现更灵活，但对于我们的项目而言，大部分场景只需要用到 takeEvery 和 takeLatest 就足够，每个 action 的监听都需要这么写就显得非常冗余。
#### 3.entry 创建麻烦
除了 redux store 的创建，中间件的配置，路由的初始化，Provider的store的绑定，saga的初始化，还要处理 reducer, component, saga 的 HMR 。

## 什么是dva？
dva 是基于现有应用架构 ( `redux + react-router + redux-saga` 等 )的一层轻量封装，他最核心的是提供了 `app.model` 方法，**用于把 reducer ,  initialState , action , saga 封装到一起**。
```
import { fetchUsers } from '../services/user';

export default {
  namespace: 'user',
  state: {
    list: [],
  },
  reducers: {
    save(state, action) {
      return {
        ...state,
        list: action.data,
      };
    },
  },
  effects: {
    *fetch(action, { put, call }) {
      const users = yield put(fetchUsers, action.data);
      yield put({ type: 'save', data: users });
    },
  },
  subscriptions: {
    setup({ dispatch, history }) {
      return history.listen(({ pathname }) => {
        if (pathname === '/user') {
          dispatch({ type: 'fetch' });
        }
      });
    },
  },
}
```
在有 dva 之前，我们通常会创建 sagas/products.js, reducers/products.js 和actions/products.js，然后在这些文件之间来回切换。

##### 6个API
- app = dva(Opts)
- app.use(Hooks)
- app.models(ModelObject)
- app.unmodel(Namespace)
- app.router(Function)
- app.start([HTMLElement])

##### 8个概念
- State
- Action
- Model
- Reducer
- Effect
- Subscription
- Router
- RouteComponent


## 核心概念
### 数据流向
<img src="https://zos.alipayobjects.com/rmsportal/PPrerEAKbIoDZYr.png" width=700 />

数据的改变发生通常是通过:  
- 用户交互行为（用户点击按钮等）
- 浏览器行为（如路由跳转等）触发的

当此类行为会改变数据的时候，可以通过 dispatch 发起一个 action

- 如果是**同步行为**，会将action发送给 Reducer，直接通过 Reducer 改变 State，然后通过 connect 重新渲染组件。
- 如果是**异步行为**，会将action发送给 Effect，一般是从服务器请求数据，服务器返回数据之后，Effect 会发送相应的 action 给 reducer，由**唯一能改变**state 的 reducer 改变 State ，然后通过connect重新渲染组件。

### Action
Action：表示操作事件，可以是同步，也可以是异步。它需要有一个 type ，表示这个 `action` 要触发什么操作；`payload` 则表示这个 action 将要传递的数据。

```
dispatch({ type: 'todos/add', payload: 'Learn Dva' });
```
```
function addTodo(text) {
  return {
    type: ADD_TODO,
    text
  }
}
dispatch(addTodo())
```


### Model
Model 是 dva 最重要的部分，可以理解为 redux、react-redux、redux-saga 的封装。 每个独立的route都对应一个model, 每个model包含如下属性:

- `namespace`：模型的命名空间，这个是必须的，而且在同一个应用中每个模型的该属性是唯一的。整个应用的 State，由多个小的 Model 的 State 以 namespace 为 key 合成。
```
namespace: 'user'
```
- `state`：与具体route相关的所有状态数据结构存放在该属性中。
```
 state: {
    list: [],
 }
```
- `subscriptions`：**用于订阅一个数据源，然后根据条件 dispatch 需要的 action**。 比如当pathname和给定的名称匹配的时候，执行什么操作之类的设置。
```
subscriptions: {
    setup({ dispatch, history }) {
      return history.listen(({ pathname }) => {
        if (pathname === '/user') {
          dispatch({ type: 'fetch' });
        }
      });
    },
}
```
- `effects`：**用于处理异步操作和业务逻辑，不直接修改 state**。简单的来说，就是获取从服务端获取数据，并且发起一个 action 交给 reducer 的地方。这是基于 redux-saga 实现的，语法为 generator。Generator 返回的是迭代器，通过 yield 关键字实现暂停功能。
```
effects: {
    *fetch(action, { put, call }) {
      const users = yield put(fetchUsers, action.data);
      yield put({ type: 'save', data: users });
    },
}
```
- `reducers`：**是唯一可以更新 state 的地方**。当数据需要从服务器获取时，需要发起异步请求，请求到数据之后，通过调用 Reducers更新数据到全局state。reducer 是 pure function，他接收参数 state 和 action，返回新的 state，即 (state, action) => newState。
```
reducers: {
    save(state, action) {
      return {
        ...state,
        list: action.data,
      };
    },
}
```

除了上面的几个属性外，需要另外注意几个方法的使用:
- **select**：从state中查找所需的子state属性。该方法参数为state, 返回一个子state对象。
- **put**：创建一条effect信息, 指示middleware发起一个action到Store。 `put({type: ‘xxxx’, payload: {}})`
- **call**：创建一条effect信息，指示middleware使用args作为fn的参数执行，例如call(services.create, payload)

基本的model结构如下:
```
export default {
  namespace: 'users',
  state: {},
  subscriptions: {},
  effects: {},
  reducers: {}
}
```

### RouteComponent
`RouteComponent` 表示 Router 里匹配路径的 Component，通常会绑定 model 的数据。
- `Container Component`对应于每个独立的route页面。每个容器组件都维护一个相关的state, 所有的state改变都由容器最终执行。容器组件**负责向其子组件(呈现组件)分配属性(props)**。

- `Presentational Component`是独立的纯粹的，例如ant.design UI组件的react实现，每个组件跟业务数据并没有耦合关系，只是完成自己独立的任务，**需要的数据通过 props 传递进来，需要操作的行为通过接口暴露出去**。 

所以在 dva 中，通常需要 connect Model的组件都是 Route Components，组织在/routes/目录下，而/components/目录下则是纯组件（Presentational Components）。

## 项目结构
```
├── mock    // mock数据文件夹
├── node_modules // 第三方的依赖
├── public  // 一般用于存放静态文件，打包时会被直接复制到输出目录(./dist)
├── src  // 用于存放项目源代码
│   ├── assets // 用于存放静态资源，打包时会经过 webpack 处理
│   ├── components // 用于存放 React 组件，一般是该项目公用的无状态组件
│   ├── models // dva最重要的文件夹，所有的数据交互及逻辑都写在这里
│   ├── routes //  用于存放需要 connect model 的路由组件
│   ├── services // 用于存放服务文件，一般是网络请求等；
│   ├── utils // 工具类库
│   ├── index.css // 入口文件样式
│   ├── index.js // 入口文件
│   └── router.js // 项目的路由文件
├── .eslintrc // bower安装目录的配置
├── .editorconfig // 保证代码在不同编辑器可视化的工具
├── .gitignore // git上传时忽略的文件
├── .roadhogrc.mock.js // 项目的配置文件
└── package.json // 当前整一个项目的依赖
  

```

## Dva VS  Redux
#### 使用Redux
- antion.js文件
```
export const REQUEST_TODO = 'REQUEST_TODO';
export const RESPONSE_TODO = 'RESPONSE_TODO';
const request = count => ({type: REQUEST_TODO, payload: {loading: true, count}});
const response = count => ({type: RESPONSE_TODO, payload: {loading: false, count}});

export const fetch = count => {
  return (dispatch) => {
    dispatch(request(count));

    return new Promise(resolve => {
      setTimeout(() => {
        resolve(count + 1);
      }, 1000)
    }).then(data => {
      dispatch(response(data))
    })
  }
}
```
- reducer.js 文件
```
import { REQUEST_TODO, RESPONSE_TODO } from './actions';

export default (state = {
  loading: false,
  count: 0
}, action) => {
  switch (action.type) {
    case REQUEST_TODO:
      return {...state, ...action.payload};
    case RESPONSE_TODO:
      return {...state, ...action.payload};
    default:
      return state;
  }
}
```
- app.js 文件
```
import React from 'react';
import { bindActionCreators } from 'redux';
import { connect } from 'react-redux';
import * as actions from './actions';

const App = ({fetch, count, loading}) => {
  return (
    <div>
      {loading ? <div>loading...</div> : <div>{count}</div>}
      <button onClick={() => fetch(count)}>add</button>
    </div>
  )
}

function mapStateToProps(state) {
  return state;
}

function mapDispatchToProps(dispatch) {
  return bindActionCreators(actions, dispatch)
}

export default connect(mapStateToProps, mapDispatchToProps)(App)
```
- index.js 文件
```
import { render } from 'react-dom';
import { createStore, applyMiddleware } from 'redux';
import { Provider } from 'react-redux'
import thunkMiddleware from 'redux-thunk';

import reducer from './app/reducer';
import App from './app/app';

const store = createStore(reducer, applyMiddleware(thunkMiddleware));

render(
  <Provider store={store}>
    <App/>
  </Provider>
  ,
  document.getElementById('app')
)
```
#### 使用dva
- model.js 文件
```
export default {
  namespace: 'demo',
  state: {
    loading: false,
    count: 0
  },
  reducers: {
    request(state, payload) {
      return {...state, ...payload};
    },
    response(state, payload) {
      return {...state, ...payload};
    }
  },
  effects: {
    *'fetch'(action, {put, call}) {
      yield put({type: 'request', loading: true});

      let count = yield call((count) => {
        return new Promise(resolve => {
          setTimeout(() => {
            resolve(count + 1);
          }, 1000);
        });
      }, action.count);

      yield put({
        type: 'response',
        loading: false,
        count
      });
    }
  }
}
```
- app.js 文件
```
import React from 'react'
import { connect } from 'dva';

const App = ({fetch, count, loading}) => {
  return (
    <div>
      {loading ? <div>loading...</div> : <div>{count}</div>}
      <button onClick={() => fetch(count)}>add</button>
    </div>
  )
}

function mapStateToProps(state) {
  return state.demo;
}

function mapDispatchToProps(dispatch) {
  return {
    fetch(count){
      dispatch({type: 'demo/fetch', count});
    }
  }
}

export default connect(mapStateToProps, mapDispatchToProps)(App)
```
- index.js 文件
```
import dva from 'dva';
import model from './model';
import App from './app';

const app = dva();

app.use({});

app.model(model);

app.router(() => <App />);

app.start();
```
通过上面两种不同方式来实现一个异步的计数器的代码结构发现：
1. 使用 redux 需要拆分出`action`模块和`reducer`模块
2. dva将`action`和`reducer`封装到`model`中，异步流程采用Generator处理

## 配合Umi使用
dva 项目通常都是这种扁平的组织方式:
```
+ models
  - global.js
  - a1.js
  - a2.js
  - b.js
+ services
  - a.js
  - b.js
+ routes
  - PageA.js
  - PageB.js
```
用了 umi 后，可以按页面维度进行组织:
```
+ models/global.js
+ pages
  + a
    - index.js
    + models
      - a1.js
      - a2.js
    + services
      - a.js
  + b
    - index.js
    - model.js
    - service.js
```
好处是更加结构更加清晰了，减少耦合，一删全删，方便 copy 和共享。另外，配合 umi 使用后降低为 0 API。

## 参考链接
- [github/dva](https://github.com/dvajs/dva/blob/master/README_zh-CN.md) 

- [DvaJS官方文档](https://dvajs.com/)  
- [Redux-Saga中文文档](https://redux-saga-in-chinese.js.org/)  
- [UmiJS官方文档](https://umijs.org/zh/)  
- [dva + umi + antd 项目实战](https://github.com/zuiidea/antd-admin)  
- [前端组件化](http://huziketang.mangojuice.top/books/react/lesson2)  
