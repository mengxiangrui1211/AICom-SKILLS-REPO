# 状态卫生：setup / teardown 配对纪律

用例执行是真实操作设备。**谁改状态谁恢复**是硬纪律——一条用例留下脏状态，污染的是后续所有用例。

## 三段式语义（引擎实际行为）

| 段 | 失败语义 | 用途 |
|---|---|---|
| setup | 任一步失败 → 整例 SKIPPED（前提不满足 ≠ 用例失败） | 确认前提 + 记录初始状态 |
| steps | 按失败策略 abort / skip / continue | 被测行为 |
| teardown | **无条件执行**（含 setup 失败、中途 abort 后），失败仅记录 | 恢复设备状态 |

## 配对模式

### 模式一：改设置 → 恢复原值

```yaml
setup:
  - command: AT+CREG?
    extract:
      orig_mode: '\+CREG:\s*(\d+),'      # 记下当前上报模式（逗号前的 n）

steps:
  - command: AT+CREG=1                    # 改
    assert: { contains: "OK" }
  # ...被测行为与回读复核

teardown:
  - command: 'AT+CREG={{orig_mode}}'      # 恢复（变量池跨段流动）
    on_failure: continue
```

初始值提取失败时 `{{orig_mode}}` 渲染失败 → teardown 该步失败但仅记录——
这正是清理步骤配 `on_failure: continue` 的意义：清理失败不该炸掉整个收尾。

### 模式二：开连接/实例 → 先关后毁（双层清理）

```yaml
teardown:
  - command: AT+QICLOSE=0                 # 第一层：关连接（可能本就未建立，失败属正常）
    on_failure: continue
  - command: AT+QIDEACT=1                 # 第二层：去激活 PDP
    on_failure: continue
```

协议类（HTTP/MQTT/FTP）逐层齐备：关会话 → 断连接 → 销毁实例
（如 HTTPDISCONNECT → HTTPDESTROY），每层 `on_failure: continue`。
**漏销毁实例会连锁污染后续用例**——实例句柄占满后新建全部失败。

### 模式三：前提确认（宽松断言）

setup 的断言目的是「确认前提」，宽松即可：

```yaml
setup:
  - command: AT+CPIN?
    assert: { contains: "READY" }         # 无卡设备在此判定前提不满足 → 整例 SKIPPED
```

严格格式断言只留给 steps 里被测指令本身。

## 禁止事项

- setup 里做被测行为本身（那是 steps 的事）
- teardown 里写严格断言（清理失败很常见——资源本就没建起来）
- 用例结束留下：未关连接、未销毁实例、未恢复的设置、未关的回显
  （字节级断言用例的标配是 setup `ATE0` + teardown `ATE0`）
