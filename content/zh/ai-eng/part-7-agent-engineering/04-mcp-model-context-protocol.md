---
title: "MCP：模型上下文协议"
description: "如何通过一个开放协议把工具、数据和提示词暴露给任意 AI 宿主；端到端搭出一个服务端和一个客户端。"
slug: 04-mcp-model-context-protocol
part: 7
chapter: 4
difficulty: intermediate
prerequisites:
  - 02-the-agent-loop
  - 03-tool-design
tags: [llm, agents]
date_created: 2026-06-12
date_updated: 2026-08-31
---

# MCP：模型上下文协议

你已经做出一个有用的工具：一个函数，搜索公司 wiki 并返回前三页。现在你的团队在用 Claude Code、Cursor、VS Code Copilot，还有一个自研的 Slackbot。这是四个宿主（host）。明天有人再加第五个。每个宿主都想用不一样的方式把这个搜索工具接进去：不同的配置格式、不同的 SDK、不同的参数声明方式和结果流式返回方式。

这就是 M × N 问题。M 个宿主、N 个工具、M × N 层胶水，而且宿主一改 API，对应的胶水层当场烂掉。十年前编辑器撞的是同一堵墙：每个 IDE 都给每种语言的分析器做一套定制集成。解法是 Language Server Protocol：一套线格式把编辑器与分析器解耦，于是一个 Python 的 LSP 服务端不用改一行代码就能在 VS Code、Neovim、Emacs 和 Zed 里跑。

模型上下文协议（Model Context Protocol，MCP）就是 AI 上下文领域的 LSP。[^1] 一套 JSON-RPC schema。说得上话的宿主都能连说得上话的服务端。Anthropic 于 2024 年 11 月把 MCP 开源，2025 年 12 月捐给 Linux 基金会；到 2026 年中，每一个主流 AI 编程界面都原生支持它，公共 registry 里挂着超过 9,400 个服务端。[^2]

## 三个角色，不是两个

多数工程师第一次接触 MCP，会在脑子里把它压成“客户端和服务端”，然后五分钟内就想错。这里有三个角色，中间那个最容易把人绊倒。

- **宿主（host）**：用户看到的那个 AI 应用：Claude Desktop、Cursor、你自己写的 agent。宿主同时拥有模型和用户的授权。
- **服务端（server）**：一个独立程序，对外提供能力（工具、文件、提示词）。服务端负责自己包住的后端系统的集成：数据库、SaaS API、本地文件系统。
- **客户端（client）**：宿主*内部*的一个对象，一个服务端配一个，专门负责接线。宿主连三个服务端，就实例化三个客户端。

客户端不是宿主。客户端也不是一个独立进程。它是宿主为了跟某一个具体服务端通话而创建的连接器对象，连接一断它就死掉。这种隔离是刻意的：一个恶意服务端能污染它自己的客户端，但没法跨过去读另一个服务端客户端里你发过去的内容。[^3]

规范版本迭代很快。本章对应 2025-11-25 修订版，也就是 2026 年中的最新稳定版。[^1]

![示意图：一个 AI 宿主内部有三个客户端对象，各自连向不同的服务端。两个本地服务端通过 stdio 子进程箭头连接，一个远程服务端通过 Streamable HTTP 连接，旁边带 OAuth 锁图标](../../../ai-eng/part-7-agent-engineering/images/04-host-client-server.png)
*一个宿主、三个客户端、三个服务端。每个客户端都是宿主内部一个独立的连接器；服务端可以作为本地子进程运行，也可以是远程 HTTP 服务。*

## 构建服务端

Python SDK 自带一个叫 `FastMCP` 的高层封装，它替你处理 JSON-RPC 的管道活儿，并从类型注解生成工具 schema。下面这个服务端暴露一个工具、一个资源、一个提示模板，够用上四个原语里的三个：

```python
# server.py (pip install mcp)
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("notes-server")
NOTES: dict[str, str] = {}

@mcp.tool()
def add_note(name: str, content: str) -> str:
    """Add or overwrite a note.

    Args:
        name: Unique note identifier (alphanumeric, no spaces).
        content: Text content of the note.
    """
    NOTES[name] = content
    return f"Stored note '{name}'"

@mcp.resource("notes://all")
def list_notes() -> str:
    """All stored notes as newline-separated key=value pairs."""
    return "\n".join(f"{k}={v}" for k, v in NOTES.items()) or "(no notes)"

@mcp.prompt()
def summarize_notes() -> str:
    """Return a prompt asking the LLM to summarize all stored notes."""
    body = "\n".join(f"- {k}: {v}" for k, v in NOTES.items()) or "(none)"
    return f"Summarize these notes concisely:\n{body}"

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

三个装饰器干掉了大部分活。`@mcp.tool()` 读函数签名和 docstring，生成模型看到的那份 JSON Schema；模型就靠这份 schema 挑工具、填参数。`@mcp.resource("notes://all")` 在一个固定 URI 上注册只读数据源。`@mcp.prompt()` 注册一个供用户选择的模板，在宿主 UI 里通常表现为斜杠命令。

> [!WARNING]
> **不要在 stdio 服务端里 `print()`。** stdio transport 把 stdout 完全留给按行分隔的 JSON-RPC。多打一句 `print("debug:", x)` 就把这条流写坏，客户端报解析错误，你能为此追一个小时。所有日志走 stderr：`print(..., file=sys.stderr)` 或者 `logging.basicConfig(stream=sys.stderr)`。这是官方 MCP 文档里最常见的入门错误。[^4]

## 构建客户端

客户端说的是同一个协议的另一端。真实场景里，客户端活在 Claude Desktop 或你的 agent 运行时内部；开发时你自己写一个，用来把服务端单独测一遍。

```python
# client.py (pip install mcp)
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def run():
    params = StdioServerParameters(command="python3", args=["server.py"])
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()                       # 1. 握手

            tools = await session.list_tools()               # 2. 发现
            print("Tools:", [t.name for t in tools.tools])

            result = await session.call_tool(                # 3. 调用
                "add_note", {"name": "hello", "content": "world"}
            )
            print("Tool result:", result.content[0].text)

            data = await session.read_resource("notes://all")  # 4. 读取
            print("Resource:", data.contents[0].text)

asyncio.run(run())
```

跑一下 `python3 client.py`，你会看到宿主把服务端作为子进程拉起来、完成协议握手、列出一个工具（`add_note`）、调用它，然后读 `notes://all` 资源并打印 `hello=world`。真实宿主每次连接跑的正是这套完整的“发现—使用”循环。

上面标了号的那四行就是你要掌握的全部词汇。`initialize()` 做版本与能力协商。`list_tools()`、`list_resources()`、`list_prompts()` 发现服务端手里有什么。`call_tool()` 和 `read_resource()` 去实际用它。分页、变更通知、进度上报，全都是 SDK 在同一套 JSON-RPC 原语上包好的封装。

## 四个原语

三个原语在服务端。有一个把信任方向反了过来，在客户端。它们不能互换，选错是 MCP 服务端设计里最常见的错误。

**工具（tools）** 是模型调用的函数。允许有副作用，参数必需，结果以文本、图像、音频或结构化 JSON 流回去。工具是*模型控制*的：由 LLM 根据任务决定什么时候调用。[^5] 一个操作只要真干了事（写一行数据、发一封邮件、带参数跑个查询），它就是工具。

**资源（resources）** 是按 URI 标识的只读数据源：`file:///etc/hosts`、`git://main/README.md`、`notes://all`。资源是*应用控制*的：挂哪些作为上下文，由宿主或用户决定，不是模型。[^6] 数据只要有稳定地址、没有参数，它就是资源。

**提示模板（prompts）** 是*用户*选择的模板，通常表现为斜杠命令。它们返回一串预格式化好的消息：一个 few-shot 范例、一段系统提示片段、一个多步流程的开场白。提示模板由用户控制。

界线很利：模型控制的是工具，应用控制的是资源，用户控制的是提示模板。拿不准时问一句：由谁决定 LLM 能不能看到这段内容。

**采样（sampling）** 是那个方向反过来的。服务端请求*客户端*替它跑一次 LLM 补全，走 `sampling/createMessage`。[^7] 这听着颠倒，直到你看见原因：一个需要在工具内部调用 LLM 的服务端作者有两条路，要么（a）把 Anthropic API key 硬编码进服务端、推理费用自己掏，要么（b）借宿主的模型、让用户付钱。采样是（b）。控制权仍在宿主手里：它挑真正用的模型，它把请求摆到用户面前等批准，它也可以拒绝。

规范在这一点上毫不含糊：“There SHOULD always be a human in the loop with the ability to deny sampling requests.”[^7] 把采样当成一种只授予受信服务端的特权，因为这个方向反转意味着恶意服务端可以拿你的钱跑任意 LLM 交互。

## 两种 transport

transport（传输方式）决定字节怎么在客户端与服务端之间走。数据层（JSON-RPC、原语、生命周期）两种方式完全一样；选哪种，取决于服务端住在哪里。

**stdio** 把服务端作为宿主的子进程拉起来。消息是按行分隔的 JSON，走 stdin 和 stdout。零网络配置，天然的进程沙箱，不需要认证，因为宿主本来就信任自己启动的任何东西。规范明确推荐它：“Clients SHOULD support stdio whenever possible.”[^8]

**Streamable HTTP** 把服务端作为独立服务跑。单一端点（例如 `https://api.example.com/mcp`）同时处理客户端发来的 POST，以及一个用于打开 Server-Sent Events 流、接收服务端到客户端消息的 GET。服务端在初始化时用 `Mcp-Session-Id` 头分配 session ID，客户端在之后每个请求里回带它。[^8] 这取代了 2024-11-05 版规范里用两个独立端点的 HTTP+SSE transport，新服务端只该上 Streamable HTTP。

默认用 stdio，只要东西是本地的：文件系统访问、git、shell 工具、跑在你笔记本上的数据库。等到服务端必须活得比客户端久（Sentry、GitHub 这类 SaaS 集成）、要同时服务多个客户端，或者跑在和宿主不同的机器上，才升级到 Streamable HTTP。中间没有过渡地带，这个选择是结构性的。

| 选哪种 transport | stdio | Streamable HTTP |
|---|---|---|
| 服务端的运行形态 | 宿主的子进程 | 独立的网络服务 |
| 认证 | 进程信任（协议层没有） | OAuth 2.1 |
| 多租户 | 一个进程一个客户端 | 一个服务端实例 N 个客户端 |
| 延迟下限 | 几毫秒（管道 IO） | 一次网络往返 |
| 适用场景 | 本地文件、git、shell、开发工具 | SaaS API、共享基础设施、托管服务端 |

## 认证：OAuth 2.1，以及它的锋利边缘

认证在 MCP 里是可选的，stdio 服务端应当直接跳过（改从环境变量读凭据）。[^9] 但你一旦上线 Streamable HTTP，规范就强制要求 OAuth 2.1，且每个客户端都必须用 PKCE。[^9] 两种流程覆盖真正要紧的场景：agent 代表用户行事（多数 SaaS 集成）用 Authorization Code；机器对机器用 Client Credentials。

MCP 特有的麻烦在发现（discovery）这一步。MCP 服务端是一个 OAuth *资源服务器*，它通过 OAuth 2.0 Protected Resource Metadata（RFC 9728）公布自己的授权服务在哪里。未认证的客户端发来请求时，401 响应里带一个 `WWW-Authenticate` 头，指向服务端的资源元数据文档；从 2025-11-25 修订版起，服务端也可以直接把这份文档发布在 `/.well-known/oauth-protected-resource`，客户端不必先吃一个 401 就能发现。文档里的 `authorization_servers` 字段给出授权服务器，授权服务器自己的元数据（RFC 8414）再提供真正的端点。[^9] 老客户端仍然硬编码 2025 年 6 月之前的行为：把 MCP 服务端 URL 的路径剥掉当 auth base URL，再去上面探 `/.well-known/oauth-authorization-server`；对着一个跟规范走的服务端，这就是那种没头没尾的“no auth server found”报错的经典成因。

还有两条规则值得背下来：token 放在 `Authorization: ***` 头里，绝不进 query string；服务端 SHOULD 支持 Dynamic Client Registration（RFC 7591），让客户端自己注册，不用人工协调。跳过 DCR，你就只能靠发邮件发 client ID 了。

放到架构尺度上看，多服务系统里的认证在 HLD 手册的[认证 vs 授权一章](../../hld/part-7-security-at-scale/00-authn-authz.md)里有专门的深入讨论；MCP 的 OAuth 2.1 流程只是那片更大图景里切得比较清楚的一块。

## 会卡死的握手

每次会话都由三条消息开头。弄错一条，连接就静悄悄地锁死。

1. 客户端发 `initialize`，带上自己支持的 `protocolVersion`、声明的 `capabilities` 和 `clientInfo`。
2. 服务端回它实际会用的 `protocolVersion`、自己的 `capabilities` 和 `serverInfo`。
3. 客户端发 `notifications/initialized`（单向通知，没有 `id` 字段，不期待回应）。

第三步是最常被跳过的。少了它，守规矩的服务端会拒绝处理任何其它请求，因为规范写着：“the server SHOULD NOT send requests other than pings and logging before receiving the initialized notification.”[^10] 如果你的 `tools/list` 调用一直挂着、连个错误都不报，第一件要查的就是这条漏发的 initialized 通知。Python SDK 的 `session.initialize()` 帮你把两条消息都发了，这也是该用 SDK 而不是裸写 JSON-RPC 的原因之一。

能力协商是另一份你不能违反的契约。没声明 `sampling: {}` 的客户端收不到 `sampling/createMessage` 请求，服务端想发也发不过来；规范说未声明的能力 MUST NOT 被使用。声明了自己其实没实现的能力，坏得一样彻底。声明你代码能做到的，别声明路线图上承诺的。

## 不顺利的那条路

协议设计里有一处尖角，被正常路径（happy path）遮住了。服务端可以把提示注入的指令埋进某个工具的 `description` 字段，而客户端一调 `tools/list`，这些指令就进了模型上下文，在任何工具被真正调用之前就已经执行完。Trail of Bits 在 2025 年 4 月拿 Claude Desktop、Cursor、Cline 和 Windsurf 演示了这一点，把这类攻击命名为“line jumping”。[^11] 一项覆盖五种 MCP 实现、847 个攻击场景的学术研究测出：针对 MCP 集成的攻击成功率，比等价的直接集成高出 23% 到 41%。[^12]

这一类攻击，以及真正管用的缓解手段，单独留给了下一章：[MCP 安全](./05-mcp-security.md)讲工具投毒、rug pull、跨服务端的工具影子，以及生产部署用来把它们圈住的网关模式。

## 参考文献

[^1]: Model Context Protocol, "Specification revision 2025-11-25," https://modelcontextprotocol.io/specification/2025-11-25 (latest stable as of June 2026).
[^2]: digitalapplied.com, "MCP Ecosystem H1 2026 Retrospective," https://www.digitalapplied.com/blog/mcp-ecosystem-h1-2026-retrospective-adoption-data-points
[^3]: Model Context Protocol, "Architecture Overview," https://modelcontextprotocol.io/docs/learn/architecture
[^4]: Model Context Protocol, "Build an MCP server," https://modelcontextprotocol.io/docs/develop/build-server
[^5]: Model Context Protocol, "Tools, spec 2025-11-25," https://modelcontextprotocol.io/specification/2025-11-25/server/tools
[^6]: Model Context Protocol, "Resources, spec 2025-11-25," https://modelcontextprotocol.io/specification/2025-11-25/server/resources
[^7]: Model Context Protocol, "Sampling, spec 2025-11-25," https://modelcontextprotocol.io/specification/2025-11-25/client/sampling
[^8]: Model Context Protocol, "Transports, spec 2025-11-25," https://modelcontextprotocol.io/specification/2025-11-25/basic/transports
[^9]: Model Context Protocol, "Authorization, spec 2025-11-25," https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization
[^10]: Model Context Protocol, "Lifecycle, spec 2025-11-25," https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle
[^11]: Trail of Bits, "Jumping the line: How MCP servers can attack you before you ever use them," April 21 2025, https://blog.trailofbits.com/2025/04/21/jumping-the-line-how-mcp-servers-can-attack-you-before-you-ever-use-them/
[^12]: arXiv, "Security Analysis of the Model Context Protocol Specification and Prompt Injection Vulnerabilities in Tool-Integrated LLM Agents," 2601.17549, https://arxiv.org/abs/2601.17549v1
