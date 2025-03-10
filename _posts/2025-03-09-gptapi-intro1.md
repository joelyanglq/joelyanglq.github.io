---
title: '（一）GPT API Guide: '
date: 2025-03-09
permalink: /posts/2025/03/LLM/
tags:
  - GPT
  - API Guide
---

> 本系列计划包含4个帖子：
> 1. [(一) GPT's API Guide: Setup and how to get reponse from GPT's API]()
> 2. [(二) GPT's API Guide: Function call]()
> 3. [(三) GPT’s API Guide: Finetuning using OpenAI]()
> 4. [(四) GPT’s API Guide：LangChain and RAG Intro]()
>
> 是对 **Developing Apps with GPT-4 and ChatGPT Build Intelligent Chatbots, Content Generators, and More (Olivier Caelen, Marie-Alice Blete) ** 的一个笔记。书中还提到了其他的特性，但由于不再常用或者作者暂时不知道其用处的原因，只会在文章中一笔带过，比如第50页起的道德审查模型，第5张介绍的插件功能。

## Setup

先从官网获取API key，一个以'sk'开头的长字符串。通常不建议明文使用API key，一个安全的使用办法是将API key设置成环境变量。在Linux或Mac上可以如此处理

```cmd
# set environment variable OPENAI_API_KEY for current session
export OPENAI_API_KEY=sk-(...)
# check that environment variable was set
echo $OPENAI_API_KEY
```

在Windows上
```cmd
# set environment variable OPENAI_API_KEY for current session
set OPENAI_API_KEY=sk-(...)
# check that environment variable was set
echo %OPENAI_API_KEY%
```

为了永久的设置环境变量，你需要`windows+R`进入`运行框`，随后输入`sysdm.cpl`设置环境变量。

需要使用时，在Python中加载API key
```
# Load your API key from file
openai.api_key_path = <PATH>, 

# Load your API key 
openai.api_key = os.getenv("OPENAI_API_KEY")
```

接下来，安装所需包
```
pip install openai
```

## 使用

```python
import openai

# For GPT 3.5 Turbo, the endpoint is ChatCompletion
openai.ChatCompletion.create(
    # For GPT 3.5 Turbo, the model is "gpt-3.5-turbo"
    model="gpt-3.5-turbo",
    # Conversation as a list of messages.
    messages=[
        {"role": "system", "content": "You are a helpful teacher."},
        {
            "role": "user",
            "content": "Are there other measures than time complexity for an algorithm?",
        },
        {
            "role": "assistant",
            "content": "Yes, there are other measures besides time complexity for an algorithm, such as space complexity.",
        },
        {"role": "user", "content": "What is it?"},
    ],
)
```

`model`和`message`是两个必选参数。其他常用参数包括：
- temperature: 默认1，接受[0, 2]。参考LLM解码策略。
- n：默认1.类似独立重复试验次数。对于这个prompt，生成多少次回答。
- stream：默认False。如果设置为True,则像网页版GPT那样一个一个吐字。
- `max_tokens`: integer. 生成的最大token数，用于控制成本。

API返回的格式长这样:
```python
{ 
	"choices": [ 
		{ 
			"finish_reason": "stop", 
			"index": 0, 
			"message": { 
				"role": "assistant", 
				"content": "Hello there! How may I assist you today?" 
			} 
		} 
	], 
	"created": 1681134595, 
	"model": "gpt-3.5-turbo", 
	"id": "chatcmpl-73mC3tbOlMNHGci3gyy9nAxIP2vsU", 
	"object": "chat.completion", 
	"usage": { 
		"completion_tokens": 10, 
		"prompt_tokens": 11, 
		"total_tokens": 21 
	} 
}
```

GPT生成的部分在
```python
# Extract the response
print(response["choices"][0]["message"]["content"])
```