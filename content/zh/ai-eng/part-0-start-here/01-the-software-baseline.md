---
title: "软件基线"
description: "本书默认你已经具备的七项工程技能（Python、HTTP、JSON、异步、Git、Docker、Postgres），它们对 AI 工作各自为何重要，以及该去哪儿补。"
slug: 01-the-software-baseline
part: 0
chapter: 1
difficulty: beginner
prerequisites:
  - 00-what-an-ai-engineer-is
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 软件基线

第 0 部分结束时，你会写出一个小脚本：调用模型 API，失败时重试，记下每次调用花了多少钱。听起来微不足道。

这个脚本同时用上七项技能：Python 写主体，HTTP 发请求，JSON 解析回复，批量调用要靠 asyncio，提交靠 Git，交付靠 Docker，记录成本靠 Postgres。本书假定这七项你都已经会了。这一章划的是及格线，不是课程。

上一章[AI 工程师是什么（以及不是什么）](./00-what-an-ai-engineer-is.md)划出了角色边界：你在模型 API 这条线之上工作。所以基线就是一个能干活的软件工程师的工具箱，仅此而已。不要机器学习数学，不要分布式系统理论，不要算法深度。这些都不是本书该有的前置知识；真到需要的时候，隔壁那几本手册负责讲。

![一张居中的脚本卡片，连着七个填色的技能圆圈，分别标注 Python、HTTP、JSON、异步、Git、Docker、Postgres](../../../ai-eng/part-0-start-here/images/01-seven-skills.png)
*一个普通脚本，七项普通技能；哪一根辐条缺了，脚本就发不出去。*

把这七项当成一次体检，而不是一份课程表。下面每一行只问一个问题：现在、今天，不查资料你能做得到吗？能，就往下走。哪一行让你犹豫，就打开它的链接，花上几个小时，再回来。别想着一次把七项全学了。

## 七项技能

| 技能 | 在这里为什么重要 | 你应该能做到 | 补课 |
|---|---|---|---|
| **Python** | 本书里每个 SDK（openai、anthropic、httpx、pydantic）都是 Python 写的。没有它，一段示例都跑不起来。 | 写一个函数：遍历列表，对每项调点东西，捕获异常，返回 dict。会 `pip install` 并导入包。 | [Python 教程](https://docs.python.org/3/tutorial/)[^1] |
| **HTTP / REST** | 每家服务商都在 HTTPS 上暴露 REST API，SDK 只是一层薄封装。重试、降级、流式输出全靠它。 | 说清一个 POST 的构成（method、URL、headers、body），知道 200/400/429/500 各是什么意思，会用 Bearer token。 | [HTTP 概览](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview)[^2] |
| **JSON** | 它是每个请求和响应的传输格式，也是每个 tool call 的 schema、每条 eval 日志和 trace 的格式。 | 能读嵌套键，用 `json.loads()` 解析，用 `json.dumps()` 序列化 dict。 | [JSON 入门](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/Scripting/JSON)[^3] |
| **异步（asyncio）** | 模型调用是慢速网络 I/O。agent 会扇出 tool call，eval 会扇出 prompt；你要让它们一起等，而不是一个一个等。 | 定义 `async def`，`await` 它，用 `asyncio.gather()` 跑起一堆协程。 | [asyncio 文档](https://docs.python.org/3/library/asyncio.html)[^4] |
| **Git** | prompt 的改动就是代码的改动。第 3 部分把 prompt 模板按 commit 做版本管理，在 pull request 里评审。 | clone、开分支、用像样的提交信息 commit、push、开 PR、解决一个简单的合并冲突。 | [Pro Git](https://git-scm.com/book/en/v2)[^5] |
| **Docker** | 生产的 AI 是以容器化服务的形式上线、带真实 URL 的，不是 notebook。推理服务、eval runner、agent 后端全都打包成容器发出去。 | 给一个 Python 服务写 Dockerfile，`docker run` 跑起来，用 Docker Compose 拉起一个服务加 Postgres。 | [Docker 入门](https://docs.docker.com/get-started/)[^6] |
| **SQL / Postgres** | Postgres 是三件事的默认存储：pgvector 做检索、eval 日志、聊天历史。 | 会写 SELECT、INSERT、UPDATE、DELETE，知道主键和索引是什么，能连上本地 Postgres。 | [PostgreSQL 教程](https://www.postgresql.org/docs/current/tutorial.html)[^7] |

检查环境时顺带说一下版本。本书用的是 Python 3.10+ 语法，所以动手前先确认 `python --version` 显示 3.12 或更新；3.8 会直接报语法错误。上面链接的 Python 与 asyncio 文档截至 2026 年 6 月跟到 3.14.6，Postgres 教程跟到 18.4。

> [!WARNING]
> **JSON 和 JSON Schema 不是一回事，而第 2 部分两个都用。** JSON 是上面那一行说的数据格式。JSON Schema 是另一份规范，描述一个合法 JSON 对象必须具备的*形状*，tool call 和结构化输出就是这样定义的。如果“给你的工具写一个 JSON Schema”听起来莫名其妙，缺口就在这里。格式就是上面链接的那篇一小时读物；schema 规范会在用到它的上下文里再介绍。

## 现在就该认出来的一个惯用法

异步那项的自检是“你能 gather 协程”。它具体长这样——第 2 部分并行调模型、第 7 部分并行调工具时，你都会碰到它。

```python
# 示意用：这是扇出（fan-out）惯用法，不是可运行的 demo
import asyncio

async def call_model(prompt: str) -> str:
    await asyncio.sleep(0)            # 代替真实的异步 SDK 调用
    return f"response to: {prompt}"

async def main() -> None:
    prompts = ["summarize A", "summarize B", "summarize C"]
    results = await asyncio.gather(*[call_model(p) for p in prompts])
    print(results)

asyncio.run(main())
```

把 `gather(*[f(item) for item in items])` 读成一个动作：一起发出去，等全部回来，按顺序收结果。这一行看得顺，你就准备好了。看着别扭，就在第 2 部分之前花一个小时看那篇 asyncio 链接。

收益不是装点门面。50 条 prompt 用 `for` 循环串行跑，每条 3 秒，你要花 150 秒。同样 50 次调用走 `gather`，总耗时大致等于最慢的那一次，常常不到 10 秒。你写第一个 eval 脚本就能差出 15 倍——这就是异步在这份清单上、而不在附录里的原因。

## 不在清单上的东西

这是地板，不是天花板。有三个话题后面会大规模出现，各自归隔壁手册：

- **大规模 HTTP**（负载均衡、连接池、HTTP/2 多路复用）在 HLD 手册里讲。你需要的是请求/响应模型，暂时不需要扩展理论。
- **大规模数据库**（复制、分区、分片）同样是 HLD 的地盘。这里的及格线只有一条：“你到底会不会用 Postgres。”
- **算法与数据结构**完全不在这份清单上。想要那个深度，去 DSA 手册。AI 工程不以它为门槛。

容器编排也是同一档：本地 Docker 和 Docker Compose 就是及格线，Kubernetes 或 ECS 只在讲到部署时给一句话和一个交叉链接。

这一章之后的所有内容，都假定这七项已经到位，不再解释。

## 参考文献

[^1]: Python Software Foundation, "The Python Tutorial," Python 3.14.6 documentation, https://docs.python.org/3/tutorial/
[^2]: MDN Web Docs contributors, "Overview of HTTP," Mozilla Developer Network, https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview
[^3]: MDN Web Docs contributors, "Working with JSON," Mozilla Developer Network, https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/Scripting/JSON
[^4]: Python Software Foundation, "asyncio - Asynchronous I/O," Python 3.14.6 standard library docs, https://docs.python.org/3/library/asyncio.html
[^5]: Scott Chacon and Ben Straub, "Pro Git," 2nd edition, Apress, licensed CC BY-NC-SA 3.0, https://git-scm.com/book/en/v2
[^6]: Docker Inc., "Get started," Docker official documentation, https://docs.docker.com/get-started/
[^7]: PostgreSQL Global Development Group, "Part I. Tutorial," PostgreSQL 18.4 documentation, https://www.postgresql.org/docs/current/tutorial.html
