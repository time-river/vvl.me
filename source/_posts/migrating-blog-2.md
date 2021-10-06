---
title: 第二次迁移博客
date: 2019-06-12 20:21:18
categories: "local movies"
---

忙碌了一天，给博客换了个主题。

这次把主题从[fexo](https://github.com/time-river/fexo)换成了[hexo-theme-next](https://github.com/theme-next/hexo-theme-next)，因为后者的表现形式更丰富。对比之前的主题，在以下做出了改进：

- 使用[pangu.js](https://github.com/theme-next/theme-next-pangu)支持在中文与英文、数字、符号之间自动添加空格
- 使用[fancybox](https://github.com/theme-next/theme-next-fancybox3)显示图片

此外，去掉了一些插件：

- [hexo-search](https://github.com/forsigner/hexo-search)与fexo主题耦合，在新主题上没有用处
- [hexo-renderer-ejs2](https://www.npmjs.com/package/hexo-renderer-ejs2)不再维护，使用[hexo-renderer-ejs](https://github.com/tj/ejs)替代
- 主题已有代码高亮功能，插件[hexo-prism-plugin](https://github.com/ele828/hexo-prism-plugin)不再使用
- 主题已有LaTex功能，插件[https://www.npmjs.com/package/hexo-renderer-mathjax2](https://github.com/Mybrc91/hexo-renderer-mathjax2)不再使用
- 去掉看起来没啥用的以来[eslint](https://www.npmjs.com/package/eslint)

补充了博客的花絮，也把关于生活的随笔从这里移出，打算迁移到另一个站点下面。
