---
title: 关于hexo的安装的一些记录其一
date: 2019-01-29 11:03:50
categories: hexo的一些设定
tags: 
- hexo
---

## 基础安装流程

- 安装 `hexo` `cnpm install hexo -g`
  - `hexo -v`
- 安装 `hexo` 命令行 `cnpm install -g hexo-cli`
  - `cd test/`
  - `hexo init`
  - `hexo new post "关于hexo的安装的一些记录"`
  - `cnpm install hexo-server --save`
  - `hexo server`
  - `hexo generate --deploy`

## `hexo`  进阶 `next `

```shell
  mkdir themes/next
  git clone https://github.com/iissnan/hexo-theme-next themes/next
  // 主要参考 : https://www.jianshu.com/p/9f0e90cc32c2
```

## 多主机发布

- 拉下代码后 切换分支 `git checkout hexo`
- 主要原因还是对与 `js` 的工具链不熟悉

  ```js
  cnpm install hexo
  cnpm install
  cnpm install hexo-deployer-git --save
  ```

- 这样就能继续愉快的写`blog`了
- 进一步参考（未实现）
- [hexo花哨的进阶](https://reuixiy.github.io/technology/computer/computer-aided-art/2017/06/09/hexo-next-optimization.html#fn:2)