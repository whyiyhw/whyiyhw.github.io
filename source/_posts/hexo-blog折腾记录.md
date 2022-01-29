---
title: hexo-blog折腾记录
date: 2019-01-29 11:03:50
updated: 2020-04-07 19:19:24
categories: blog
tags: [hexo, git, fluid]
---

## 安装折腾总览

目前的 blog 折腾经过了
- ~~自己用 PHP 写 自建，样式太丑，放弃（17年）~~
- ~~Github page 国内访问速度太慢，加上 next 的样式太素（19年）~~ 
- 最后改成 fluid 这个样式，page 换到 Gitee（20年）

### 目录也分为三块

- hexo 安装
- hexo 的主题 next 与 fluid 的修改
- 最后就是 过程中碰到的一些问题
  - 样式与生成出来文件的分支管理？
  - 多主机如何进行发布？
  - git 如何超高速拉取 github 仓库的文件？

## HEXO 基础安装流程

- `npm` 跟 `cnpm` 的区别就不说了

    ```shell
    `cnpm install hexo -g` #安装 `hexo` 
    `hexo -v`  #查看版本
    ```
    
    ```shell
    `cnpm install -g hexo-cli` 				# 安装 `hexo` 命令行
    `hexo init dir` 						# 初始化目录
    `hexo new post "关于hexo的安装的一些记录"` # 新建文件
    `cnpm install hexo-server --save`  		# 更新预览服务
    `hexo server`							# 启动服务，可访问 http://localhost:4000 进行预览
    `hexo generate --deploy`			    # 编译生成静态文件，并上传至 git 服务上
    `hexo d -g`                             # 可缩写
    ```

## hexo 的主题 next 与 fluid 的修改`hexo`  进阶 `next `

- next 主题相关
```shell
  cd themes # 进入主题目录
  git clone https://github.com/iissnan/hexo-theme-next themes/next #克隆主题文件
  // 主要参考 : https://www.jianshu.com/p/9f0e90cc32c2
```

- fluid 主题相关
```shell
    cd themes # 进入主题目录
    git clone https://github.com/fluid-dev/hexo-theme-fluid.git    #克隆主题文件
    // 主要参考 : https://hexo.fluid-dev.com/ 进行配置
```

## 过程中可能碰到的一些问题

### 样式与生成出来文件的分支管理？

- 需要 新建一个 hexo 分支 来避免 master 主分支被覆盖 ，主分支 master 为静态文件目录 ，hexo 分支为 样式与配置文件目录

### 多主机如何进行发布？

- `git checkout hexo` 拉下代码后 切换分支  至配置生成目录

```shell
cnpm install hexo 					  # 重新安装 hexo
cnpm install 						  # 重新拉所需的运行所需文件
cnpm install hexo-deployer-git --save # 重新生成 钩子才能上传成功
```

### git 设置 https 代理的正确姿势?

- `git config --global http.https://github.com.proxy socks5://127.0.0.1:1080` 仅设置 github 在 https 端口的 代理
- 前提条件，你的 SS 开了，而且本地端口为 1080
- git 全局 设置代理
    ```shell
    git config --global http.proxy 'socks5://127.0.0.1:1080' # http全局设置代理
    git config --global https.proxy 'socks5://127.0.0.1:1080'# https全局设置代理
    git config --global --unset http.proxy # http全局取消代理
    git config --global --unset https.proxy# https全局取消代理
    ```

###  搭建 hexo，在执行 hexo deploy 时,出现 ERROR Deployer not found: git 的错误

- `cnpm install hexo-deployer-git --save` #重新生成 钩子才能上传成功

### hexo 怎么删除文章？
-  先使用 hexo clean 再删除
-  最后再重新 hexo generate --deploy 生成就OK

### 如何让 百度收录
- https://ziyuan.baidu.com/linksubmit/url 自行提交

**如果想自行安装，请学习检索**

-  技能点需求 难度 ♥ ♥
  - `git`、 `hexo` 、`npm`、 `md` 、`html/文件目录` 相关基础知识  	
-  这样就能继续愉快的写`blog`了

**参考** 
[hexo官方文档](https://hexo.io/zh-cn/docs/)
[fluid官方文档](https://hexo.fluid-dev.com/)