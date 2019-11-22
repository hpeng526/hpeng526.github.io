---
title: 利用Docker写Hexo文章
date: 2016-11-08 21:36:48
tags: [Docker, Hexo]
---

 因为，每一次换环境都要重新弄 Hexo 的环境，才能愉快的写文（wu）章（liao）。于是乎，想到肯定有人把环境弄成 Docker 的，挂在目录就行了。这不，一搜就搜到了一个镜像。 [simplyintricate/hexo](https://hub.docker.com/r/simplyintricate/hexo/)

### 准备

0. 在你的 GitHub pages 项目另外创一个分支，用来存放你的 Hexo 源码。
0. Docker 环境

<!-- more -->

### 正式开始

0. 由于不需要改动镜像里面的东西，就不重新 build 特定的 image 了。直接
    ```bash
    docker pull simplyintricate/hexo
    ```
    当然，国内我喜欢用 daocloud 的加速服务
    ```bash
    dao pull simplyintricate/hexo
    ```

0. 将源码 clone 下来
    ```bash
    git clone -b [你的分支] [你的地址.git]
    ```
    这里有个坑，就是如果你的 theme 依赖别人的 git 项目，他是没有被 clone 下来的，这时候需要手动去 clone ，还有一些 theme 的 config 也没有定制好，这里要自己去配置一下。

0. 挂载目录
    0. `/root/.ssh/` 如果需要部署，就挂载
    0. `<hexo source dir>` 这个不用说了
    0. `<hexo themes dir>` 主题地址
    0. `<hexo _config.yml>` 配置一定要挂载
    0. `<hexo package.json>` 如果你的版本跟镜像不一致，就需要挂载上去。
    0. 所以最后的启动命令看起来是这样的
        ```bash
        docker run --name hexo -it -p 4000:80 -d -v <hexo dir>/source:/usr/share/nginx/html/source -v <hexo dir>/themes:/usr/share/nginx/html/themes -v <hexo dir>/_config.yml:/usr/share/nginx/html/_config.yml -v <hexo dir>/package.json:/usr/share/nginx/html/package.json simplyintricate/hexo
        ```

0. 访问 4000 端口就能看到效果了

0. 与容器交互
    0. `docker exec -it hexo hexo generate` 重新生成
    0. 主要是这条命令啦`docker exec` 剩下就是执行 bash 命令了。[doc Docker-exec](https://docs.docker.com/engine/reference/commandline/exec/)

### 结语
Docker 实在是太方便了。Docker 实在是太方便了。Docker 实在是太方便了。

### 后话
其实可以 build 一个专门的镜像，扔到 CI 上去，每当 [你的源码] 分支有更新，就自动构建，自动部署。（特么的，真想转运维了。好玩）
