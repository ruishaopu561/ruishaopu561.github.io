---
title: Hexo + Github
date: 2020-06-12 22:11:13
top_img: /img/cover/hexo_github.png
cover: /img/cover/hexo_github.png
description: 配置个人专属博客网站，采坑小结
tags: 
  - config
  - 教程
---
这是我的第一篇博客，因为暂时没有什么内容需要记录，就记录一下这个博客创建的过程。

这里是[参考博文链接](https://zhuanlan.zhihu.com/p/35668237)，这里是[主题Butterfly](https://github.com/jerryc127/hexo-theme-butterfly)的链接地址。

参考博文的记录非常详细了，这里主要记录一些我认为重要的以及踩过坑的地方。

### 创建文章

```bash
hexo new post article-title
```

这里```article-title```是要创建的.md文件的名称。

### Hexo清理、生成、运行与提交

+ Hexo清理
```bash
hexo clean
```

+ Hexo生成
```bash
hexo g
```

+ Hexo运行
```bash
hexo s
```

输入指令后根据提示就可以在```http://localhost:4000```看到预期结果了

+ Hexo提交
```bash
hexo d
```

这里需要先在```_config.yml```配置发布，代码如下：
```bash
deploy:
  type: git
  repo: https://github.com/ruishaopu561/ruishaopu561.github.io
  branch: hexo
  message: update blog again, maybe something useful
```

`branch`是你上传代码的分支，`message`是上传时提交的信息（其实`hexo d`就是做了git add到push那一套）

### 关于github上的repo

在你应用主题之后，发布上去的应该是`public`文件夹里的代码，这也正是`hexo g`生成文件的位置。所以如果网站访问有问题的话应该去看看github里的文件分别是什么。

### 应用主题之后的修改

应用主题之后可能会想修改默认背景、默认头像等。这里可以参考`Butterfly`主题的[参考文档](https://docs.jerryc.me)

### 解决markdown里latex公式显现问题

[参考链接1](https://www.jianshu.com/p/e8d433a2c5b7)  
[参考链接2](https://www.jianshu.com/p/7ab21c7f0674)

#### 第一步：删除无用包
```bash
npm uninstall hexo-renderer-marked --save
npm uninstall hexo-math --save
```

#### 第二步：安装新的替换包
```bash
npm install hexo-renderer-kramed --save
npm install hexo-renderer-mathjax --save
```

#### 第三步：修改代码
更改`<your-project-dir>/node_modules/hexo-renderer-kramed/lib/renderer.js`，更改：
```js
// Change inline math rule
function formatText(text) {
    // Fit kramed's rule: $$ + \1 + $$
    return text.replace(/`\$(.*?)\$`/g, '$$$$$1$$$$');
}
```
为：
```js
// Change inline math rule
function formatText(text) {
    return text;
}
```

更新 Mathjax 的 CDN 链接，打开`<path-to-your-project>/node_modules/hexo-renderer-mathjax/mathjax.html`，把`<script>`更改为：
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-MML-AM_CHTML">
</script>
```

更改默认转义规则
打开`<path-to-your-project/node_modules/kramed/lib/rules/inline.js`，

把:
```js
escape: /^\\([\\`*{}\[\]()#$+\-.!_>])/,
```
更改为:
```js
escape: /^\\([`*\[\]()# +\-.!_>])/,
```
把
```js
em: /^\b_((?:__|[\s\S])+?)_\b|^\*((?:\*\*|[\s\S])+?)\*(?!\*)/,
```
更改为:
```js
em: /^\*((?:\*\*|[\s\S])+?)\*(?!\*)/,
```

#### 第四步：开启mathjax
在**主题**里的` _config.yml`中开启 Mathjax， 找到 **mathjax** 字段添加如下代码：
```yml
mathjax:
    enable: true
```
在博客中开启 Mathjax，添加以下内容：

```markdown
---
···
mathjax: true
---
```