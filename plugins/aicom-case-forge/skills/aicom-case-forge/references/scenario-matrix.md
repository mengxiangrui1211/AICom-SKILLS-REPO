# 场景矩阵：指令形态 × 必备场景

为指令集生成用例时，逐条指令先判断形态，再按下表铺场景。**宁可少而准，不可多而假**——每个场景都必须有明确的验证目标。

## 形态识别

| 形态 | 语法特征 | 例 |
|---|---|---|
| 查询 | `AT+X?` 或无参数回读 | AT+CSQ / AT+CEREG? / AT+CGATT? |
| 测试 | `AT+X=?` | AT+CGDCONT=? / AT+CSCS=? |
| 设置 | `AT+X=<p>[,<p>...]` | AT+CEREG=1 / AT+CFUN=1 |
| 动作 | 执行一个过程（建立/发起/触发） | AT+QIOPEN=...（建立连接）/ ATD（拨号） |

## 各形态必备场景

### 查询类

| 场景变体 | 类型段 | 验证目标 |
|---|---|---|
| `QUERY_FORMAT` | RESP | 响应字节级格式：前缀、冒号空格、行帧、OK 收尾——`matches` 逐段写 |
| `QUERY_STATE` | RESP | 业务值合法域：extract + B 类断言（between / in / eq） |

查询不改设备状态，一般不需要状态快照；依赖前提（如 SIM 就绪）时 setup 宽松断言确认。

### 测试类（=?）

| 场景变体 | 类型段 | 验证目标 |
|---|---|---|
| `TEST_RANGE` | RESP | 返回参数支持范围/枚举与文档一致；格式完整含 OK |

### 设置类（=）

| 场景变体 | 类型段 | 验证目标 |
|---|---|---|
| `PARA-VALID` | PARA | 合法值写入 OK，**且回读查询复核值已生效**（只看 OK 不算验证） |
| `PARA-OVER` | PARA | 超上界 → ERROR/+CME ERROR，错误码与文档一致 |
| `PARA-UNDER` | PARA | 低于下界 → 同上 |
| `PARA-WRONG_FORMAT` | PARA | 类型错位（数字位给字符串等）→ ERROR |
| `PARA-ENUM` | PARA | 枚举参数：合法枚举逐一 VALID；非法枚举挑 1 个代表做 WRONG |

设置类**必配状态卫生**（见 `state-hygiene.md`）：setup 记初始值，teardown 恢复。

### 动作类

| 场景变体 | 类型段 | 验证目标 |
|---|---|---|
| `FUNC-NORMAL` | FUNC | 前提满足时执行成功（最终态查询复核 / 成功标志） |
| `FUNC-NOLINK` | FUNC | 前提不满足时给出明确错误（如未附着网络时发起连接 → 具体错误码） |

动作类注意事项：
- **异步动作**（受理 OK 后业务结果后到）：引擎的 wait_urc 为排期中字段不可用。拆步处理——
  步骤 1 发起（断言受理 OK），步骤 2 查询最终态（如连接状态查询指令）并在该步放宽 timeout。
  在用例 description 注明「异步结果以查询复核为准」。
- **破坏性动作**（关机/复位/擦除）：默认不生成；用户点名才做，且必须在汇报中明确
  「引擎无法自动恢复电源态，需人工恢复」。

## 优先级

- `p0`：核心功能 + 高频指令（查询格式 / 注册 / 附着 / 基础连接）
- `p1`：常规设置类与边界
- `p2`：错误码精确匹配、极端边界（生成量受限时先砍 p2）

## 生成自查清单

- [ ] 指令集中每条指令至少 1 个用例
- [ ] 查询类双场景（QUERY_FORMAT + QUERY_STATE）；测试类 TEST_RANGE
- [ ] 设置类四件套（VALID / OVER / UNDER / WRONG_FORMAT）或枚举版
- [ ] 动作类 FUNC-NORMAL + FUNC-NOLINK（前提可构造时）
- [ ] 改状态的指令有 setup 快照 + teardown 恢复
- [ ] 服务器/账号参数全部 `{{组.参数}}`，零硬编码
- [ ] 文件名分段合法、tags 与之一致、平台专属带平台首段
- [ ] 无排期中字段（data / when / retry / poll / wait_urc / expect / interval / parameters / loop / 套件）
- [ ] 全部 testcase_save 成功（保存即校验零报错）
