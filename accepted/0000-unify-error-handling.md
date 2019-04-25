- Start Date: 2019-04-25
- RFC PR: (leave this empty)
- Umi Issue: (leave this empty)

# Summary

统一做出错处理，覆盖编译时的 webpack、插件、配置、路由等场景，功能上包含整理 errorCode、处理多语言、出错时给适当的建议等等。

# Motivation

没有统一的出错处理，导致的问题有：

- 可能的遗漏
- 错误风格不一致
- 多语言实现复杂
- 出错时不能给出很好的建议（尤其时由于三方依赖出错，目前用户端不能及时收到我们给出的修复建议，进而导致大量的咨询）
- 等

为解决这些问题，所以要做出错的统一处理。

此外，统一整理 errorCode 还有利于做出错统计和多平台的信息共享。

# Detailed design

## 如何抛错

通过 umi-utils 提供 UmiError 类，扩展 Error，允许传入 `code`、`data`、`message` 和 `tips` 配置项。

抛错的方式如下：

```js
// 只给 code，message 和 tips 如果没有，会从 ERROR_MAP 里找
throw new UmiError({
  code: 10001,
});

// message 如果是 template，可接收变量传入
throw new UmiError({
  code: 10001,
  context: {
    configKey: 'chainWebpack',
  },
});

// 可自带 message
throw new UmiError({
  code: 10001,
  message: 'Failed to minify the bundle.',
});

// 可自带 tips，格式为数组
throw new UmiError({
  code: 10001,
  tips: ['Steps:', '1. Install deps', '2. Umi dev'],
});

// tips 也可以是字符串
throw new UmiError({
  code: 10001,
  tips: 'UglifyJS 问题请参考文章解决 https://github.com/sorrycc/blog/issues/83',
});

// 传入额外的 tips
throw new UmiError({
  code: 10001,
  extraTips: ['还可以尝试删除 node_modules 重装，说不定就好了。'],
});

// 只传 message，会通过从 ERROR_MAP 里过滤有 test() 检测函数的进行检测，猜出 errorCode
throw new UmiError({
  message: 'Failed to minify the bundle.',
});

// 既没 code，又没 message 的直接抛 UmiError 错误
throw new UmiError({
  tips: [],
});
```

umi 里可以直接引 `import { UmiError } from umi-utils` 使用，插件里通过 `api.UmiError` 露出。

## ERROR_MAP 表

通过单独的仓库 `umijs/umi-error-map` 维护，umi 里通过 `^` 前缀声明依赖，便于快速更新。

格式如下：

```js
{
  (errorCode: string): {
    message?: string,
    test?: function,
    tips?: string[] | string,
    // Default context
    context?: object,
  }
}
```

示例：

```json
{
  10001: {
    message: 'Config failed',
  },
  10002: {
    message: 'Config item <%= item %> invalid',
    tips: [],
  },
  10003: {
    test(error) {
      return e.message.includes('UglifyJS'),
    },
    tips: `UglifyJS 问题请参考文章解决 https://github.com/sorrycc/blog/issues/83`,
  },
}
```

### 分类

errorCode 为 5 位数，然后通过第一位数字分类，

* `1` ，umi-core 错误
* `2`，webpack 错误
* `3-8`，预留
* `9`，上层框架错误，比如 Bigfish 的抛错

### 扩展

允许上层框架扩展 ERROR_MAP，比如 Bigfish；不允许插件层扩展 ERROR_MAP，因为没有必要，插件里直接自己抛 tips、message 等就好了。

通过环境变量进行扩展，比如：

```bash
process.env.EXTRA_ERROR_MAP = require.resolve('bigfish-error-map');
```

## 统一的错误输出形式

在 `umi-utils` 里提供 `printUmiError` 方法，统一处理错误输出，用 `console.error`，打在 stderr 上。

```js
Error: Failed to minify the bundle. Error: 0.0f3f4c41.async.js from UglifyJs
Error Code: 10001
Tips: 
    UglifyJS 问题请参考文章解决 https://github.com/sorrycc/blog/issues/83
Stack:
    at foo (/private/tmp/sorrycc-SIqNNX/a.js:19:9)
    at Object.<anonymous> (/private/tmp/sorrycc-SIqNNX/a.js:35:3)
    at Module._compile (internal/modules/cjs/loader.js:688:30)
    at Object.Module._extensions..js (internal/modules/cjs/loader.js:699:10)
    at Module.load (internal/modules/cjs/loader.js:598:32)
    at tryModuleLoad (internal/modules/cjs/loader.js:537:12)
    at Function.Module._load (internal/modules/cjs/loader.js:529:3)
    at Function.Module.runMain (internal/modules/cjs/loader.js:741:12)
    at startup (internal/bootstrap/node.js:285:19)
    at bootstrapNodeJSCore (internal/bootstrap/node.js:739:3)
```

上色版，

![](https://cdn.nlark.com/yuque/0/2019/png/86025/1556187981500-1be51415-c892-4eea-a320-b8e6131d1bf5.png)

# How We Teach This

不需要。

# Drawbacks

没有。

# Alternatives

无。

# Unresolved questions

暂无。

# Ref

* https://github.com/umijs/umi/issues/1813
