# ES Module 和 CommonJS 学习笔记（二） —— NodeJS 中使用 ESM 和 CJS

## 在 NodeJS 中使用 ES6 模块

当前较新版本的 NodeJS 支持 ESM 和 CJS ，但默认使用的是 CJS 规范去解析 JS 代码，直接使用 CJS 是没有任何问题的，而使用 ESM 需要做一些处理

### `.mjs`文件

在 NodeJS 中用`.mjs`后缀的文件名表示这个文件为 ES6 模快文件，**可以在`.mjs`文件中直接使用 ESM 语法（使用`import/export`指令）**。在执行含 ES6 模块的脚本时，由于不同 NodeJS 版本的支持程度不一样，需要按照不同方式执行

```text
v16.4.0 完全支持
v13.2.0 试验性支持，可省略 --experimental-modules 
更老的版本需要显式使用参数 --experimental-modules 执行 或 甚至不支持该试验特性

node --experimental-modules <path/to/file>
```

### 修改模块规范

可以通过**修改 NodeJS 解析 JS 代码时的模块规范**，使得 ESM 可以直接在`.js`文件中使用（不修改的话直接使用会报错）。在需要修改默认解析规范的模块文件根目录下创建一个`package.json`文件（可以在要创建的目录下执行`npm init -y`生成`package.json`），并将`"type"`设置为`"module"`

```json
// ./package.json
{
  "type": "module"
}

// 当不设置 type 或设置为 "commonjs" 则采用的是 CJS 规范
```

做了以上配置之后，**如果要使用 CJS 语法（使用`module.exports/require()`），则需要在`.cjs`文件中使用**，否则会报未定义的错误

当被认为时 ES6 模块时，就会自动采用严格模式（浏览器端也一样）

## ES6 模块 和 CommonJS 模块的相互引用

ESM 是编译时加载（静态加载），有一个独立的模块依赖的解析阶段，模块输出的是接口（引用），它的加载、解析、执行都是异步的；CJS 是运行时加载（动态加载），模块输出的是值（模块对象），值会被缓存，它的加载、解析、执行都是同步的（`require()`为一个同步方法）。

由上述对两种规范的描述可见， ESM 和 CJS 规范的实现是不一样的，如果想在 ES6 模块中引入 CJS 模块或在 CJS 模块中引入 ES6 模块，则需要遵循以下规则

### CJS 模块加载 ES6 模块

由于 ES6 为异步加载，CJS为同步加载，不能直接用`require()`去加载 ES6 模块，只能**利用同样是异步加载且 CJS 和 ESM 都支持的`import()`来加载**

```js
// import() 返回一个 promise
import('....').then(res => {});

(async () => {
  await import('....');
}())
```

### ES6 模块加载 CJS 模块

**ESM 的`import`可以直接加载 CJS 模块，但只能整体加载**。因为 ES6 模块需要支持静态代码分析，而 CommonJS 模块的输出接口是`module.exports`，是一个对象，无法被静态分析，所以只能整体加载

```js
import { a } from "./foo.cjs"; // 报错

import m from "./foo.cjs";
const { a } = m; // 正确方式
```

`require()`不能在`.mjs`文件中使用（只能使用`import`或`import()`），也不能用于加载`.mjs`文件（即上述的不能加载 ES6 模块）

另外值得注意的是，**在使用 webpack 打包的项目中**，同一个文件中**不建议 ESM 和 CJS 混用**，babel 转译能让 ESM 变为 CJS，这往往可能会导致出现一些莫名其妙的错误

```js
// 不建议这么做，
import m from "./foo.js";

module.exports = {
  m
}
```

> <https://github.com/xiaoxiaojx/blog/issues/27>
> <https://www.tangshuang.net/7686.html>

## 模块（包）加载规则

默认情况下（不通过`package.json`的`exports`选项设置别名的情况下），**`import`命令**加载一个不在`node_modules`下的模块时，除非是系统模块，否则不能省略后缀名，**必须给出完整的文件路径**

```js
// ./module/index.js
import "./foo/index.js";
// 即使 ./foo/package.json 设置了入口 "main": "index.js" 也不能 import 直接引入
import "./foo";
```

但对于 CJS 没有这些限制

```js
// 可以直接 require 引入
require("./foo")
```

NodeJS 完全支持的 CJS 规范，默认情况下，它的加载规则如下

```js
require("./foo/index.js");
// 提供了详细文件路径（含文件后缀），如果找不到就直接报错

require("./foo");
/*
  省略了文件路径或后缀（有可能是 ./foo/index.js 或 ./foo.js），按以下顺序找：
  查找同名 JS 文件 foo.js；
  查找同名目录 foo 下的 index.js 文件（如果 foo 下的 package.json 指定了
  入口文件，则按照 package.json 中的配置查找）；
  找不到报错
*/

require("foo");
/*
  只提供一个字符串，按以下规则查找：
  查找同名系统模块；
  查找 node_modules 下的文件，先查找同名JS文件 foo.js，再查找同名目录 foo 下
  的 index.js 文件（如果 foo 下的 package.json 指定了入口文件，则按照
   package.json 中的配置查找）；
  找不到报错
*/
```

### 配置入口文件和路径别名

可以通过`package.json`中的`main`和`exports`配置一些方便引入模块的设置

- `package.json`中的`main`：  
  这个是指定模块入口

  ```js
  /*
    假设目录结构如下
    ---foo
      |---index.js
      |---index2.js
      |---package.json
  */

  // 假设现在不做任何配置，那么默认的入口就是 index.js
  require("./foo"); // 相当于 ./foo/index.js

  // 假设在 package.json 中添加 "main": "index2.js"，那么入口就变成 index2.js
  require("./foo"); // 相当于 ./foo/index2.js
  ```

- `package.json`中的`exports`：  
  `exports`的优先级高于`main`  

  可以用于设置文件路径别名：

  ```json
  {
    "exports": {
      "./foo": "./utils/foo",
    }
  }
  ```

  上述提及，`import`指令引入的模块路径必须是完整的，这里可以通过配置起路径别名来简略路径
  
  ```json
  {
    "exports": {
      "./foo": "./foo.js",
    }
  }
  ```

  此时就可以直接`import "./foo"`

  可以用于设置主入口 main 的路径别名：  
  用`.`表示主入口

  ```json
  {
    "exports": {
      ".": "./index.js",
    }
  }

  // 如果只有主入口配置可以略写为
  {
    "exports": "./index.js"
  }
  ```

  由于exports字段只有支持 ES6 的 Node.js 才认识，所以可以用来兼容旧版本的 Node.js

  ```json
  // 利用了 exports 优先级大于 main 的特点
  {
    "main": "./index.js",
    "exports": "./index2.js"
  }
  ```

  可以实现条件加载：\
  入口文件的别名设置`.`中，还有两个子配置`require`和`default`，分别指定 CJS 的入口和其他入口（包括 ESM）

  ```json
  {
    "exports": {
      ".": {
        "require": "./index.cjs", // cjs 入口
        "default": "./index.mjs"  // esm 入口
      }
    }
  }

  // 一样的，这里如果只有主入口配置的话可以采用简写
  {
    "exports": {
      "require": "./index.cjs", // cjs 入口
      "default": "./index.mjs"  // esm 入口
    }
  }
  ```

  这个功能需要在 Node.js 运行的时候，打开`--experimental-conditional-exports`标志

  ```bash
  node --experimental-conditional-exports ....
  ```

## 同时支持两种格式的模块

- 指定不同的入口文件
  通过上文提到的`exports`配置实现

  ```json
  {
    "exports": {
      ".": {
        "require": "./index.cjs", // cjs 入口
        "default": "./index.mjs"  // esm 入口
      }
    }
  }
  ```

- CJS 和 ESM 相互转化\
  CJS 转 ESM：

  ```js
  // 从 CJS 模块中导入模块
  import m from "./foo.cjs";

  // 提取需要的部分
  export const bar = m.bar;
  // ....
  ```

  ESM 转 CJS
  
  ```js
  // 从 ES6 模块中导入模块
  let m = null;
  (async () => {
    m = await import("./foo.mjs");
  })()

  module.exports = {
    ...m
  }
  ```

或者在子目录使用`package.json`的`"type"`配置当前模块使用的规范，让一个目录为 CJS 格式，另一个目录为 ESM 格式，但终究还是涉及到格式转换，入口文件指定等这些配置，具体可以参考一些现有的包，他们往往都支持两种格式
  
## ES6 模块注意事项

- import 命令的限制  
  除了上述提到的`import`时**路径必须是完整的（包括后缀名）**，**`import`的路径也必须是相对路径，不能为绝对路径（浏览器中可以为绝对路径）**、**只支持加载本地模块**

- 为了和浏览器的 import 加载规则相同，NodeJS 的`.mjs`文件支持 URL 路径

  ```js
  import "./foo.mjs?query=10086"
  ```

  同一个脚本只要参数不同，就会被加载多次，并且保存成不同的缓存（没有参数时或参数相同时都是加载多次只执行一次）。由于这个原因，只要文件名中含有`:`、`%`、`#`、`?`等特殊字符，最好对这些字符进行转义

- 内部变量
  为了使 ES6 模块能够不作任何修改就在服务器和浏览器端通用，NodeJS 规定了在 ES6 模块中不能使用 NodeJS 的一些内部变量

  ```text
  arguments
  module
  exports
  require
  __dirname
  __filename
  ```

文章源码：<>

参考：
><https://es6.ruanyifeng.com/#docs/module-loader>  
><https://es6.ruanyifeng.com/#docs/module>
><https://zhuanlan.zhihu.com/p/337796076>
><https://www.jianshu.com/p/1e42fcabf039>
