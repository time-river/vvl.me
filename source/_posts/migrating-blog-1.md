---
title: 迁移博客有感
date: 2016-09-07 09:38:45
categories: "local movies"
---

恩，我发现我还是在这里写了不少东西的。虽然才使用半年有余，已然留下了不少东西。

DigitalOcean 的优惠券到期了，几个月前在 V 站看到 [carina](https://getcarina.com) 提供免费 Docker 容器的消息，当初跟风注册了一把，也没用过：当测试机延迟太大，其他用途呢，好像也没啥用，倒是有不少 v 友因部署了某种服务而被销号。现在这可派上了用场。

当初使用 Hexo，还是因为看到了[这个](https://www.v2ex.com/t/260175)，主题拉取过来便使用了，也没做什么调整，都是默认的设置。后续遇到了不少问题：比如使用 LaTex 公式，代码高亮等。这次迁移，便完善、添加了一些功能：

* 使用 [hexo-renderer-mathjax](https://github.com/phoenixcw/hexo-renderer-mathjax) 为 LaTex 提供语法支持
* 代码高亮使用 [prism](http://prismjs.com/)，由插件 [hexo-prism-plugin](https://github.com/ele828/hexo-prism-plugin) 实现
* [hexo-generator-feed](https://github.com/hexojs/hexo-generator-feed) 为 RSS 订阅提供支持(虽然知道没几个人看)
* 加入 [hexo-search](https://github.com/forsigner/hexo-search) 插件，启用搜索
* [hexo-generator-sitemap](https://github.com/hexojs/hexo-generator-sitemap) 生成站点地图，为搜索引擎优化
* 加入 [hexo-deployer-git](https://github.com/hexojs/hexo-deployer-git)

使用过 Jekyll，知道代码高亮可以用 [Pygments](http://pygments.org/) 与 [hightlightjs](https://highlightjs.org/) 实现。Hexo 不能使用 [Pygments](http://pygments.org/) 咯，捣鼓了半天 [hightlightjs](https://highlightjs.org/)，不仅自定义的 js 不可用，还没法实现行号。后来发现了 [prism](http://prismjs.com/)，恩，被 [Coy](http://prismjs.com/index.html?theme=prism-coy) 主题吸引了，看着挺不错。不过功能拓展插件没搞明白怎么使用，是个遗憾。

三十多篇文章，断断续续迁移了三天了吧。迁移过程中草草地过了一遍，又修正了一些版式、内容错误，删减了不少标签。以现在的眼光看，当初写的“废话”真多，有些内容渐渐模糊。

写到这到想起了分类只有`style`与`local_movies`的原因。嗯这两个词是两种图标的名称，我呢，赋予了他们一些含义：`style`嘛，也能看到，学到的东西都归类于此，生活的多姿多彩亦如此。`local_movies`则意指过去像一场场电影，值得未来来品味。
