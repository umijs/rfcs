- Start Date: 2019-04-25
- RFC PR:
  - [add import from umi by xiaohuoni · Pull Request #2336 · umijs/umi · GitHub](https://github.com/umijs/umi/pull/2336)
- Umi Issue: (leave this empty)

# Summary

支持在插件里添加 umi 的 export 内容，比如 `import { locale, connect, lodash } from 'umi'`，`import { Component } from 'umi/react'` 和 `import { Button } from 'umi/antd'`。

# Motivation

目前 umi 内部是通过 webpack 的 alias 打通各个依赖库的引用的，包括 `react`、`antd`、`dva` 等等，带来的问题是：

* ts 定义找不到
* eslint 的报错

原因是，比如我们 `import from react` 时，node_modules 下并没有 react。

另外，有些插件会导出额外的方法，比如 `umi-plugin-locale`，会导出自己的 `locale` 方法，而这个方法不管是放 `umi-plugin-locale` 还是 `umi-locale` 导出都不太合适。

所以，umi 需要提供一种能力，让插件可以为 `umi` 或 `umi/xxx` 注入额外的导出文件。

# Detailed design

分两部分实现：

1. 支持 `umi` 添加导出
2. 支持 `umi/xxx` 添加导出

## `umi` 添加导出

`umi/index.js` 里添加一行，

```js
export * from '@tmp/umiExports';
```

然后新增插件接口 `api.addUmiExports` 用于生成临时文件 `umiExports.js`，

```js
// export all
// 生成：export * from 'dva';
api.addUmiExports([
  {
    exportAll: true,
    source: 'dva'
  },
]);

// export 部分
// 生成：export { connect } from 'dva';
api.addUmiExports([
  {
    specifiers: ['connect'],
    source: 'dva',
  },
]);

// 支持更名
// 生成：export { default as dva } from 'dva';
api.addUmiExports([
  {
    specifiers: [{ local: 'default', exported: 'dva' }],
    source: 'dva',
  },
]);
```

`@tmp` 的别名在 `api.onStart()` 处理，解析 `jsconfig.json`（vscode）、`tsconfig.json`（ts 项目）和 `webpack.config.js`（Intellij）进行补全。（把 `@` 的别名补全也一起做了）

## `umi/xxx` 添加导出

新增插件接口 `api.addUmiFileExport`，用于启动阶段在 `umi` 下生成临时文件，

```js
// export * from '/absolute/path/to/antd';
api.addUmiFileExports([
  {
    fileName: 'antd',
    source: require.resolve('antd'),
  }
]);
```

然后在生成临时文件阶段，通过 `api.applyPlugins('addUmiFileExports', { initialValue: [] })` 拿到配置表，然后在 `umi` 包的根目录下生成相关文件。

## 冲突检测和预留

以上两个功能都需注意，

1. 数组的 fileName key 重复检测
2. 已存在文件检测
3. 通过黑名单预留一些关键文件或导出项

# How We Teach This

需要做好兼容，并通过提示引导用户使用新接口。

# Drawbacks

暂无。

# Alternatives

无。

# Unresolved questions

暂无。

# FAQ

## 为啥要有 `umi/xxx`，不统一走 `umi` + tree-shaking 实现？

原因有几点：

1. 像 react 之类的库，通常会通过 externals 进行提速
2. 像 antd 之类的库，会有自己的按需加载策略

以上场景，放 `umi` 导出实现很困难，而且没啥必要。

