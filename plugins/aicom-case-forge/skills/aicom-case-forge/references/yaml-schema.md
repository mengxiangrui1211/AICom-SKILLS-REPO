# YAML 语法全表（引擎对齐版）

一个 YAML 文件 = 一个用例，UTF-8，`.yaml` 后缀。保存经 `testcase_save`，校验器拒绝任何未知键（拼写错误当场拦截）。

> **引擎支持范围**
> 已支持：setup/steps/teardown 三段式、command、timeout、extract、A/B 类断言、on_failure、模板渲染。
> 排期中（⛔）：下表标注的字段——保存可通过、执行不支持，生成用例时**禁用**。

## 顶层字段

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `name` | str | 是 | 非空白；报告以「文件名 + name」双标识展示 |
| `description` | str | 建议 | 三段式：场景前提 / 验证目标 / 文档依据 |
| `tags` | list[str] | 建议 | 归组过滤用，见「命名与 tags」 |
| `setup` | list[Step] | 否 | 前置；任一步失败 → 整例 SKIPPED（前提不满足 ≠ 用例失败） |
| `steps` | list[Step] | **是**(≥1) | 主步骤序列 |
| `teardown` | list[Step] | 否 | 后置，无条件执行；失败仅记录不影响用例结果 |
| `on_failure` | str | 否 | 用例级失败策略 `abort`/`skip`/`continue`（步骤未配时兜底） |
| `port` | str | 否 | 元数据标注，不影响实际执行端口 |
| `parameters` | list[dict] | ⛔排期中 | 参数化矩阵 |
| `loop` | dict | ⛔排期中 | 压测 `{count, interval, warmup, abort_on_failure}` |

## 步骤字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `command` | str | AT 指令；发送前渲染 `{{var}}`/`{{组.参数}}`；与 `data` 二选一 |
| `timeout` | float | 单步超时秒数（>0），默认 5s；连接/协商类指令显式放宽 |
| `extract` | dict | `{变量名: 正则}`；捕获组 1 → 变量；无匹配不入池 |
| `assert` | 元素或列表 | 见下节；多元素 AND，全过步骤才通过 |
| `on_failure` | str | `abort`（默认，中止跳 teardown）/ `skip`（记 SKIPPED 继续）/ `continue`（记 FAIL 继续） |
| `data` | dict | ⛔排期中 数据流输入 |
| `when` | str | ⛔排期中 条件跳过 |
| `retry` | dict | ⛔排期中 `{count, interval}`；与 poll/wait_urc 互斥 |
| `poll` | dict | ⛔排期中 `{until, timeout, interval}` 轮询 |
| `wait_urc` | str | ⛔排期中 异步 URC 终结 |
| `expect` | str | ⛔排期中 附加完成正则（如 `\r\n>` 提示符） |
| `interval` | int | ⛔排期中 发送前延迟毫秒 |

## 断言（两类，同一元素不可混用）

元素可加 `name` 字段便于报告定位（A/B 类通用）。

**A 类·响应原文断言**（对完整响应文本求值，四选一）：

| 键 | 语义 | 备注 |
|---|---|---|
| `contains` | 子串包含 | 不可空串 |
| `not_contains` | 不含子串 | 不可空串 |
| `matches` | 正则匹配 | 字节级断言首选 |
| `equals` | 与完整响应全等 | 断言空响应用 `equals: ''` |

**B 类·变量断言**（对 extract 提取的变量求值，`var` + `op` 必须同时提供）：

| op | 参数 | 语义 |
|---|---|---|
| `eq` / `ne` | value | 字符串相等/不等（extract 提取的是**字符串**：`"1"` 不是 `1`） |
| `gt` / `lt` / `ge` / `le` | value | 数值比较（转数值失败判失败） |
| `between` | min + max | 闭区间，校验 min ≤ max |
| `in` | values 列表 | 成员判断，如 `values: ["1", "5"]` |
| `contains` / `matches` | value | 变量值包含/匹配 |

变量未定义（extract 没匹配上）→ 该断言判失败，原因为「变量 X 未定义」。

## extract 与模板

```yaml
- command: AT+CEREG?
  extract:
    cereg_stat: '\+CEREG:\s*\d+,(\d)'
  assert:
    - { name: 已注册或漫游, var: cereg_stat, op: in, values: ["1", "5"] }
```

- 捕获组 1 优先，无分组取整体匹配；无匹配 → 变量不入池
- 变量池跨 setup → steps → teardown 顺序流动，先提取后引用

模板占位符（只做字符串替换，无表达式）：

| 写法 | 查找顺序 |
|---|---|
| `{{var}}`（无点号） | ① 用例变量池 → ② 测试环境 default 组 → 无则本步渲染失败 |
| `{{组.参数}}`（一级点号） | 仅查测试环境配置（如 `{{tcp.host}}`） |
| `{{a.b.c}}`（两级以上点号） | 解析期直接报错 |

内置变量：`timestamp`（每步刷新）、`port`（实际执行端口）；二者是保留字，extract 不得占用。
详细引用规则见 `env-binding.md`。

## 命名与 tags

文件名分段，功能块-指令-类型-变体：

| 段 | 规则 | 示例 |
|---|---|---|
| [平台] | 平台专属用例才加首段（厂商扩展指令/依赖平台行为） | `EG915` |
| 功能块 | 大写功能域 | `NETWORK` / `TCP` / `HTTP` |
| 指令 | 被测指令去 `AT+` 裸名 | `CEREG` / `CSQ` |
| 类型 | `RESP` 响应格式 / `PARA` 参数边界 / `FUNC` 功能验证 / `REGRESS` 回归 | `RESP` |
| 变体 | 具体测试点；回归用例放 BUGID | `QUERY_FORMAT` |

- 跨平台通用（纯标准指令）：`NETWORK-CSQ-RESP-QUERY_FORMAT.yaml`，tags `[NETWORK, CSQ, RESP, p0]`
- 平台专属：`EG915-TCP-QIOPEN-FUNC-NORMAL.yaml`，tags `[EG915, TCP, QIOPEN, FUNC, p1]`（**tags 首段平台名**）
- 单一职责第一原则：一个文件只测一个指令的一个维度；前置依赖只进 setup（宽松断言），清理只进 teardown
- `suite-` 前缀保留给套件索引文件，普通用例禁用

## 常见错误速查

| 症状 | 原因与对策 |
|---|---|
| 保存报「未知键」 | 键拼写错误（`notcontains` ≠ `not_contains`），对照上表 |
| 「变量 X 未定义」 | extract 没匹配上不写入池；检查正则（`\s*`、排除字符类） |
| `contains: ""` 被拒 | 空串行为反直觉，解析期报错；空响应断言用 `equals: ''` |
| retry 与 poll 同时写被拒 | 二者互斥（wait_urc 与 poll 也互斥）——反正都是排期中字段，直接别用 |
| 业务码响应等待超时 | 引擎终结只认 OK/ERROR/+CME/+CMS；纯业务码结尾的步骤加小超时（如 `timeout: 1.2`）兜底 |
| 冒号后空格数对不上 | 严格断言按指令文档响应格式逐字节写，不臆测（查 response-patterns.md） |

## 最小骨架（复制起点）

```yaml
name: <指令>-<测试点>
description: |
  场景前提：……
  验证目标：……
  文档依据：……
tags: [<功能块>, <指令>, <类型>, p0]

setup:
  - command: ATE0                      # 关回显，保证字节级断言稳定
    assert: { contains: "OK" }         # 宽松断言：只确认前提

steps:
  - command: AT+XXX?
    extract:
      v: '\+XXX:\s*([^\r\n,]+)'
    assert:
      - { name: 格式, matches: '^\r\n\+XXX: [^\r\n]+\r\nOK\r\n$' }
      - { name: 值域, var: v, op: in, values: ["1", "5"] }

teardown:
  - command: ATE0
```
