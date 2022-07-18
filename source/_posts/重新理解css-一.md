---
title: 重新理解css(一)
categories: 前端相关
tags:
  - css
abbrlink: d5681d32
date: 2019-02-12 20:32:40
---
# CSS
    - 全称 层叠样式表 Cascading Style Sheet
## 选择器
    - 分类和权重
        - 元素选择器   a{}
        - 伪元素选择器 ::before{}
        - 类选择器    .link{}
        - 属性选择器  [type=radio]{}
        - 伪类选择器  :hover{}
        - ID选择器    #id{}
        - 组合选择器  [type=checkbox] + label{}
        - 否定选择器  :not(.link){}
        - 通用选择器  *{}
        - 权重
            - ID选择器 #id{} + 100
            - 类 属性 伪类 +10
            - 元素 伪元素 +1
            - 其它选择器 +0
            - 权重不会进位 11 个类 的权重 也比 ID 选择器 小
        - !important 优先级最高
        - 元素属性（内部关联写法） 优先级高 但是 没有!important 优先级高
        - 相同的权重 后写的生效（覆盖）
    - 解析方式和性能
        浏览器 是 从右往左去 解析的
## 非布局样式
    - 字体、字重、颜色、大小、行高
    - 背景、边框
    - 滚动、换行
    - 粗体、斜体、下划线  
    - 其他
    ---
    - 字体
        - 字体族
        - 多字体 fallback
        - 网络字体，自定义字体
        - iconfont 原理实际上是自定义字体
    - 行高
        - 行高的构成
            - inline-box 的高度 决定行高
            - 行高变大，会导致外层元素高度变高，但是文字排版渲染高度不会变（仅与字体大小有关）
        - 行高相关的现象
            - 垂直居中
                - 设定 height-align 和外层元素高度一致就行
            - 文字对齐
                - 设定 vertical-align: middle
            - 图片 3px 空隙问题
                - 解决方案 设定 vertical-align: bottom
    - 背景
        - 背景颜色
            - background: #f00 | hsl(230,100%,50%,0.3) | rgba(255,0,0,0.3)
            - background:url(./test.png)
        - 渐变色背景
            - background: linear-gradient(90deg, red, yellow 80%, blue) 线性渐变
        - 多背景叠加
        - 背景图片和属性（雪碧图）
        - base64 和性能优化
        - 多分辨率适配
            - 把 大图片 通过 background-size 来缩小 之后 在高分辨屏幕下查看 不会模糊