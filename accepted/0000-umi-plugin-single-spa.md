- Start Date: 2019-05-10
- RFC PR: (leave this empty)
- Umi Issue: (leave this empty)

# Summary

基于 umi 的微前端方案。

# Motivation

微前端是有适用场景的。

一种场景是比如这篇 [DDD: Strategic Design: Core, Supporting, and Generic Subdomains](https://blog.jonathanoliver.com/ddd-strategic-design-core-supporting-and-generic-subdomains/) 的划分方式，把业务域划分成 Core、Supporting 和 Generic，出于人力和成本的考虑，除了 Core 的业务域都可能交给外包或者通过购买的方式实现，技术栈难免不一致，而这些业务域在挂载进来之后又不能影响到 Core 业务域。

一种场景是跨团队维护，比如一个系统分多个子系统，每个子系统由不同的团队维护，可能跨部门团队、外包，甚至 ISV，技术栈上可以要求一致，但发布节奏会不同，并且跨团队的沟通成本高，所以技术自治是比较好的方式，每个子应用保留自己的测试、CI、构建、部署等流程。

一种场景是比如一个系统在构建时只管开发子功能，但不清楚要多少功能，需要多少只有在最终运行时才知道，或者用户可能选择一部分功能形成一个新系统，但是加载时又不能把用户不需要的功能文件加载进去。每个子功能是一个文件，格式可以是 umd 或 amd，然后按需加载子功能的文件实现。这一类，可以归为运行时的插件系统。

那么，为啥 SPA 不行？

SPA 的特点是一个代码仓库，一次构建，一次下载（可以按需），对于上述需要外包、隔离、跨团队沟通协作、跨技术栈的场景，SPA 会让事情变得复杂，并且提升成本。

![](https://cdn.nlark.com/yuque/0/2019/png/86025/1556440486748-225de833-7f42-4474-8a19-d412251b20ac.png)

另外，像上图一样，当后端根据业务域扩张时，SPA 无法灵活地扩张，仍然是一个仓库、一次构建，。。。。

# Detailed design

首先是基于 umi，分主应用和子应用，主应用也基于 umi，分别挂载不同的 umi 插件实现打包。

[GitHub - umijs/umi-plugin-single-spa: Umi plugin for single-spa.](https://github.com/umijs/umi-plugin-single-spa)

## 使用

主应用配 `umi-plugin-single-spa/master` 插件，

```js
export default {
  plugins: [
    ['umi-plugin-single-spa/master', {
      // 注册子应用信息
      apps: [
        {
          name: 'app1',
          // 支持 config entry
          entry: {
            scripts: [],
            styles: [],
          },
        },
        {
          name: 'app2',
          // 支持 html entry
          entry: {
            html: '/path/to/app/index.html',
          },
        },
      ],
    }],
  ],
}
```

子应用配 `umi-plugin-single-spa/slave` 插件，

```js
export default {
  plugins: [
    ['umi-plugin-single-spa/slave', {
    }],
  ],
}
```

## 关键技术点

然后，还有一些关键的技术点，我画了张图，

<img src="https://cdn.nlark.com/yuque/0/2019/png/86025/1557467014355-7462bd1f-c57c-4568-98fd-04e66e927f83.png" width="500" />

实现掉前三个，基于 single-spa 就可以跑起来了，其他是增量可选功能。

### Config Entry

以配置的方式注册子应用信息，通过 `apps` 配置项露出，包括 name、routerPrefix、scripts 和 styles，scripts 和 styles 同时支持外链和内联。

Config Entry 是相对于 HTML Entry 而言的，HTML Entry 的方式是在运行时解析 HTML 拿到 Config 信息，而 Config Entry 则是在编译时拿到，少了运行时解析的一步，但会损失一些开发者的便利性。

比如：

```js
{
  apps: [
    {
      name: 'app1',
      routerPrefix: '/app1',
      entry: {
        scripts: [
          // 外链
          {
            src: '/path/to/a.js',
            isEntry: true,
          },
          // 外链的简写
          '/path/to/a.js',
          // 内联
          {
            content: 'alert(\'app1\');',
          },
        ],
        styles: [
          // 外链
          {
            href: '/path/to/a.css',
          },
          // 外链的简写
          '/path/to/a.css',
          // 内联
          {
            content: 'body { color: red; }',
          },
        ],
      },
    },
  ]
}
```

以下是子 app 配置项，

#### name

唯一 key，不允许重复。

#### routerPrefix

路由匹配时激活该子应用。

#### entry

#### entry.scripts

script 的属性有：

- src
- content
- isEntry，是否为入口

没有显式的 isEntry 标记，则最后一个 script 为 entry。

#### entry.styles

### 按需加载

根据 Config Entry 的配置，在激活页面时加载相应的 JS 和 CSS。（如果支持 HTML Entry，需先加载 HTML 解析拿到 Config Entry）

实现上先基于 [import-html-entry](https://github.com/kuitos/import-html-entry) 的方案来做，fetch 拿到内容，`eval` 执行。（后续看看还有没有更好的方案，比如 systemjs + amd 的方式）

基本流程：

1. 获取外链的 JS 和 CSS 内容，把 `{ src }` 或 `{ href }` 转化为 `{ content }`
2. 执行 JS 和挂载 CSS

获取外链内容需要做缓存，可能多个子应用外链了同一个 url 的文件。（进而还可以做本地持久化，因为 cdn 上的文件是不可覆盖的，不存在更新问题）

由于 JS 执行需要拿到 entry 文件 export 的生命周期方法，所以需要特殊处理。事先记录 window 对象属性，事后比对 window 对象属性，最后新增的 window 对象属性即 entry 文件 export 的内容。

挂载 CSS 比较简单，在 `<head>` 标签里新增 `<style>` 实现。

### 父子应用通讯

> 子应用如何调父应用方法，以及父应用如何下发状态。

主应用里约定 `src/rootExports.js` 文件的内容会通过 `window.g_rootExports` 露出。

在 mount 子应用时传入，子应用通过 Context 露出。

#### React 组件

子应用通过 `api.addUmiExports` export `useRootContext` 方法，

```js
import { useRootContext } from 'umi';
const rootContext = useRootContext();
```

#### dva 的 state 获取

子应用通过 `api.addUmiExports` export `getRootState` 和 `dispatchRootAction` 方法，具体实现来自主应用的 `window.g_rootExports`，

使用方式如下：

```js
import { getRootState, dispatchRootAction } from 'umi';

const state = getRootState();
dispatchRootAction({
  type: 'global/showErrorMessage',
  payload: {},
});
```

#### 其他场景

直接用 `window.g_rootExports`。

### 公共依赖加载

> 大部分子应用都用到的资源怎么处理？

#### 基本思路

* 子应用做 external（可以用 auto-external 插件），把 react、antd 等提出来，cdn 模式，有一定规则
* cdn 需 http/2
* 主应用存 map 表，请求资源时做统一化处理，比如 antd@3.4.1 和 antd@3.3.4，都换成 antd@3.4.1

#### 具体实现

主应用提供 map 表配置，写入临时文件 `singleSpaScriptsMap.js`，应用中可通过 `@tmp/singleSpaScriptsMap` 取到，

```js
{
  "scriptsMap": {
    "antd": {
      "3": "3.17.0"
    },
    "react: {
      "16": "16.8.6"
    }
  }
}
```

在按需加载时，请求前先做一个 unify 的处理，

比如 antd，

```
https://gw.alipayobjects.com/os/lib/antd/3.13.0/dist/antd.js
↓ ↓ ↓ ↓ ↓ ↓
https://gw.alipayobjects.com/os/lib/antd/3.17.0/dist/antd.js

https://gw.alipayobjects.com/os/lib/antd/3.13.0/dist/antd.css
↓ ↓ ↓ ↓ ↓ ↓
https://gw.alipayobjects.com/os/lib/antd/3.17.0/dist/antd.css
```

比如 react，

```
https://gw.alipayobjects.com/os/lib/react/16.3.0/umd/react.production.min.js
↓ ↓ ↓ ↓ ↓ ↓
https://gw.alipayobjects.com/os/lib/react/16.8.6/umd/react.production.min.js
```

这样，多个子应用请求的 antd 和 react 就有缓存了。

#### 备选方案

* 子应用 external + 主应用 script 加载资源（有更新问题）
* amd + systemjs

### HTML Entry

> Config Entry 的进阶版，简化开发者使用，但把解析消耗留给了用户。

### JS 沙箱

> 子应用之间互不影响，包括全局变量、事件等处理。

### CSS 重载

> 子应用之间样式互不影响，切换时做卸载和装载。

### 预加载

> 网络空闲时加载子应用资源，最好有用户行为数据支持。

### 子应用并行

> 多个微前端如何同时存在，进阶用法。

### 子应用嵌套

> 微前端如何嵌微前端，进阶用法。

### 子应用调试

# How We Teach This

暂无。

# Drawbacks

暂无。

# Alternatives

TODO

# Unresolved questions

暂无。
