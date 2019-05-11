- Start Date: (2019-05-11)
- RFC PR: (leave this empty)
- Umi Issue: (leave this empty)

# Summary

区块 2.0 版本，支持重复添加区块和区块合集（典型页面模板）。

# Motivation

当前 umi 的区块（block）就是一个页面的代码片段，不支持我们把更细粒度 UI 片段提取为区块，使得可以基于 umi 研发更细粒度的区块，这样就能够通过添加多个区块来更灵活的来初始化页面了。

除了可以重复添加区块，还要提供能够把多个区块做为一个合集来快速添加，这样又能够把分散的区块整合到一起使得能够通过它们快速的初始化一个页面。

# Detailed design

### 重复添加区块

区块的代码结构不变，和现有的区块保持一致（@ 目录不再推荐使用了，局部区块的方案也不再支持它）：

```
- src // 通常我们推荐一个区块只包含 index.js 和 style.less 不再有 model 相关内容
  - index.js
  - style.less
- package.json
```

添加区块的命令行也保持不变，比如 `tnpx bigfish block add [blockpath]`（未来会支持通过 umi ui 来快速添加）。

但是具体将代码添加到项目的时候策略会修改，具体步骤如下：

#### 检测目标路径是否已经有组件存在

比如说 --path=/NewPage 时检测 `page/NewPage/index.(jsx?|tsx)` 是否存在。

#### 如果不存在则添加一个新的容器组件

如果不存在则添加 `page/NewPage/index.(js|tsx)`，通过判断是否有 tsconfig.json 来决定创建 tsx 还是 js。

该组件作为新的页面组件，并在路由中添加新的路由，逻辑和当前的区块添加逻辑保持一致。组件的内容如下：

```jsx
import React, { Component } from '@alipay/bigfish/react';

class NewPage extends Component {
  render() {
    return (
	  <Reeat.Fragment>
	  </Reeat.Fragment>
	)
  }
}
```

#### 下载区块到目标目录的子目录下

```
- page
  - NewPage
    - index.js
    - BlockName // 把区块代码下载到该目录，如果有已经有重名的则提示用户输入新的名称
      - index.js
      - style.less
```

#### 往容器组件中添加区块


```diff
import React, { Component } from '@alipay/bigfish/react';
+ import BlockName from './BlockName';

class NewPage extends Component {
  render() {
    return (
	  <Reeat.Fragment>
+		<BlockName />
	  </Reeat.Fragment>
	)
  }
}
```

### 区块合集

区块合集和一个区块一样也是一个包含 package.json 的地址，只不过它没有代码内容，只有配置。

```json
// package.json
{
  "blockConfig": {
    "blockGroup": {
      "layout": {
        "type": "antd-grid", // 默认是内置的顺序添加的 layout
        "config": [
          [{
            "span": 24,
            "block": "BlockA",
          }],
          [{
            "span": 12,
            "block": "BlockB",
          },{
            "span": 12,
            "block": "BlockB",
          }]
        ],
      },
      "blocks": {
        "BlockA": "http://a.block.url",
        "BlockB": "http://a.block.url",
      },
    }
  }
}
```

其中的 layout type 对应 `umi-block-layout-antd-grid` 的一个 npm 包，这个包需要暴露一个方法接收配置，返回对应的页面的 JS。比如 umi 可以内置一个 layout：

```js
export default (config) => {
  return `<React.Fragment>${config.map(blockName => `<${blockName} />`)}</React.Fragement>`;
}
```

### 兼容性

对于已有的区块，可以添加如下配置来保留之前整个区块直接添加到目标文件夹的方式：

```
"blockConfig": {
  "specVersion": "0.1"
}
```

也支持通过参数 `--direct=true` 来实现。

# How We Teach This

暂无

# Drawbacks

暂无

# Alternatives

暂无

# Unresolved questions

暂无
