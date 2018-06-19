---
title: vscode配置前端vue开发环境
date: 2018-06-19 10:42:39
tags:
  - vscode
---

    摘要:
      使用vscode配置前端vue开发环境

1.安装好vscode后,安装常用的几个插件

    Auto Close Tag : 自动闭合标签插件

    Beautify : 格式化js,json,css,sass,html等文件

    ESLint : 使用eslint规范对代码进行处理

    file-icons : 文件图标,便利区分不同类型的文件

    Monokai Theme : 一款类似sublime text主流的主题设置,使界面美观,享受美好的编码心情

    Path Intellisense : 自动提示文件路径插件

    Prettier : 因为vscode默认的格式化是不能通过eslint校验规范的,需要改为此插件

    Vetur : vscode的vue工具插件

2.安装好上述插件后,对其进行配置设置

    在文件->首选项->设置->用户设置里面写入以下配置

  ```json
  {
    "workbench.startupEditor": "newUntitledFile",
    "extensions.ignoreRecommendations": false,
    "workbench.iconTheme": "file-icons",
    "workbench.colorTheme": "Monokai",
    "vetur.format.defaultFormatter.js": "vscode-typescript",
    "extensions.autoUpdate": false,
    "update.channel": "none",
    "eslint.autoFixOnSave": false,
    "files.autoSave": "off",
    "eslint.validate": [
        "javascript",
        "javascriptreact",
        {
            "language": "vue",
            "autoFix": true
        },
        {
            "language": "html",
            "autoFix": true
        },
        "vue"
    ],
    "eslint.options": {
        "plugins": [
            "html"
        ]
    },
    "editor.tabSize": 2,
    "prettier.eslintIntegration": true,
    "vetur.format.defaultFormatter": {
        "html": "prettier",
        "css": "prettier",
        "postcss": "prettier",
        "scss": "prettier",
        "less": "prettier",
        "js": "prettier",
        "ts": "prettier",
        "stylus": "stylus-supremacy"
    }
  }
  ```
