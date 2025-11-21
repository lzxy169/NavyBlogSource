---
title: Hexo 命令
date: 2018-04-08 14:10:08
tags:
categories: [Hexo]
---

### 文章
```
$ hexo new "new article"
```

### 草稿
草稿相当于很多博客都有的“私密文章”功能。
```
$ hexo new draft "new draft"
```

会在source/drafts目录下生成一个new-draft.md文件。但是这个文件不被显示在页面上，链接也访问不到。也就是说如果你想把某一篇文章移除显示，又不舍得删除，可以把它移动drafts目录之中。

如果你希望强行预览草稿，更改配置文件：
```
$ render_drafts: true
```

或者，如下方式启动server：
```
$ hexo server --drafts
```

下面这条命令可以把草稿变成文章，或者页面：
```
$ hexo publish [layout] <filename>
```

### hexo
```
npm install hexo -g #安装  
npm update hexo -g #升级  
hexo init #初始化
```

### 简写
```
hexo n "我的博客" == hexo new "我的博客" #新建文章
hexo p == hexo publish
hexo g == hexo generate#生成
hexo s == hexo server #启动服务预览
hexo d == hexo deploy#部署
```

### 服务器
```
hexo server #Hexo 会监视文件变动并自动更新，您无须重启服务器。
hexo server -s #静态模式
hexo server -p 5000 #更改端口
hexo server -i 192.168.1.1 #自定义 IP

hexo clean #清除缓存 网页正常情况下可以忽略此条命令
hexo g #生成静态网页
hexo d #开始部署
```

### 监视文件变动
```
hexo generate #使用 Hexo 生成静态文件快速而且简单
hexo generate --watch #监视文件变动
```

### 完成后部署
```
两个命令的作用是相同的
hexo generate --deploy
hexo deploy --generate

hexo deploy -g
hexo server -g
```

### 草稿
```
hexo publish [layout] <title>
```

### 模版
```
hexo new "postName" #新建文章
hexo new page "pageName" #新建页面
hexo generate #生成静态页面至public目录
hexo server #开启预览访问端口（默认端口4000，'ctrl + c'关闭server）
hexo deploy #将.deploy目录部署到GitHub

hexo new [layout] <title>
hexo new photo "My Gallery"
hexo new "Hello World" --lang tw
```

### 模版（Scaffold）
```
hexo new photo "My Gallery"
```

### 写作
```
hexo new page <title>
hexo new post <title>
```

### 推送到服务器上
```
hexo n #写文章
hexo g #生成
hexo d #部署 #可与hexo g合并为 hexo d -g
```

