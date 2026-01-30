---
title: 我艰难的blog搭建之旅
date: 2026-01-30 09:20:05
tags: 
    - hexo
    - github
    - 教程
categories: 教程
toc: true
mathjax: true
---

一直想要搭建一个自己 blog，但一直没有选择开始，也算是机缘巧合下，看到一个特别喜欢的主题，决定从零搭建。期间不管是框架的改变，还是编译遇到各种问题，都搞得晕头转向，不过最终也算是成功了。


## 想法背景
我是在寻找 "解决windows跨平台编译" 问题的过程中，看到了别人搭建的[Blog][other-blog]，让我萌发搭建自己的Blog的想法。

最开始直接搜索的如何基于github pages 搭建一个 blog，推荐的是基于 Jekyll 框架和 github pages 搭建，但是主题、build都比较复杂，搞了一天，发现维护部署不够简单，然后看了篇 hexo 的搭建教程，发现非常适合我这种懒狗，于是一拍即合，开始搭建......


## 搭建过程
Hexo 的搭建过程主要参考了知乎的这篇 [教程][step]。

### 环境准备
Hexo 是基于 node.js 搭建的，所以首先需要准备好 Node.js 环境，Jekyll 是基于 Ruby 搭建的，我之前自己鼓捣了 obsidian 的插件，刚好有 node 环境，这也是我选择 hexo 的原因。

Node.js 的安装直接参考 [菜鸟关于Node.js的教学](https://nodejs.org/en/download)就可以了。安装好后，在命令提示符窗口（cmd）使用 npm 全球安装 hexo-cli：

```bash
npm install -g hexo-cli
```

### 创建博客
在选择好目录后，使用 hexo init 初始化博客：

```bash
hexo init <blog-name>
```
hexo 初始化需要空目录，否则会报错，所以需要把 .git 目录备份到其它地方，初始化后再把 .git 目录恢复。

进入博客目录，安装依赖：
```bash
npm install
``` 
然后就可以本地进行构建（`hexo generate` 或 `hexo g`）；
```bash
hexo generate # 简写 hexo g
```

启动本地服务器（ `hexo server`），查看效果：

```bash
hexo server # 简写 hexo s

# 如果要指定服务端口，默认是 4000
hexo server -p 4001
```

清除 build 目录：

```bash
hexo clean 
```

### 选择主题
选择一个喜欢的主题，我选择的是 [hexo-theme-oranges][themes]，可以直接 clone 到 themes 目录下：

```bash
git clone https://github.com/zchengsite/hexo-theme-oranges.git themes/oranges
```

或者使用 submodule 方式添加主题(我使用的这种方式)，这种如果别人更新了主题，我也可以很方便地更新：

```bash
git submodule add https://github.com/zchengsite/hexo-theme-oranges.git themes/oranges
```

如果要删除 submodule 方式添加的主题，相对来说比较麻烦，需要执行以下命令：

1. 停止对子模块的追踪
```bash
git submodule deinit themes/oranges
```
这会从 _.git/config_ 文件中移除子模块的相关信息。

2. 删除 submodule 目录
```bash
git rm themes/oranges
```
同时，删除 _.git/modules/子模块路径_ 目录：
```bash
rm .git/modules/themes/oranges 
```
3. 从 Git 索引中移除子模块
```bash
git rm --cached themes/oranges
```
4. 提交更改
```bash
git commit -m "Remove submodule themes oranges"
```

### 配置博客
编辑博客根目录下的 _config.yml 文件，将 theme 字段设置为 oranges：

```yaml
theme: oranges
```

然后将 oranges 下的 _config.yml 文件中的内容复制到根目录 _config.oranges.yml，用来修改主题的相关配置。

### 部署到 github pages

可以使用 hexo deploy 部署到 github pages：

```bash
hexo deploy
``` 

但这种方式没办法将源码提交上去，或者是我自己也没搞懂，等有时间再研究一下。

我是直接将源码提交到 github pages 分支，然后在 github pages 中设置为从该分支构建，添加了一个 workflow 文件，用来自动构建并部署到 github pages。我让直接让 AI 帮忙写了一个，内容如下：

```yml
name: Pages

on:
  push:
    branches:
      - main # default branch

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          # If your repository depends on submodule, please see: https://github.com/actions/checkout
          submodules: recursive
      - name: Use Node.js 20
        uses: actions/setup-node@v4
        with:
          # Examples: 20, 18.19, >=16.20.2, lts/Iron, lts/Hydrogen, *, latest, current, node
          # Ref: https://github.com/actions/setup-node#supported-version-syntax
          node-version: "20"
      - name: Cache NPM dependencies
        uses: actions/cache@v4
        with:
          path: node_modules
          key: ${{ runner.OS }}-npm-cache
          restore-keys: |
            ${{ runner.OS }}-npm-cache
      - name: Install Dependencies
        run: npm install
      - name: Build
        run: npm run build
      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public
  deploy:
    needs: build
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4

```
这样的话，只需要在每次修改博客内容后，提交到 main 分支，就会自动触发 workflow 构建并部署到 github pages。

## 遇到的问题及解决方案

在搭建过程中，遇到了不少问题，这里记录下来提供参考：

### 问题一：主题配置不生效

在配置oranges主题时，一开始创建了单独的配置文件 `_config.oranges.yml`，但发现主题没有生效。浏览器报错如下：

```
Refused to apply style from 'http://localhost:4000/css/main.css' because its MIME type ('text/html') is not a supported stylesheet MIME type...
Failed to load resource: the server responded with a status of 404 (Not Found)
```

**问题原因**：
Hexo构建是获取css资源是直接访问根目录获取的，我的配置中 `url: https://feng-jianwutong.github.io/fengjianwutong`，`root: /fengjianwutong/`，如果访问 `http://localhost:4000/css/main.css` 是正确的，但是访问 `https://feng-jianwutong.github.io/fengjianwutong/css/main.css` 就会报错，因为 `css/main.css` 是在 `public` 目录下的，而不是在 `root` 目录下的。原因是主题中的 layout 中的代码使用的是绝对路径，如果使用子目录部署会有问题。

**解决方案**：
我直接 fork 主题代码到[我的仓库][my-themes]，修改了其中的代码，将绝对路径改为相对路径，解决了这个问题。

比如在 `oranges/layout/_partial/head.ejs` 中，有如下代码：

```ejs
<link rel="stylesheet" href="/css/main.css">
```

我将其改为：

```ejs
<link rel="stylesheet" href="<%- url_for('/css/main.css') %>">
```

### 问题二：标签页面无法访问

在配置完主题后，我发现访问 `/tags/` 页面时返回404错误，尽管文章中定义了标签。

**解决方案**：
这个参考主题的 README 中提到了如何使用标签索引插件，在 hexo 博客项目根目录下执行，在source文件夹下生成tags文件夹，用来存放标签页面的索引文件。

```bash
hexo new page tags
```
接着修改tags文件夹下index为以下内容
```markdown
---
title: tags
date: 2019-05-03 12:03:35
type: "tags"
categories:
tags:
---
```

并在配置文件_config.oranges.yml修改对应enable为true，如不想展示，设置为false即可。
```yml
navbar:
  -
    name: 标签
    enable: true
    path: /tags/
```

## 总结

这次部署经历期间好几次都感觉要失败准备放弃，但是通过查阅、请教公司前端的同事，最后还是克服重重困难，成功部署。

我想说的是，成功真的会让人上瘾，这次的成功，会让我下一次遇到困难时迎难而上。

最后再次感叹时代的进步，AI真的很好用，帮助我快速定位和解决问题。

[other-blog]: https://ashe27.github.io/2025/07/18/202507181/ "解决windows跨平台编译问题" 
[cankao]: https://hexo.theme.oranges.zcheng.site/ "参考站点"
[step]: https://zhuanlan.zhihu.com/p/60578464 "步骤参考"
[themes]: https://github.com/zchengsite/hexo-theme-oranges?tab=readme-ov-file "主题"
[my-themes]: https://github.com/Feng-jianwutong/hexo-theme-oranges "我的主题"
