# GitBook 搭建日志

## 环境准备

### 1. 项目初始化
```bash
# 在本地创建项目目录
mkdir -p chaoxuzhong.github.io/notes
cd chaoxuzhong.github.io/notes
```

### 2. GitBook 安装
```bash
# 安装 GitBook CLI
sudo npm install -g gitbook-cli
```

## 问题排查

### 1. GitBook 初始化报错
执行 `gitbook init` 时遇到以下错误：
```text
Installing GitBook 3.2.3
/usr/local/lib/node_modules/gitbook-cli/node_modules/npm/node_modules/graceful-fs/polyfills.js:287
if (cb) cb.apply(this, arguments)
^
TypeError: cb.apply is not a function
at /usr/local/lib/node_modules/gitbook-cli/node_modules/npm/node_modules/graceful-fs/polyfills.js:287:18
at FSReqCallback.oncomplete (node:fs:199:5)
```

### 2. 尝试使用 HonKit
```bash
# 卸载 GitBook CLI
sudo npm uninstall -g gitbook-cli

# 安装 HonKit
npm install -g honkit
```

但遇到新错误：
```bash
/usr/local/lib/node_modules/honkit/node_modules/undici/lib/web/fetch/response.js:527
ReadableStream
^
ReferenceError: ReadableStream is not defined
```

## 解决方案：降级 Node.js

### 1. 安装 nvm
```bash
# 安装 nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# 使配置生效
source ~/.zshrc
```

### 2. 切换 Node.js 版本
```bash
# 安装 Node.js v16
nvm install 16

# 切换到 v16
nvm use 16
```

### 3. 重新安装 HonKit
```bash
# 卸载旧版本
npm uninstall -g honkit

# 安装新版本
npm install -g honkit
```

### 4. 初始化项目
```bash
honkit init
```



### 5. 切换 Node.js 版本 到18才解决问题
```bash
# 安装 Node.js v16
nvm install 18

# 切换到 v16
nvm use 18
```


## 本地项目启动
```bash
# 初始化书籍
honkit init

# 启动本地服务
honkit serve

```


## 支持插件，增加导航功能
`book.json` 配置文件中添加插件来支持导航功能。让我帮你设置：

1. 首先创建或编辑 `book.json` 文件：
```json
{
  "title": "MIT 6.824 学习笔记",
  "author": "Your Name",
  "plugins": [
    "chapter-fold",
    "back-to-top-button",
    "copy-code-button",
    "search-pro"
  ],
  "pluginsConfig": {
    "theme-default": {
      "showLevel": true
    }
  }
}
```

2. 安装插件：
```bash
# 删除之前的 node_modules 和 _book
rm -rf node_modules _book

# 安装插件
honkit install
```

3. 重启服务：
```bash
 honkit serve
```

主要插件说明：
- `expandable-chapters`：支持章节的展开/折叠
- `expandable-chapters-small`：使用更小的箭头图标
- `chapter-fold`：支持目录折叠功能
- `navigation`：添加导航功能

如果安装插件时遇到问题，也可以尝试使用 npm 直接安装：
```bash
npm install gitbook-plugin-expandable-chapters
npm install gitbook-plugin-chapter-fold
npm install gitbook-plugin-navigation
```

然后重新构建和启动服务：
```bash
honkit build
honkit serve
```
