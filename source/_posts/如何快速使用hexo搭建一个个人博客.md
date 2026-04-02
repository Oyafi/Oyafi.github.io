---
title: 如何快速使用hexo搭建一个个人博客
date: 2026-04-02 20:18:48
categories: 技术
tags: hexo
---

## 使用hexo快捷搭建博客指南

### 1.基础环境下载

首先下载node

```shell
# 下载并安装 nvm：
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash

# 代替重启 shell
\. "$HOME/.nvm/nvm.sh"

# 下载并安装 Node.js：
nvm install 24

# 验证 Node.js 版本：
node -v # Should print "v24.14.1".

# 验证 npm 版本：
npm -v # Should print "11.11.0".
```

然后下载hexo的脚手架

```shell
npm install -g hexo-cli
```

将hexo加入到环境变量

```shell
echo 'PATH="$PATH:./node_modules/.bin"' >> ~/.zshrc
```

### 2.初试化项目

选择一个空文件加并进入

~~~shell
mkdir blog
cd blog
~~~

初始化

~~~shell
hexo init myBlog
cd myBlog
npm install
~~~

这样你就得到了一个个人博客了，执行命令启动博客

~~~
hexo server
~~~

打开弹出的链接就可以看到了

### 3.更换主题

hexo创建时默认使用的主题是landscape，我们可以在`https://hexo.io/themes/`中寻找自己想要的主题，然后下载并更换

比如我们想要更换为butterfly主题，我们首先在网站中找到该主题，点击其标题，跳转到其github界面

![截屏2026-04-02 20.34.38](https://oyafi-blog.oss-cn-beijing.aliyuncs.com/%E6%88%AA%E5%B1%8F2026-04-02%2020.34.38.png)



然后复制其路径，用git clone 下载下来

![截屏2026-04-02 20.35.46](https://oyafi-blog.oss-cn-beijing.aliyuncs.com/%E6%88%AA%E5%B1%8F2026-04-02%2020.35.46.png)

注意在博客根目录再在克隆

```shell
git clone "你复制的仓库路径" themes/"主题名称"
```

然后进入 themes/"主题名称"目录，将.git .github .gitignore文件删除，以免后续部署时发生仓库嵌套

回到根目录，将根目录下的_config.yml中的theme字段改为你主题的名称

将主题文件中的_config.yml复制到根目录并换名称

~~~shell
cp themes/"主题名称"/_config.yml _config."主题名称".yml 
~~~

这样主题的更换就完成了，具体的主题内部的配置，可以进入其示例网站，一般都会有字段说明

### 4.将博客部署到个人的github.io上

首先在github上创建一个仓库，仓库的命名如下

Xxxx.github.io          Xxxx 为你github的名称

然后修改仓库setting/pages/Build and deployment的source，将其改为GitHub actions

![](https://oyafi-blog.oss-cn-beijing.aliyuncs.com/%E6%88%AA%E5%B1%8F2026-04-02%2020.49.27.png)

然后初始化本地仓库，关联远程仓库，并提交代码

~~~shell
git init 
git remote add origin "仓库路径"
git add .
git commit -m "init"
git push origin main
~~~

之后，在本地仓库的.github/workflows目录下创建pages.yml文件

~~~shell
vim .github/workflows/pages.yml
~~~

在其中写入以下信息，注意替换node版本数字为自己的版本

~~~yaml
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
~~~

然后重新提交

~~~shell
git add .
git commit -m "deploy"
git push origin main
~~~

这样就完成了，可以访问"你的名称".github.io，查看是否部署成功
