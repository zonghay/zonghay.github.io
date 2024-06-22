---
title: "ChatGPT Prompt Summary of Skills"
date: 2024-06-13T12:58:57+08:00
lastmod: 2024-06-13T12:58:57+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- ChatGPT
description: ""
weight:
slug: ""
draft: false # 是否为草稿
comments: true # 本页面是否显示评论
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hideMeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
hideSummary: false
ShowWordCount: true
ShowBreadCrumbs: true #顶部显示路径
cover:
    image: "" #图片路径例如：posts/tech/123/123.png
    zoom: # 图片大小，例如填写 50% 表示原图像的一半大小
    caption: "" #图片底部描述
    alt: ""
    relative: false
---

## First Of All

**This is important which the official docs is later and more detailed than summary ⬇️⬇️⬇️️**

[https://platform.openai.com/docs/examples](https://platform.openai.com/docs/examples)

## To do & Not to do

> Instead of just saying what not to do, say what to do instead.

与其告知模型不能干什么，不妨告诉模型能干什么。

|  场景   | Less Effective  |  Better   |  原因   |
|  ----  | ----  |-----|-----|
| 推荐雅思必背英文单词  | Please suggest me some essential words for IELTS |  Please suggest me 10 essential words for IELTS   |  后者 prompt 会更加明确，前者会给大概 20 个单词。这个仍然有提升的空间，比如增加更多的限定词语，像字母 A 开头的词语。   |
| 推荐香港值得游玩的地方  | Please recommend me some places to visit in Hong Kong. Do not recommend museums. |  Please recommend me some places to visit in Hong Kong including amusement parks.   |  后者的推荐会更准确高效一些，但如果你想进行一些探索，那前者也能用。   |

## 基于示例回答

如果你无法用文字准确解释问题或指示，你可以在 prompt 里增加一些案例：
```
Suggest three names for an animal that is a superhero.

Animal: Cat
Names: Captain Sharpclaw, Agent Fluffball, The Incredible Feline
Animal: Dog
Names: Ruff the Protector, Wonder Canine, Sir Barks-a-Lot
Animal: Horse
Names:
```
增加例子后，Output 的结果会更接近我想要的那种风格的名字。

```
Gallop Guardian, Equine Avenger, The Mighty Stallion
```

## 使用引导词

在代码生成场景里，有一个小技巧，上面提到的案例，其 prompt 还可以继续优化，在 prompt 最后，增加一个代码的引导，告知 AI 我已经将条件描述完了，你可以写代码了。    
在 prompt 的最后增加 SELECT 可以很好地提示 AI 可以写 SQL 代码了。Better prompt： 

```
Create a MySQL query for all students in the Computer Science Department:
Table departments, columns = [DepartmentId, DepartmentName]
Table students, columns = [DepartmentId, StudentId, StudentName]
SELECT
```

## 增加 Role

尝试让 AI 假设自己是一个小学老师，并且很擅长将复杂的内容转变成 7、8 岁小朋友也能听懂的话，然后再改写这段话：
```
You are a primary school teacher who can explain complex content to a level that a 7 or 8 year old child can understand. Please rewrite the following sentences to make them easier to understand:
OpenAI is an American artificial intelligence (AI) research laboratory consisting of the non-profit OpenAI Incorporated (OpenAI Inc.) and its for-profit subsidiary corporation OpenAI Limited Partnership (OpenAI LP). OpenAI conducts AI research with the declared intention of promoting and developing a friendly AI. OpenAI systems run on the fifth most powerful supercomputer in the world.[5][6][7] The organization was founded in San Francisco in 2015 by Sam Altman, Reid Hoffman, Jessica Livingston, Elon Musk, Ilya Sutskever, Peter Thiel and others,[8][1][9] who collectively pledged US$1 billion. Musk resigned from the board in 2018 but remained a donor. Microsoft provided OpenAI LP with a $1 billion investment in 2019 and a second multi-year investment in January 2023, reported to be $10 billion.[10]
```
Output 就变成了下面这样。不管是用词，还是语法，看起来都更容易懂了。
```
Sure, let's break this down into simpler concepts:

OpenAI is a place where smart people work on making computers really clever. They want computers to be helpful and safe for everyone. It's like a big science club that has two parts: one part that doesn't want to make money, and another part that does.

This club works with a super-duper powerful computer—one of the strongest in the whole world—to learn more about how to make computers smart.

OpenAI was started in a city called San Francisco by a group of friends in 2015. These friends gave a lot of money, like a whole billion dollars, to help start the club. One of the friends who helped start it is named Elon Musk, but he decided to stop helping run it in 2018. He still gives them money, though.

A really big company named Microsoft also gave OpenAI a lot of money to help them with their work. They gave them one billion dollars in 2019 and even more money in 2023!
```

## 使用特殊符号将指令和需要处理的文本分开

可以用“”“将指令和文本分开，这样会提升 AI 反馈的准确性（这个技巧来自于 OpenAI 的 API [最佳实践文档](https://help.openai.com/en/articles/6654000-best-practices-for-prompt-engineering-with-the-openai-api)）

像我们之前写的 prompt 就属于 Less effective prompt。为什么呢？据我的测试，主要还是 AI 不知道什么是指令，什么是待处理的内容，用符号分隔开来会更利于 AI 区分。
Better prompt:
```
Please summarize the following sentences to make them easier to understand.

Text: """
OpenAI is an American artificial intelligence (AI) research laboratory consisting of the non-profit OpenAI Incorporated (OpenAI Inc.) and its for-profit subsidiary corporation OpenAI Limited Partnership (OpenAI LP). OpenAI conducts AI research with the declared intention of promoting and developing a friendly AI. OpenAI systems run on the fifth most powerful supercomputer in the world.[5][6][7] The organization was founded in San Francisco in 2015 by Sam Altman, Reid Hoffman, Jessica Livingston, Elon Musk, Ilya Sutskever, Peter Thiel and others,[8][1][9] who collectively pledged US$1 billion. Musk resigned from the board in 2018 but remained a donor. Microsoft provided OpenAI LP with a $1 billion investment in 2019 and a second multi-year investment in January 2023, reported to be $10 billion.[10]
"""
```


## 通过示例阐明所需的输出格式

Less effective ❌:
```
Extract the entities mentioned in the text below. Extract the following 4 entity types: company names, people names, specific topics and themes.

Text: {text}
```

Better ✅:
```
Extract the important entities mentioned in the text below. First extract all company names, then extract all people names, then extract specific topics which fit the content and finally extract general overarching themes

Desired format:
Company names: <comma_separated_list_of_company_names>
People names: -||-
Specific topics: -||-
General themes: -||-

Text: {text}
```

**Even More**

```
Summarize the main points of the following speech
Use the following format:
Topic 1: <topic_name_1>
- <point_1>
..
Topic 2: <topic_name_2>
- <point_1>
..
Topic 10: ..

Text: """
Thank you so much, Fred, for that lovely introduction. And thanks to the Atlantic Council for hosting me today.

The course of the global economy over the past two years has been shaped by COVID-19 and our efforts to fight the pandemic. It’s now evident, though, that the war between Russia and Ukraine has redrawn the contours of the world economic outlook. Vladimir Putin’s unprovoked attack on Ukraine and its people is taking a devastating human toll, with lives tragically lost, families internally displaced or becoming refugees, and communities and cities destroyed.
...

"""
```
Output 是这样的：

```
Topic 1: The war in Ukraine
- The war is taking a human toll with lives lost, families displaced, and communities destroyed
- The Biden administration is committed to holding Russia accountable
- The war has violated international law and is a challenge to the international order

Topic 2: The global economy
- The war is having negative impacts on the global economy, including higher commodity prices and inflation
- The IMF and World Bank will be focused on helping developing countries weather the impacts of the war
- The ultimate outcome for the global economy depends on the path of the war
```

## Zero-Shot Chain of Thought

这个技巧使用起来非常简单，只需要在问题的结尾里放一句 **Let‘s think step by step** （让我们一步步地思考），模型输出的答案会更加准确。
[https://learningprompt.wiki/zh-Hans/docs/chatGPT/tutorial-tips/tip-7-zero-shot-chain-of-thought](https://learningprompt.wiki/zh-Hans/docs/chatGPT/tutorial-tips/tip-7-zero-shot-chain-of-thought)

## Few-Shot Chain of Thought

> 通过向大语言模型展示一些少量的样例，并在样例中解释推理过程，大语言模型在回答提示时也会显示推理过程。这种推理的解释往往会引导出更准确的结果。

## CRISPE Prompt Framework

CRISPE 分别代表以下含义：

* CR： Capacity and Role（能力与角色）。你希望 ChatGPT 扮演怎样的角色。
* I： Insight（洞察力），背景信息和上下文（坦率说来我觉得用 Context 更好）。
* S： Statement（指令），你希望 ChatGPT 做什么。
* P： Personality（个性），你希望 ChatGPT 以什么风格或方式回答你。
* E： Experiment（尝试），要求 ChatGPT 为你提供多个答案。

![图片](/images/gpt_freamwork.png)

# 参考 

[Learning Prompt](https://learningprompt.wiki/zh-Hans/docs/chatGPT/tutorial-extras/chatGPT-prompt-framework)