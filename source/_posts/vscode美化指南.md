---
title: vscode美化指南
date: 2026-04-02 18:43:50
categories: 技术
tags: vscode
---

### 这是一个个人使用的vscode改造指南

首先当然是下载一堆好看但没用的插件了

1. Vscode-icons: 自动给文件加上小图标
2. Indent-rainbow: 缩进美化
3. Code Runner: 赋予vscode直接运行代码的能力
4. Better Comments：给注释加上颜色
5. Image preview：通过链接预览图片
6. Path Intellisense：资源路径补全

### Background插件配置指南

上述的插件只是优化代码环境，如果要美化整个vscode的环境，推荐安装background插件。

下载后打开其配置界面

![截屏2026-04-02 19.48.36](https://oyafi-blog.oss-cn-beijing.aliyuncs.com/%E6%88%AA%E5%B1%8F2026-04-02%2019.48.36.png)

可以看到其分别支持辅助栏，编辑器，全屏，面板，侧边栏的配置，直接点击在setting.json中编辑即可，这里注意，如果配置的是Fullscreen，那么其他的配置就没有用了，会被覆盖。

```json
"background.fullscreen": {
        "images": ["file:///Users/oyafi/Documents/mainBack.png"],
        "opacity": 0.2,
        "size": "cover",
        "position": "center",
        "interval": 0,
        "random": false
    },
```

这里以全屏的配置为例，其他的配置也是一样的，具体各参数的含义可以查看官方的文档，一般添加一下图片路径，改一下透明度就可以了。

ok，这样vscode是不是就漂亮多了。
