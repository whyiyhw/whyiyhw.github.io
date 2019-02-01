---
title: 关于hexo的安装的一些记录其一
date: 2019-01-29 11:03:50
categories: hexo的一些设定
tags: 
- hexo
---

## 基础安装流程
  - 安装 `hexo` 		`yarn add  hexo -g`
  - `hexo -v`
  - 安装 `hexo` 命令行 `npm install -g hexo-cli`
  - `cd test/`
  - `hexo init`
  - `hexo new post "关于hexo的安装的一些记录"`
  - `yarn add hexo-server --save`
  - `hexo server`
  - `hexo generate --deploy`
## hexo 进阶next
```
	mkdir themes/next
	git clone https://github.com/iissnan/hexo-theme-next themes/next
  // 主要参考 : https://www.jianshu.com/p/9f0e90cc32c2
```