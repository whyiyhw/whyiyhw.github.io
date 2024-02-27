---
title: 本地大模型在obsidian中的实践
tags:
  - ai
  - ollama
  - openai
  - obsidian
categories: ai
abbrlink: 57e41d6f
date: 2024-02-26 19:33:32
---

## 概念解释
> Obsidian: 一种支持多平台的知识管理和笔记应用，它允许用户创建、编辑和链接他们的笔记，支持Markdown格式，可以帮助用户更好地组织和查找他们的知识。

>Ollama: 是一个 local AI 工具，它可以在本地运行，并提供与openai 类似的API，无需联网便能实现强大的计算和数据分析功能。

>**2B**/**7B** : 指的是模型的大小，2B 指的是20亿参数模型，7B 指的是70亿参数模型。这些参数决定了模型的复杂度和处理能力。

> GitHub Copilot : 是一个由GitHub和OpenAI共同开发的人工智能编程助手。它可以根据你的代码和注释，自动生成代码建议，帮助你更快地编写代码。
## 0x01 起源

本篇将会介绍，如何在 Obsidian 中，通过 `ollama` / `openai` 的能力进行文章编写。这个想法的来源是 GitHub Copilot 中的 idea。

在软件开发中，GitHub Copilot 已经展示了其强大的编程辅助能力，那么为何我们不能在文章编写中也利用到类似的技术呢？这就是我想在 Obsidian 中利用 ollama 或 openai 能力进行快速文章编写的初衷。

首先，我们在 GitHub 上找到了一个 插件 [Local GPT](https://github.com/pfrankov/obsidian-local-gpt)提供了 Obsidian 链接 到 `OpenAI`&`ollama` 的功能，这使我们可以直接在 Obsidian 中调用AI的能力。这个插件的使用非常简单，只需要简单的安装和配置，就可以开始我们的AI写作旅程了。

## 0x02 本地`model`VS`openai`

> 最大的问题就在于**隐私和数据安全**这一方面。虽然OpenAI已经采取了严格的数据安全措施，但数据仍然需要通过网络发送，而OpenAI本身也会存储这些数据。

> **成本**是另一个需要考虑的问题。当本地模型的硬件足够时，我们最多需要考虑的是电费，但OpenAI 的 gpt4 api 的价格并不便宜。

> 在谈到**可定制性**时，这需要你具备一定的开发能力，以及对硬件设备、数据采集和模型训练的经验。然而，随着开源产品的不断迭代和改进，这种要求将会变得更为普遍且操作更为简便。
### ollama

通过 [ollama](https://ollama.ai/) 下载安装，然后根据你的电脑配置选择合适的模型。如果你的显存是6G以下，最好选择2B左右的模型，而如果你有8G或12G的显存，那么7B的模型对你来说已经没有什么压力了。
至于模型的能力，确实是一分算力一分货。模型的大小与训练时的文本质量，直接影响了生成的文本质量，更大的模型通常能生成更准确，更自然的文本。
### OpenAI

对于`OpenAI`，我们需要通过其API进行调用，并且需要注意API的调用次数和费用。我们也需要注意个人敏感信息的安全。虽然 OpenAI 的计算能力强大，但使用它需要一定的编程基础和对API 的使用与限制（内容安全，上下文大小，prompt 工程）有所了解。由于它是云端服务，所以在数据安全方面也存在一定的风险。

我本身是通过 `GitHub Copilot` 购买了一年的服务，然后通过 [https://github.com/whyiyhw/copilot-gpt4-service](https://github.com/whyiyhw/copilot-gpt4-service) 开放了 `GPT-4 API`，因此我可以无限制地使用API，而不用担心账单问题。我本身也阅读了 `Local GPT` 的源码，并确认了使用它是无风险的。其他使用者需要自行关注这些点。

## 0x03 如何使用

一旦你完成了上述的安装和配置，你就可以开始你的AI协作编辑之旅了。

我自己定义了一些常用的功能，这些功能可以通过 `Local GPT > Add new manually` 来加入。这些功能包括**续写**，**总结**，**翻译**，**语句语法检查**，**自动给文章加标签**，以及在 obsidian 中特色的**出链**，**画 Mermaid 图**，和**从文本中列出任务**等。

接下来，我将简单介绍其中的两个功能。
### 续写

你可以在 Obsidian 中创建一个新的文档，然后开始输入你的想法或者草稿，接着，你只需要按下特定的快捷键（比如`alt + \` 我定义的），ai 就会自动为你生成一段接续的文本， 并且这段文本会紧接着你的原始内容，形成一个完整的故事或者观点。这样你就可以无缝地继续你的写作，而不需要担心受到创作阻塞的困扰。
【ps: 🌟其实算是偷懒，大致写出骨架，AI 补全剩下的内容, 不用太关注细节，更加流畅。】

![/img/post/article_ollama_obsidian_01](/img/post/article_ollama_obsidian_01.png)

### 翻译

> 通过对指定部分的选中，你可以将其翻译为任何你需要的语言。
> By selecting the specified part, you can translate it into any language you need.
![[Pasted image 20240227141409.png]]
![/img/post/article_ollama_obsidian_02](/img/post/article_ollama_obsidian_02.png)

## 0x04 结论

通过以上的介绍和讨论，我们可以看出，利用 AI 技术，特别是 `ollama` 和 `OpenAI`。极大地增强了 Obsidian 作为文本编辑器的能力，可以大大提高我们的写作效率和质量 , 它们可以帮助更好地增强文本的连贯性和吸引力，同时，也能激发我们的创新思维，提升我们的写作技巧。

最重要的是，这可以提升我持续输出的能力。**write less , do more**。

## 0x05 扩展小问题？

1. 我可以在哪里下载和安装 `ollama` 和 如何获取 `openai` token ？
2. 如何在 Obsidian 中配置和使用 `ollama` 和 `openai`？
3. `ollama` 和 `openai` 有什么区别和优势，该如何选择？