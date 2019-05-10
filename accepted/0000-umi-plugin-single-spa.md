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

主应用配 `umi-plugin-single-spa/master`，

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

#### entry.styles

### 按需加载

根据 Config Entry 的配置，在激活页面时加载响应的 JS 和 CSS。（如果支持 HTML Entry，需先加载 HTML 解析拿到 Config Entry）



### 父子应用通讯

> 子应用如何调父应用方法，以及父应用如何下发状态。

分基于 dva 和不基于 dva 两种场景，基于 dva 可以让父子应用的 action 串起来。

### 公共依赖加载

> 大部分子应用都用到的资源怎么处理？

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

# How We Teach This

暂无。

# Drawbacks

暂无。

# Alternatives

TODO

# Unresolved questions

暂无。