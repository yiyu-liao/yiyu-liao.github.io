---
layout: post
title:  "利用git hooks快速部署node服务"
date:   2020-03-27
catalog: true
tags: Deploy
---

Automate Deployment for NodeJS Service with git hooks

## 前言

最近在写Nodejs后台服务，需要部署到腾讯云测试。最初的工作流是将最新代码push到git远程repo, 然后ssh连接腾讯云，输入密码，然后git pull更新服务器代码, 最后跑npm命令重启服务。

后来发现这种工作流太繁琐，所以想要寻找一个比较高效的方法避免重复的工作。

## 思路

一开始想到的是docker jenkins 部署，但是针对项目本身，这些部署太重了，我需要的是轻量，快速更新的方式。

于是选择了服务器创建git仓库，相当于git的remote repo。本地开发环境关联服务器仓库, 有代码更新时候，直接把代码push到服务器git仓库，
触发git-hooks实现自动化部署。

## 实现步骤

#### step 1

创建一个空仓库，路径可自行定义。什么是bare repo? 可以参考stackoverflow的[解析](https://stackoverflow.com/questions/5540883/whats-the-practical-difference-between-a-bare-and-non-bare-repository/28428742#28428742);

```
git init --bare /root/deploy/app.git
```

#### step 2

创建存放代码代码的路径, 我把代码放到了`/root/deploy/code`。为什么需要进行这个操作，贴出stackoverflew的一个解析：

> Presumably, when it creates a bare repository, Git assumes that the bare repository will serve as the origin repository for several remote users, so it does not create the default remote origin. What this means is that basic git pull and git push operations won't work since Git assumes that without a workspace, you don't intend to commit any changes to the bare repository:

```
git clone /root/deploy/app.git /root/deploy/code
```

#### step 3

编写hook脚本, 在此我选择了post-receive，其他hooks可参考 [git hooks]([https://www.digitalocean.com/community/tutorials/how-to-use-git-hooks-to-automate-development-and-deployment-tasks])


```
cd /root/deploy/app.git/hooks

nano post-receive

```

脚本需要做的是，确定git上下文，切换到develop分支，安装依赖，执行npm命令。我使用了pm2进程管理node服务，具体如下：

```bash
#!/bin/bash
echo ‘post-receive: Triggered.’
cd /root/deploy/code
echo ‘post-receive: git check out…’
git --git-dir=/root/deploy/app.git --work-tree=/root/deploy/code checkout develop -f
echo ‘post-receive: npm install…’
npm install \
&& echo ‘post-receive: → done.’ \
&& (pm2 delete dev || true) \
&& pm2 start 'npm run start' --name dev -- start \
&& echo ‘post-receive: app started successfully with pm2.
```

保存，然后执行`chmod ug+x ./post-receive`让脚本生效。


#### step 4

在本地的开发环境关联服务器的git仓库

```
git remote -v

git remote add auto-deploy ssh://root@123.123.123.123/root/deploy/app.git

```

`auto-deploy`为你的远程服务器仓库标识名称，`123.123.123.123`为你的服务器ip地址。

关联后，运行`git push auto-deploy develop`代码将会更新到服务器，这时候就会触发hook, 跑脚本实现自动构建了～～