---
title: "Agent Loop：让 Agent 自己跑起来"
date: 2026-04-07 09:06:59
categories:
    - 编程故事
tags:
    - 公众号迁移
wechat_url: "https://mp.weixin.qq.com/s/GXMgtuSwcwrtqqmb8Gr9mg"
---

上篇文章我们聊了 **Tool Use**——LLM 通过结构化的方式"点菜"，Agent 程序负责"上菜"。

但光有工具还不够，LLM 调了一次工具、拿到了结果，然后呢？如果任务还没完成，它还得继续调其他工具，对吧？

这个"继续"的过程，就是今天要聊的核心——**Agent Loop**。

**Anthropic 怎么说的**

Anthropic 在他们那篇很有影响力的文章《Building effective agents》中，对 Agent 有一个非常精辟的定义：

*Agents can handle sophisticated tasks, but their implementation is often straightforward. They are typically just LLMs using tools based on environmental feedback in a loop.*

*Agent 可以处理复杂的任务，但它们的实现往往很简单。本质上就是 LLM 在一个循环中，根据环境反馈来使用工具。*

注意这句话里的关键词：**in a loop**（在一个循环中）。

这就是 Agent 和普通 LLM 应用最本质的区别。普通的 LLM 应用是你问一句它答一句，流程是预先写死的。

而 Agent 是把控制权交给 LLM，由模型自己决定下一步做什么、调什么工具、什么时候停下来。

![](/img/wechat/57-agent-loop/01.png)

Anthropic 在文章中还特别强调了一点：Agent 的核心循环结构非常简单，关键在于模型的能力和工具的设计，而不是复杂的编排逻辑。

Claude Code 就是这个理念的直接产物。如果你用过 Claude Code，你应该能感受到：你给它一个任务，它会自己去读代码、改文件、跑命令，一步步推进，直到搞定。

背后驱动这一切的，就是 Agent Loop。

**从一个真实场景说起**

假设你在用 Claude Code 写代码，然后你说了一句：

*"帮我在项目里找到所有 TODO 注释，整理成一个清单。"*

这个任务对人来说很简单：打开搜索，输入 TODO，看看结果，整理一下。但对 Agent 来说，它需要经历一个完整的循环。

我们一步一步来看。

**循环的每一步**

**第一轮：搜索 src 目录**

你的任务发给 LLM 之后，LLM 会思考：要找 TODO 注释，我应该先用搜索工具在项目里搜一下。

于是它返回了一个 `tool_use`：

```json
{  
  "type": "tool_use",  
  "name": "search",  
  "input": { "pattern": "TODO", "path": "./src" }  
}
```

Agent 执行搜索，拿到结果：

```text
src/auth.js:42  // TODO: add token refresh logic  
src/api.js:18   // TODO: handle timeout errors  
src/utils.js:7  // TODO: refactor this function
```

Agent 把结果作为 `tool_result` 返回给 LLM。

**第二轮：还要不要继续搜？**

LLM 拿到搜索结果后，不是直接就给你答案了。

它会再想一下：找到了 3 个 TODO，但我只搜了 src 目录，是不是还应该看看测试目录？

于是它又返回一个 tool\_use：

```json
{  
  "type": "tool_use",  
  "name": "search",  
  "input": { "pattern": "TODO", "path": "./tests" }  
}
```

Agent 执行，发现测试目录里没有 TODO，把空结果返回给 LLM。

**第三轮：够了，生成回答**

这时候 LLM 判断信息已经足够了，不再调用工具，而是直接输出一段文字：

> "项目中共有 3 处 TODO 注释：
>
> 1. src/auth.js 第 42 行：需要添加 token 刷新逻辑
> 2. src/api.js 第 18 行：需要处理超时错误
> 3. src/utils.js 第 7 行：需要重构该函数"

此时 `stop_reason` 变成 `"end_turn"`，循环结束。

**整个流程画出来**

你再回头看一遍，其实就是一个循环：

**用户输入 → LLM 思考 → 需要工具？→ 是 → 执行工具 → 结果返回 LLM → LLM 再想 → 需要工具？→ 是 → …… → 不需要了 → 输出最终结果**

这就是所谓的 **Agent Loop**（智能体循环）。

![](/img/wechat/57-agent-loop/02.png)

用代码来表达的话，核心逻辑就是一个 while 循环：

```python
while true:  
    response = 调用 LLM(消息历史)  

    if response 包含 tool_use:  
        result = 执行工具(tool_use)  
        把 tool_result 加入消息历史  
        continue  // 继续循环  
    else:  
        输出最终回答  
        break     // 退出循环
```

没错，就这么简单。正如 Anthropic 说的：Agent 可以处理复杂的任务，但它的实现往往很简单。整个 Agent 最核心的调度逻辑，就是这几行伪代码。

**为什么这个循环这么重要**

没有这个循环，LLM 就只能做一问一答的事情。你问它一个问题，它凭自己脑子里的知识回答你，完事了。

但有了这个循环，事情就完全不一样了。LLM 可以**根据中间结果动态决定下一步做什么**。搜完代码发现有个文件可疑，那就再读一下这个文件；读完发现还需要看看 git 历史，那就再调一个命令；命令执行发现报错了，分析一下原因再试一次。

Anthropic 把这种模式叫做 **autonomous agent**（自主智能体），和那些流程写死的 workflow 做了明确区分。workflow 是你提前规定好第一步做什么、第二步做什么；而 agent 是让模型自己决定该怎么走。这种灵活性，正是 Agent Loop 带来的。

这也是为什么 Claude Code 处理一个稍微复杂点的任务，可能要跑几十轮循环。它不是一步到位的，而是像人一样，边做边想，一步步推进。

**循环中的关键细节**

看上去简单的循环，实际跑起来有几个重要的细节：

**消息历史是关键**

每一轮循环，LLM 的回复（包含 tool\_use）和工具的执行结果（tool\_result）都要追加到消息历史里。下一轮调用 LLM 时，要把完整的历史都带上。

为什么？因为 LLM 本身是无状态的，它不记得上一次说了什么、做了什么。只有你把历史消息全部喂给它，它才能知道："哦，我之前已经搜过 src 目录了，现在应该搜 tests 目录。"

消息历史就是 Agent 的"记忆"。

![](/img/wechat/57-agent-loop/03.png)

**stop\_reason 决定循环走向**

上篇文章我们提到过，LLM 的返回里有一个 stop\_reason 字段：

* `"tool_use"`：LLM 想调工具，循环继续
* `"end_turn"`：LLM 觉得完事了，循环结束

Agent 程序就是靠这个信号来决定：是继续跑，还是停下来把结果给用户。

**循环不能无限跑**

LLM 有时候会陷入奇怪的状态，比如反复调用同一个工具，或者在两个工具之间来回跳。所以实际的 Agent 都会设一个最大循环次数，超过了就强制停止。

这不是什么高深的技术，就是一个计数器加一个 if 判断。但不加的话，你的 Agent 可能会一直跑下去，token 费用也跟着蹭蹭涨。

**一个更复杂的例子**

上面找 TODO 的例子比较简单，只跑了两三轮。

来看一个更贴近真实场景的例子。

你跟 Claude Code 说："帮我修一下 login 接口的那个 bug，用户登录后 token 没有正确返回。"

Claude Code 可能会这样跑：

| 轮次 | LLM 决策 | 工具调用 | 结果 |
| --- | --- | --- | --- |
| 1 | 先找到 login 接口在哪 | `search("login")` | 找到 src/auth/login.ts |
| 2 | 读一下这个文件 | `read_file("src/auth/login.ts")` | 拿到源码 |
| 3 | 看看 token 生成逻辑 | `read_file("src/utils/token.ts")` | 发现 generateToken 函数 |
| 4 | 找到 bug 了，改代码 | `write_file("src/auth/login.ts")` | 修复完成 |
| 5 | 跑一下测试确认 | `bash("npm test")` | 2 个测试失败 |
| 6 | 看看测试为什么挂了 | `read_file("tests/auth.test.ts")` | 发现测试也需要更新 |
| 7 | 更新测试代码 | `write_file("tests/auth.test.ts")` | 修改完成 |
| 8 | 再跑一次测试 | `bash("npm test")` | 全部通过 ✅ |
| 9 | 任务完成，汇报结果 | 无（end\_turn） | 输出修复总结 |

9 轮循环，用了 4 种不同的工具，中间还经历了一次测试失败后的自我纠正。

![](/img/wechat/57-agent-loop/04.png)

这就是 Agent 的威力：它不是按固定脚本执行的，而是根据每一步的实际结果，动态调整下一步该做什么。

就像一个真正的开发者在工作一样：遇到问题就分析，分析完就动手，动手后验证，验证不通过就再来。

**一句话总结**

Agent Loop 就是一个 while 循环：不断调 LLM、执行工具、喂回结果，直到任务完成。

它是整个 Agent 的调度核心，把 LLM 的"思考能力"和工具的"执行能力"串成了一条完整的工作链。没有这个循环，LLM 就只是一个聊天机器人；有了它，LLM 才真正变成一个能干活的 Agent。

到这里，AI Agent 的三大核心概念：LLM、Tool Use、Agent Loop，我们就全部讲完了，你已经理解了一个 Agent 到底是怎么工作的。
