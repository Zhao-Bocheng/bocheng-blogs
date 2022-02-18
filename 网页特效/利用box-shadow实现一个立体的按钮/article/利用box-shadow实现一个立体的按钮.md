# 利用box-shadow实现一个立体的按钮

改变按钮`button`的默认样式，创建一个立体的按钮

![效果图](img/finalEffect.gif "效果图")

这个效果是通过`box-shadow`实现

这里要注意`box-shadow`是允许对同一个元素设置多个阴影的，需要用逗号分隔；**多个阴影的层叠顺序是：写在前面的阴影将覆盖在后写的阴影上面**

主要代码：

```html
<button>Btn</button>
```

```css
button {
  /* 关键是使用这个 盒子阴影 的样式 */
  box-shadow: -6px 6px 8px inset rgba(255, 255, 255, 0.6),
    6px -6px 8px inset rgba(0, 0, 0, 0.2);
}

/* 按钮被点击时将阴影切换 */
button:active {
  box-shadow: -6px 6px 8px inset rgba(0, 0, 0, 0.2),
    6px -6px 8px inset rgba(255, 255, 255, 0.6);
}
```

## 演示链接

CodePen演示链接：<https://codepen.io/Zhao-Bocheng/pen/zYwrPEL>

文章源码：<https://gitee.com/thisismyaddress/bocheng-blogs/tree/master/%E7%BD%91%E9%A1%B5%E7%89%B9%E6%95%88/%E5%88%A9%E7%94%A8box-shadow%E5%AE%9E%E7%8E%B0%E4%B8%80%E4%B8%AA%E7%AB%8B%E4%BD%93%E7%9A%84%E6%8C%89%E9%92%AE>

参考：
><https://www.bilibili.com/video/BV1w64y1b7eF>
