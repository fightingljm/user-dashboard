# YES！It's dva！

[原文地址](https://github.com/sorrycc/blog/issues/18)

本文会一步步引导大家如何创建一个 CURD 应用，包含查询、编辑、删除、创建，以及分页处理，数据 mock，自动处理 loading 状态等，基于 react, [dva](https://github.com/dvajs/dva) 和 [antd](https://github.com/ant-design/ant-design) 。

最终效果：



开始之前：

- 确保 node 版本是 6.5 +
- 用 [cnpm](https://github.com/cnpm/cnpm) 或 [yarn](https://github.com/yarnpkg/yarn) 能节约你安装依赖的时间

**Step 1. 安装 dva-cli 并创建应用**

先安装 dva-cli，并确保版本是 0.7.x。

```bash
$ npm i dva-cli@0.7 -g
$ dva -v
0.7.0
```

然后创建应用：

```bash
$ dva new user-dashboard
$ cd user-dashboard
```

**Step 2. 配置 [antd](https://github.com/ant-design/ant-design) 和 [babel-plugin-import](https://github.com/ant-design/babel-plugin-import)**

babel-plugin-import 用于按需引入 antd 的 JavaScript 和 CSS，这样打包出来的文件不至于太大。

```bash
$ npm i antd --save
$ npm i babel-plugin-import --save-dev
```

修改 `.roadhogrc`，在 `"extraBabelPlugins"` 里加上：

```
["import", { "libraryName": "antd", "style": "css" }]
```

**Step 3. 配置代理，能通过 RESTFul 的方式访问 [http://localhost:8000/api/users](http://localhost:8000/api/users)**

修改 `.roadhogrc`，加上 `"proxy"` 配置：

```
"proxy": {
  "/api": {
    "target": "http://jsonplaceholder.typicode.com/",
    "changeOrigin": true,
    "pathRewrite": { "^/api" : "" }
  }
},
```

然后启动应用：(这个命令一直开着，后面不需要重启)

```bash
$ npm start
```

浏览器会自动开启，并打开 [http://localhost:8000](http://localhost:8000) 。

访问 [http://localhost:8000/api/users](http://localhost:8000/api/users) ，就能访问到 [http://jsonplaceholder.typicode.com/users](http://jsonplaceholder.typicode.com/users) 的数据。(由于 typicode.com 服务的稳定性，偶尔可能会失败。不过没关系，正好便于我们之后对于出错的处理)



**Step 4. 生成 users 路由**

用 dva-cli 生成路由：

```bash
$ dva g route users
```
然后访问 [http://localhost:8000/#/users](http://localhost:8000/#/users) 。

**Step 5. 构造 users model 和 service**

用 dva-cli 生成 Model ：

```bash
$ dva g model users
```

修改 `src/models/users.js` ：

```js
import * as usersService from '../services/users';

export default {
  namespace: 'users',
  state: {
    list: [],
    total: null,
  },
  reducers: {
    save(state, { payload: { data: list, total } }) {
      return { ...state, list, total };
    },
  },
  effects: {
    *fetch({ payload: { page } }, { call, put }) {
      const { data, headers } = yield call(usersService.fetch, { page });
      yield put({ type: 'save', payload: { data, total: headers['x-total-count'] } });
    },
  },
  subscriptions: {
    setup({ dispatch, history }) {
      return history.listen(({ pathname, query }) => {
        if (pathname === '/users') {
          dispatch({ type: 'fetch', payload: query });
        }
      });
    },
  },
};
```

新增 `src/services/users.js`：

```js
import request from '../utils/request';

export function fetch({ page = 1 }) {
  return request(`/api/users?_page=${page}&_limit=5`);
}
```

由于我们需要从 response headers 中获取 total users 数量，所以需要改造下 `src/utils/request.js`：

```js
import fetch from 'dva/fetch';

function checkStatus(response) {
  if (response.status >= 200 && response.status < 300) {
    return response;
  }

  const error = new Error(response.statusText);
  error.response = response;
  throw error;
}

/**
 * Requests a URL, returning a promise.
 *
 * @param  {string} url       The URL we want to request
 * @param  {object} [options] The options we want to pass to "fetch"
 * @return {object}           An object containing either "data" or "err"
 */
export default async function request(url, options) {
  const response = await fetch(url, options);

  checkStatus(response);

  const data = await response.json();

  const ret = {
    data,
    headers: {},
  };

  if (response.headers.get('x-total-count')) {
    ret.headers['x-total-count'] = response.headers.get('x-total-count');
  }

  return ret;
}
```

切换到浏览器（会自动刷新），应该没任何变化，因为数据虽然好了，但并没有视图与之关联。
但是打开 Redux 开发者工具，应该可以看到 `users/fetch` 和 `users/save` 的 action 以及相关的 state 。



**Step 6. 添加界面，让用户列表展现出来**

用 dva-cli 生成 component：

```bash
$ dva g component Users/Users
```

然后修改生成出来的 `src/components/Users/Users.js` 和 `src/components/Users/Users.css`，
并在 `src/routes/Users.js` 中引用他。具体参考这个 [Commit](https://github.com/fightingljm/user-dashboard/commit/3dc1b698e66dae46b9de952ae578a1bed134b920)。

需留意两件事：

- 对 model 进行了微调，加入了 page 表示当前页
- 由于 components 和 services 中都用到了 pageSize，所以提取到 `src/constants.js`

改完后，切换到浏览器，应该能看到带分页的用户列表。


**Step 7. 添加 layout**

添加 layout 布局，使得我们可以在首页和用户列表页之间来回切换。

- 添加布局，src/components/MainLayout/MainLayout.js 和 CSS 文件
- 在 src/routes 文件夹下的文件中引用这个布局

参考这个 [Commit](https://github.com/fightingljm/user-dashboard/commit/9a1b9f015ce9c48ebf9ca81691f8e1b0e625638d)。

> 注意：
页头的菜单会随着页面切换变化，高亮显示当前页所在的菜单项

**Step 8. 通过 [dva-loading](https://github.com/dvajs/dva/tree/master/packages/dva-loading) 处理 loading 状态**

dva 有一个管理 effects 执行的 hook，并基于此封装了 dva-loading 插件。通过这个插件，我们可以不必一遍遍地写 showLoading 和 hideLoading，当发起请求时，插件会自动设置数据里的 loading 状态为 true 或 false 。然后我们在渲染 components 时绑定并根据这个数据进行渲染。

先安装 dva-loading ：

```bash
$ npm i dva-loading --save
```

修改 `src/index.js` 加载插件，在合适的地方加入下面两句：

```js
+ import createLoading from 'dva-loading';
+ app.use(createLoading());
```

然后在 `src/components/Users/Users.js` 里绑定 loading 数据：

```js
+ loading: state.loading.models.users,
```

具体参考这个 [Commit](https://github.com/fightingljm/user-dashboard/commit/a104aa6eab80eb5e77c2065d0b3c5ecaa40904e6) 。

切换到浏览器，你的用户列表有 loading 了没?

**Step 9. 处理分页**

只改一个文件 `src/components/Users/Users.js` 就好。

处理分页有两个思路：

- 发 action，请求新的分页数据，保存到 model，然后自动更新页面
- 切换路由 (由于之前监听了路由变化，所以后续的事情会自动处理)

我们用的是思路 2 的方式，好处是用户可以直接访问到 page 2 或其他页面。

参考这个 [Commit](https://github.com/fightingljm/user-dashboard/commit/586b3757346bb5f90b89170afe9d032736a830d7) 。

**Step 10. 处理用户删除**

经过前面的 9 步，应用的整体脉络已经清晰，相信大家已经对整体流程也有了一定了解。

后面的功能调整基本都可以按照以下三步进行：

- service
- model
- component

我们现在开始增加用户删除功能。

1. service, 修改 `src/services/users.js`：

```js
export function remove(id) {
  return request(`/api/users/${id}`, {
    method: 'DELETE',
  });
}
```

2. model, 修改 `src/models/users.js`：

```js
*remove({ payload: id }, { call, put, select }) {
  yield call(usersService.remove, id);
  const page = yield select(state => state.users.page);
  yield put({ type: 'fetch', payload: { page } });
},
```

3. component, 修改 `src/components/Users/Users.js`，替换 `deleteHandler` 内容：

```js
dispatch({
  type: 'users/remove',
  payload: id,
});
```

切换到浏览器，删除功能应该已经生效。

**Step 11. 处理用户编辑**

处理用户编辑和前面的一样，遵循三步走：

- service
- model
- component

先是 service，修改 `src/services/users.js`：

```js
export function patch(id, values) {
  return request(`/api/users/${id}`, {
    method: 'PATCH',
    body: JSON.stringify(values),
  });
}
```

再是 model，修改 `src/models/users.js`：

```js
*patch({ payload: { id, values } }, { call, put, select }) {
  yield call(usersService.patch, id, values);
  const page = yield select(state => state.users.page);
  yield put({ type: 'fetch', payload: { page } });
},
```

最后是 component，详见 [Commit](https://github.com/fightingljm/user-dashboard/commit/d4d88b500c89ec4fac446ef3fd89e3aa557c9206)。

需要注意的一点是，我们在这里如何处理 Modal 的 visible 状态，有几种选择：

- 存 dva 的 model state 里
- 存 component state 里

另外，怎么存也是个问题，可以：

- 只有一个 visible，然后根据用户点选的 user 填不同的表单数据
- 几个 user 几个 visible

此教程选的方案是 2-2，
即存 component state，并且 visible 按 user 存。
另外为了使用的简便，封装了一个 `UserModal` 的组件。

完成后，切换到浏览器，应该就能对用户进行编辑了。

**Step 12. 处理用户创建**

相比用户编辑，用户创建更简单些，因为可以共用 `UserModal` 组件。
和 Step 11 比较类似，就不累述了，详见 [Commit](https://github.com/fightingljm/user-dashboard/commit/4e4cd3275ecbb94d33a4e6357a78b498e9147158) 。

---

到这里，我们已经完成了一个完整的 CURD 应用。但仅仅是完成，并不完善，比如：

- 如何处理错误，比如请求等
- 如何处理请求超时
- 如何根据路由动态加载 JS 和 CSS
- ...

请期待下一篇。

(完)
