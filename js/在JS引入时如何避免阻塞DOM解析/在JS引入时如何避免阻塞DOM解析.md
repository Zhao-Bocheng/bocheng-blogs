# 在JS引入时如何避免阻塞DOM解析

浏览器对 HTML 文档代码的解析是**按代码的编写顺序进行**的，如果在浏览器对HTML文档进行解析时，遇到了不带任何属性的`<script>`标签，浏览器会直接加载和执行脚本，这会阻塞后续的 DOM 解析。如果 JS 代码中的代码涉及 DOM 操作而此时目标节点又没有加载出来则会导致代码执行错误。这就需要采取合适的**脚本调用策略**来避免代码执行错误的情况了

## 在文档体最后引入JS代码

为了不阻塞 DOM 解析，**利用浏览器按顺序解析文档并执行脚本代码的特点，将JS相关代码放在HTML文档体的最后**（`</body>`之前），这样当整个文档解析完毕后才会加载并执行 JS 脚本。如果 JS 脚本间有相互依赖关系，则引入时要注意顺序，比如`a.js`依赖于`b.js`的内容，则需要先加载被依赖项`b.js`

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <title>在文档体最后引入JS代码</title>
</head>
<body>
    <!-- HTML文档内容 -->
    
    <!-- 在文档体最后引入JS -->
    <script>
        // JS
    </script>

    <!-- a.js 依赖于 b.js，先导入 b.js -->
    <script src="b.js"></script>
    <script src="a.js"></script>
</body>
</html>
```

## 利用加载事件

为了在HTML文档内容加载完毕后才执行 JS 脚本，可以**利用`document`的`DOMContentLoaded`事件或`window`的`load`事件**。利用这两个事件可以在加载 JS 脚本内容后不立即执行，而是等事件触发时才执行，所以 JS 脚本需要放在对应的事件处理函数中。（注意，脚本的加载依然会阻塞 script 标签后的 DOM 解析）

以上两个加载事件是有一点区别的：`DOMContentLoaded`事件在 DOM 加载完成后就会触发（不用等待页面渲染完毕），而`load`是在与页面所有依赖资源加载完毕后触发（*建议使用`DOMContentLoaded`*，这样在 DOM 加载完毕后就可以执行JS代码了，不必等待其他内容的加载）

利用以上两个事件，就不需要担心脚本的执行会阻塞 DOM 解析了（脚本的加载仍然会阻塞在其后的 DOM 解析）

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <title>利用加载事件引入JS代码</title>
    <!-- 引入js代码 -->
    <script>
        document.addEventListener('DOMContentLoaded', function() {
            // JS 代码
        })
    </script>
    <!-- 引入外部JS文件 -->
    <script src="src.js"></script>
</head>
<body>
    <!-- HTML文档内容 -->
</body>
</html>
```

```js
// src.js
document.addEventListener('DOMContentLoaded', function() {
    // JS 代码
})
```

## 利用`<script>`的`async`和`defer`属性

`async`和`defer`属性的使用**只适用于对外部JS文件的引入**

### `async`

使用`async`属性时可告知浏览器在遇到`<script>`元素时不要中断后续 DOM 解析（**异步加载**），加载完毕后会立刻执行，执行时会阻塞 DOM 解析

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <title>利用async属性</title>
    <!-- 引入外部JS文件 -->
    <script src="path/to/src.js" async></script>
</head>
<body>
    
</body>
</html>
```

#### 优点和局限性

优点：当使用`async`属性导入 JS 文件时不需要特地等待 DOM 解析完毕（不像上述的两种方法——在文档底部引入和利用加载事件引入），DOM 的解析和 JS 的加载并行，这能提高网站的性能，这个优点在一些有大量JS代码需要加载的大型网站上体现得更明显（这也是 **`async` 属性诞生的初衷**）

局限：只适用于外部引入的JS文件；使用`async`会使JS异步加载，使JS代码的加载和执行的顺序不定，这样**当多个JS文件之间有相互依赖的关系时，可能会导致代码执行出错**；JS 代码的执行会阻塞 DOM 解析，此时脚本不应该有 DOM 操作

### `defer`

`defer`可以**延迟外部 JS 脚本到整个页面解析完毕之后才执行**。脚本也是异步加载的，但当有多个`script`标签使用了`defer`，HTML5 规范要求脚本应该**按照他们的出现顺序执行**，并且都会在`DOMContentLoaded`事件之前执行

> 不过在实际当中，推迟执行的脚本不一定总会按顺序执行或者在 DOMContentLoaded 事件之前执行，因此最好只包含这样一个脚本\
> *摘自《JavaScript高级程序设计》2.1.2节*

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>利用defer属性</title>
    <!-- 按顺序引入JS -->
    <script src="path/to/src1.js" defer></script>
    <script src="path/to/src2.js" defer></script>
    <script src="path/to/src3.js" defer></script>
</head>
<body>
    
</body>
</html>
```

### 选用`async`和`defer`的策略

如果脚本无需等待页面解析，且无依赖独立运行，那么应使用 `async`；\
如果脚本需要等待页面解析，且依赖于其它脚本，调用这些脚本时应使用 `defer`，将关联的脚本按所需顺序引入

为了更好展示`async`、`defer`和无属性时的差别，这里建议读者前往这个网站看里面的精辟解析：

><https://www.growingwiththeweb.com/2014/02/async-vs-defer-attributes.html>

## 总结

直接将JS代码在HTML文档的body最后引入，会使JS代码的**引入位置受限**；

通过两个加载事件`document`的`DOMContentLoaded`和`window`的`load`的事件处理函数，可以避免 JS 的执行阻塞 DOM 解析，但却使得JS代码本身的**内容不纯粹**，且 JS 的加载依然会阻塞 DOM 解析，如果 JS 文件很大的话，这个阻塞将可能让页面迟迟无法显示（此时还不如在最后引入）；

无论是在文档底部引入还是利用加载事件引入，在需要载入大量JS代码的大型网站上，都可能会**带来显著的性能消耗**。

利用`async`或`defer`属性可以在实现HTML文档解析和JS代码加载并行，但`async`和`defer`**只适用于外部引入的JS文件**；

**`async`可以实现异步加载JS脚本**（非常适合需要载入大量JS代码的大型网站），不必特地等待 DOM 解析完毕，但**无法控制JS脚本的载入和执行顺序**，不适合多个相互依赖的 JS 脚本，可能会导致代码执行错误；

**`defer`可以延迟JS到 DOM 解析完成后执行**，按照标准的 HTML5 规范，`defer`可以保证脚本按照引入顺序执行（即使也是异步加载脚本）。

文章源码：<https://gitee.com/thisismyaddress/bocheng-blogs/tree/master/js/%E5%9C%A8JS%E5%BC%95%E5%85%A5%E6%97%B6%E5%A6%82%E4%BD%95%E9%81%BF%E5%85%8D%E9%98%BB%E5%A1%9EDOM%E8%A7%A3%E6%9E%90>

参考：
><https://developer.mozilla.org/zh-CN/docs/Learn/JavaScript/First_steps/What_is_JavaScript#%E8%84%9A%E6%9C%AC%E8%B0%83%E7%94%A8%E7%AD%96%E7%95%A5>\
><https://developer.mozilla.org/zh-CN/docs/Web/API/Document/DOMContentLoaded_event>\
><https://developer.mozilla.org/zh-CN/docs/Web/API/Window/load_event>\
><https://www.growingwiththeweb.com/2014/02/async-vs-defer-attributes.html>
