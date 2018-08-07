---
title: travis-ci 部署 hexo
date: 2018-01-18 22:10:00
tags: [travis, hexo]
---

利用 travis-ci 自动生成 hexo 并提交，无非就几点要注意的，一个是 travis-ci 怎么有权限推送，另一个就是 travis-ci 怎么用。

<!-- more -->

# 从 `.travis.yml` 入手

下面是我的配置，很简单。

```yaml
language: node_js
node_js: stable

install:
  - npm install

script:
  - hexo cl
  - hexo g

after_script:
  - cd ./public
  - git init
  - git config --local user.name "hpeng526"
  - git config --local user.email "hpeng526@gmail.com"
  - git add --all .
  - git commit -m "update"
  - git push -f https://$GH_TOKEN@github.com/hpeng526/hpeng526.github.io.git master

branches:
  only:
    - src
```

要注意的点

- $GH_TOKEN 是 github 的 Personal access token ，生成之后，在 travis 内配置环境变量，这样才能保密
- branchs.only 是指触发的分支，这里我的分支是 src ，提交源码时候注意忽略 node 生成的文件就好了

# 主题配置

直接用 release 配置好提交到自己的仓库，省去其他麻烦。
