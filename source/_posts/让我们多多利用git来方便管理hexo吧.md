---
title: 让我们多多利用git来方便管理hexo吧
date: 2016-08-27 13:39:46
tags: [新技能, git, hexo, 写作]
---

本文的命令的idea都来自于http://www.craigjperry.com/hexo-with-github-pages.html

## 创建本地Hexo目录

现在的hexo版本又一个问题，就是如果先`git init`再`hexo init`的话，回导致`.git`目录被删除，所以解决方案就是：

——把两者的顺序换一下！

```bash
$ npm install hexo -g #先安装hexo命令行工具
$ hexo init #吃我大init辣
$ npm install #快使用npm
$ hexo s #先看看能不能冒烟
```

至此hexo目录应该就已经成功设置了，这里插播一下`.gitignore`文件，用[https://www.gitignore.io/api/node](https://www.gitignore.io/api/node) 这里的内容，可以解决大多数问题

接下来我们一套标jŭn的`git init`流程走一遍就可以了

```bash
$ git init
$ git add .
$ git commit -m 'Initial commit'
```

## 创建一个Git仓库

1. 在github网站上创建一个同名的空仓库：<your-user-name>.github.io

2. 将本地hexo目录的remote指向这个仓库：

   ```bash
   $ git remote add origin <remote repo URL>
   ```

3. 然后在本地创建一个orphan分支（这个中文应该怎么叫？孤立分支？）：

   ```bash
   $ git checkout --orphan meta-content #创建孤立分支
   $ git push -u origin meta-content #不要忘记顺便推送一下
   $ git checkout master #记得切换回master
   $ git rm -rf * #清除master里面的内容
   ```

## 配置git部署工具

*首先确定从ssh密钥可以正常工作*

```bash
$ npm install hexo-deployer-git --save
```



编辑_config.yml文件，修改里面的deploy字段：

```yaml
deploy:
  type: git
  repo: ssh://git@github.com/<Your-Github-Username>/<your-github-username>.github.io
  branch: master
```



然后我们就可以

```bash
$ hexo deploy
```

轻松走上hexo写作之路！



### 正确的创建新文章的姿势

1. 首先确定我们所在的分支：

   ```bash
   $ git branch
   ```

   应该是`meta-content`

2. 然后就是一套组合拳：

   ```bash
   $ git checkout -b my-next-post-topic
   $ hexo new post "My Next Post Topic"
   $ git add .
   $ git commit -m "started new topic"
   $ git push -u origin my-next-post-topic
   ```

3. 当我们确认写好这篇文章以后，再把这个分支合并回`meta-content`去，最后再运行

   ```bash
   $ hexo generate
   $ hexo deploy
   ```



这个办法的好处就是将github既当成离线备份，也当成同步机制，岂不美哉？
