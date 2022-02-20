# ES Module 和 CommonJS 学习笔记（一）

注：下文 esm 指 ECMAScript Module ，即 ES6 的模块语法（import/export），cjs 指 CommonJS （module.exports/require）

## 浏览器端的ESM模块加载

浏览器中使用 esm 模块语法 import/export 或加载 ES6 模块是通过 script 标签实现时，**必须加上`type="module"`**，从而浏览器会知道这是一个ES6模块。  

浏览器对于带有`type="module"`的`<script>`都是异步加载，不会造成堵塞浏览器，即等到整个页面渲染完，再执行模块脚本，等同于打开了`<script>`标签的defer属性（可以使用`async`属性）

## esm 和 cjs 的差异

**esm 是编译时加载（静态加载），有一个独立的模块依赖的解析阶段，模块输出的是接口（引用），import命令是异步加载的**；\
**cjs 是运行时加载（动态加载），模块输出的是值（模块对象），值会被缓存，require 为同步加载**  

esm 输出的是接口，是对模块内值的一个只读引用，因此不同脚本加载模块时得到的接口指向的都是同一个实例。根据输出的接口去取值，所以模块内部值的改变会被不同的脚本读取到  

cjs 输出的是值的拷贝（module.exports对象，缓存输出的值），这也是为什么 require 是运行时加载的，因为只有到运行的时候才能生成一个模块对象。一旦模块文件加载执行完毕就会输出了一个缓存值，模块内部后续的变化影响不到这个值（不同脚本之间引用的是同一个值），如果要取到模块内部变化之后的值，那么就需要特别定义一个 get 函数作为取值器来返回模块内部的值

```text
CommonJS 的一个模块，就是一个脚本文件。require命令第一次加载该脚本，就会执行整个脚本，然后在内存生成一个对象

{
  id: '...', // 模块名
  exports: { ... }, // 模块输出的各个接口
  loaded: true, // 是否执行完毕
  ...
}

以后需要用到这个模块的时候，就会到exports属性上面取值。
即使再次执行require命令，也不会再次执行该模块，而是到缓存中取值（这就是上述提及的值拷贝和缓存）。
也就是说，CommonJS 模块无论加载多少次，都只会在第一次加载时运行一次，以后再加载，就返回第一次运行的结果，除非手动清除系统缓存

参考：https://es6.ruanyifeng.com/#docs/module-loader#CommonJS-%E6%A8%A1%E5%9D%97%E7%9A%84%E5%8A%A0%E8%BD%BD%E5%8E%9F%E7%90%86

```

## 循环加载

“循环加载”是指a脚本的执行依赖b脚本，而b脚本的执行又依赖a脚本  

```js
// 比如
// a.js
const a = require("./b.js");

// b.js
const a = require("./a.js");
```

在 esm 模块 和 cjs 模块中，循环加载的表现有差异：  

cjs 只输出已经执行的部分，还未执行的部分不会输出（此时模块脚本还未执行完毕，模块输出值还会改变）  

esm 先执行的脚本a引用另一个模块脚本b时，会先去执行脚本b，b中再引用脚本a时，引擎此时会认为b要从a引入的接口已经存在了，b脚本会正常执行，此时往往可能会报未定义的错误（这种未定义错误可以通过var或function的声明提升来解决）

> <https://es6.ruanyifeng.com/#docs/module-loader#%E5%BE%AA%E7%8E%AF%E5%8A%A0%E8%BD%BD>

## 模块内容动态注入

esm注入：利用其输出的接口在不同脚本中都是指向同一个实例的特点

```js
// module.js
// 输出一个对象引用，这个引用是只读的，但是内部的值是可以修改的
export default {
  // 对象内部的属性是可以修改的
  aInterface: {}
}

// inject.js
import m from "./module.js";
m.aInterface = {
  msg: "new things"
}
```

cjs注入：利用其一旦使用`require()`后就输出一个模块对象缓存，后续在不同脚本之间都使用该缓存值的特点

```js
// module.js
// 被 require 后会生成一个对象，所有脚本都用这个对象
module.exports = {
  aInterface: {}
}

// inject.js
// 一旦被 require 就会在执行完这个模块脚本后生成一个会被缓存的对象
const m = require("./module.js");
m.aInterface = {
  msg: "new things"
}
```

文章源码：<https://gitee.com/thisismyaddress/bocheng-blogs/tree/master/js/%E6%A8%A1%E5%9D%97%E8%AF%AD%E6%B3%95ESM%E4%B8%8ECJS>

参考：
><https://es6.ruanyifeng.com/#docs/module>
><https://es6.ruanyifeng.com/#docs/module-loader>
><https://zhuanlan.zhihu.com/p/337796076>  
