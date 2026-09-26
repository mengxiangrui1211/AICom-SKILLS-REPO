---
name: aicom-case-forge
description: AICom 测试用例生成技能。触发场景：用户要求「按 AT 指令集写测试用例」「把指令集文档转成测试用例」「为某模组/平台批量生成测试用例」「补充某指令的测试覆盖」。产出符合 AICom 引擎规范的 YAML 用例，经 testcase_save 入库、testcase_run 执行。
---

# AICom 测试用例生成规范总纲

你的任务：把一份 AT 指令集（文档/手册/指令清单）转换为一组可直接执行的 AICom 测试用例（YAML，一文件一用例），逐个经 `testcase_save` 写入用例库。

## 四个机制（本规范的运作方式）

1. **规范经 MCP 分发**：本技能全部文件由 AICom 网关在线分发——**用 `skill_read` 工具读取**（所有支持 MCP 的客户端可用；不带 path 调用返回 SKILL.md 全文与文件清单）。支持 MCP resources 的客户端也可 `resources/read`，URI 形如 `aicom://skills/aicom-case-forge/SKILL.md`。零本地文件操作，规范永远与 AICom 版本一致。
2. **保存即校验**：`testcase_save` 在保存时做解析期校验——未知键直接报错、断言写法/正则空串/互斥字段当场拦截。
   你不需要任何校验脚本，保存报错按提示（带位置前缀）修正后重存即可。
3. **活知识库**：`references/response-patterns.md` 沉淀平台响应规律，写断言前必读；由维护者持续更新
   （运行报告自动沉淀机制规划中）。
4. **平台归档**：用例目录保持平铺，平台归属用文件名与 `tags` 首段平台名表达（如
   `EG915-TCP-QIOPEN-FUNC-NORMAL.yaml`）；跨平台通用的标准指令用例不加平台段。`testcase_list` 可按 tags 归组。

## 工作流（按序执行）

**第 0 步 · 判断平台状态**
先 `testcase_list` 看用例库现状：
- 目标平台已有用例 → 读 `references/response-patterns.md` 中对应平台规律，进入第 1 步
- 全新平台 → **必读** `references/platform-onboarding.md`，先完成 5 步勘测（文档来源 → 响应勘测 →
  环境参数表 → 归档约定 → 校准生成）再动手

**第 1 步 · 场景规划**
读 `references/scenario-matrix.md`，按每条指令的形态（查询 `?` / 测试 `=?` / 设置 `=` / 动作）列出场景清单。
指令较多时先输出「指令 × 场景」规划表给用户确认。

**第 2 步 · 写用例**
读 `references/yaml-schema.md`（语法唯一权威），同时遵守：
- 单一职责：一个文件只测一个指令的一个维度
- 命名分段规范（见 yaml-schema.md §命名与 tags）
- 指令会改设备状态 → 按 `references/state-hygiene.md` 配 setup 快照 / teardown 恢复
- 涉及服务器/端口/账号 → 按 `references/env-binding.md` 用 `{{组.参数}}`，禁止硬编码

**第 3 步 · 入库校准**
逐个 `testcase_save`。校验报错带位置前缀（如 `[steps[1].assert[0]]`），照改重存。全部通过后 `testcase_list` 核对。

**第 4 步 · 汇报**
- 已生成清单：文件名 → 覆盖指令 → 场景类型
- 引用到的测试环境分组（提醒用户在 AICom「测试环境」页填真实值，或经同意后你用 `testenv_set` 写入）
- 未覆盖的指令与原因（文档缺失 / 引擎排期中 / 需设备实测校准）

## 硬规则（违反任何一条 = 返工）

1. **不要使用引擎排期中的字段**：`data` / `when` / `retry` / `poll` / `wait_urc` / `expect` / `interval` /
   `parameters` / `loop` / 套件。保存能通过，执行不支持。用户明确要求预留时才写，并在汇报中注明。
2. **禁止硬编码环境参数**：IP、域名、端口、APN、用户名、密码、URL 一律 `{{组.参数}}`。
3. **正则一律单引号字符串**，`\r\n` 显式书写；业务码捕获用排除字符类如 `([^\r\n,]+)`，不用贪婪 `.+`
   （响应保留 `\r`，贪婪会吃进不可见回车）。
4. **不臆测响应格式**：先查 `references/response-patterns.md`；没有记载的规律，向用户申请用
   `serial_write` 实测一条再定断言。
5. **同步指令永远不加 wait_urc 类等待**：同步指令成功直接回 OK，等 URC 会等满超时必失败。
6. **键名严格按 yaml-schema.md 拼写**：未知键被拒收是校验器特性，不是缺陷。

## 渐进披露（按需读取，勿一次全读）

| 文件 | 何时读 |
|---|---|
| references/yaml-schema.md | 每次写用例前；语法唯一权威 |
| references/scenario-matrix.md | 第 1 步场景规划 |
| references/env-binding.md | 用例涉及服务器/账号/外部参数 |
| references/state-hygiene.md | 指令会改设备状态、开连接、开实例 |
| references/platform-onboarding.md | 首次为某平台生成用例 |
| references/response-patterns.md | 写断言前；响应规律知识库 |
