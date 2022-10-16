---
title: 生成式AI-图像类-stable-diffusion-webui（概述）
tags:
  - ai
categories: 图像生成
index_img: /img/post/04.png
abbrlink: b93583de
date: 2022-10-16 23:12:32
---

最近发现B站上的这个话题很火，就去找了下对应的库[https://github.com/AUTOMATIC1111/stable-diffusion-webui](https://github.com/AUTOMATIC1111/stable-diffusion-webui) 发现好家伙，2022-8月 开源，两个月时间斩获了13000多颗⭐，让我们看看这个库到底做了什么。

## windows 下的安装
1. 安装 `Python > 3.10.6` 安装  `git`
	1. 国内用户pip 没走代理的，在 `C:\Users\xxx\pip` 新建 pip 目录，新增php.ini 配置全局的代理
```shell
[global]
timeout =6000
index-url = https://mirrors.aliyun.com/pypi/simple/
extra-index-url=https://pypi.tuna.tsinghua.edu.cn/simple/
https://pypi.mirrors.ustc.edu.cn/simple/
https://pypi.douban.com/simple
[install]
trusted-host = mirrors.aliyun.com
https://pypi.tuna.tsinghua.edu.cn/simple/
https://pypi.mirrors.ustc.edu.cn/simple/
https://pypi.douban.com/simple
```

2. `git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui.git`
3. 下载训练好的官方模型 [https://github.com/AUTOMATIC1111/stable-diffusion-webui/wiki/Dependencies](https://github.com/AUTOMATIC1111/stable-diffusion-webui/wiki/Dependencies)

现在你有了 `sd-v1-x.ckpt` 这个 tensorflow 生成的模型 , 还有运行这个模型所需的软件，把模型改名为 `model.ckpt`  放入 `stable-diffusion-webui\models\Stable-diffusion`
目录下，windows 打开 powershell 运行  `webui-user.bat` 
第一次加载会比较慢，因为要下载很多的依赖 , 看到下面这个的时候就说明安装完成了
```txt
Running on local URL:  http://127.0.0.1:7860
```
进入 `127.0.0.1:7860` 最主要的两个功能 `txt2img` `img2img` 我们上手测试下生成的效果

![16.png](/img/16.png)

简单测试了几个词后，我的感觉就是这个算法对初学者不太友好，我评分在(30-70)左右，而且会出现奇怪的图像，那他怎么在B站爆火的呢？ 

我注意到了另一个关键词 `NOVELAI` , 在维基百科中的解释为`NovelAI是Anlatan的深度学习人工智能服务，其下有辅助故事写作以及文字作图像生成，采取订阅制的云端运算服务`
这家公司对于 日漫风格的 文字生成图片做的很不错。前几天一个 叫 novelaileak 的种子在疯传，泄露出来的这个模型，配合上这个web-ui, 实现了 普通用户用自己电脑就是生成 8分 动漫图的效果，配合上 img2img 对 网络热门人员的漫画化改造，热度不错。开源软件 + 模型 + 电脑，产生的生产力对中低端画师有了明显的压制，所以这个话题不断升温。
我找到了 这个[种子链接]([magnet:?xt=urn:btih:5bde442da86265b670a3e5ea3163afad2c6f8ecc](magnet:?xt=urn:btih:5bde442da86265b670a3e5ea3163afad2c6f8ecc)) ,以及对应的[安装指导](https://rentry.org/nai-speedrun) 种子过于庞大 完整模型 50G，实际上我们只用 到 5G 左右的模型，各位自行下载，把模型改名后移入 到 `models\Stable-diffusion` 无需重启，直接在左上方刷新切换模型就OK

随便找个模板看看效果
`{photorealistic}, 1 girl, gothic maid dress,{blue Hair},{pink Hair accessories},Sky, grassland,red eyes, depth of field,cinematic angle ,backlighting,young girl,`

![/img/post/04.png](/img/post/04.png)


可以看出来，对比原始模型 `animefull-final` 这个模型在生成动漫人物方面的效果 可以达到 70-80 的水准，不加入特定的作画风格，商用问题不大，这就触动了B站很多画师的利益, 自然就有了热度。

### 在程序员职业角度来如何看待这个软件？

我觉得是生成式AI比我想象中会更快商业化落地
- 实现了 UI展示软件与模型的分离，分工协作是大规模工业化的第一步
- 模型对于机器与硬件的要求，下降到了民用机器也能很好去实现的地步，我这台 I5-8代-16G-1060 的机器虽然生成一张普通的图需要10秒左右，但也能用了
- 熟悉了软件的机制与对应的关键词之后，大部分场景都是可以做到自动生成
软件可以改进的点，加入软件版本机制，升级需要下载最新的版本然后覆盖很容易出问题，稳定后多一些用户使用文档

img2img 先不做演示了，copy（学习）绘画大佬的风格，生成自己风格这部分，机器性能太差也不做说明了

这里只是做一个简单介绍，后续会把详细的使用说明，跟更好的安装方式完整的写出来

不管怎么说，我的封面以后不用愁了~
