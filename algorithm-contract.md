# 算法进程契约

算法与系统其余部分的**唯一**联系方式：stdin 进、stdout 出。本文是该契约的完整定义与新算法接入指南。

## 1. 契约定义

```
输入  stdin   一个 JSON 对象（结构由算法定义，如 {"values":[5,3,8]}）
输出  stdout  NDJSON 事件流，符合 protocol/spec/v1.md（每行一个事件，UTF-8，无 BOM）
调试  stderr  任意内容（ALGOLAB_DEBUG=1 可选开启），不得影响协议解析
退出  exit 0 成功（尾消息必须是 complete）
      exit 1 失败（流内应有 error 事件；invalid_input 表示输入不合法）
```

### 1.1 事件流形态

```
{... seq:0, kind:"init",     ...}   # 声明初始模型（领域 payload 结构见领域子规范）
{... seq:1..n, kind:"event", ...}   # 过程事件（compare/swap/visit/pointer_update/...）
{... seq:n+1, kind:"complete", ...} # 终态（payload 携带 result）
```

- `seq` 从 0 连续递增；`run_id` 本次运行内一致
- 无 init 的 error 流也合法（如 `invalid_input`：首行即 `error` 事件）
- 协议/领域校验规则以 [protocol/spec/v1.md](../protocol/spec/v1.md) 与 [domains/](../protocol/spec/domains) 为准，本文不重复

### 1.2 错误语义（invalid_input 示例）

```json
{"protocol_version":"1.0","run_id":"run-1","seq":0,"kind":"error","domain":"array",
 "type":"invalid_input","payload":{"message":"input.values must be an array of numbers"}}
```

输出后进程以退出码 1 结束。网关会把 run 标记为 `failed` 并将 `payload.message` 记入 `RunRecord.error`。

## 2. 硬性约束

| 约束 | 值 | 说明 |
|---|---|---|
| stdout 纯净性 | **只允许协议事件** | 日志/调试一律走 stderr，否则 parseTrace 拒绝 |
| 输入体积 | ≤ 64 KiB | 网关侧限制（HTTP 413） |
| 执行时长 | ≤ 10 s | 网关超时 SIGKILL，run 标记 failed |
| stdout 体积 | ≤ 8 MiB | 网关侧收集上限 |
| 输入校验 | **算法自己做** | 网关零校验（唯一校验点原则，见 architecture.md §3.2） |

## 3. 运行时接入指南

网关按 registry 的 `runtime` 字段选择解释器（`apps/gateway/src/runner.ts` 的 `commandFor()`）。

### 3.1 Node（`runtime: 'node'`）

- 单文件零依赖 `.mjs`，放 `algorithms/javascript/<域>/`
- 命令：`process.execPath <script>`（复用网关自身 Node，不依赖全局安装）
- 蓝本：[insertion-sort.mjs](../algorithms/javascript/sorting/insertion-sort.mjs)

### 3.2 Python（`runtime: 'python'`）

- 单文件零依赖 `.py`，放 `algorithms/python/<域>/`
- 命令：`python <script>`（要求 PATH 上有 `python`；仓库基线 3.14）
- **三个必处理的坑**（蓝本已示范）：

```python
sys.stdin.reconfigure(encoding="utf-8")    # ① Windows 默认 GBK，协议要求 UTF-8
sys.stdout.reconfigure(encoding="utf-8")
print(json.dumps(event, separators=(",", ":")), flush=True)  # ② 管道默认块缓冲：
#    不逐行 flush 则超时/崩溃时 trace 丢失，且网关收不全
json.dumps(..., ensure_ascii=True)         # ③ 默认值即 ASCII 转义，保持不改动最安全
```

- 校验输入时注意 `isinstance(True, int)` 为真——bool 需显式排除
- 蓝本：[insertion-sort.py](../algorithms/python/sorting/insertion-sort.py)、[linked-list/reverse.py](../algorithms/python/linked-list/reverse.py)

### 3.3 Java（未实现，留位）

- 可用 JEP 330 单文件源码启动：`java Xxx.java`（Java 11+，免编译；类名须匹配文件名、不写 package）
- 代价：JVM 启动约 0.5s（仍在 10s 超时内）
- 接入步骤同 §4，`commandFor()` 加 `'java'` 分支即可

## 4. 新算法接入 Checklist

1. **写脚本**：按 §3 蓝本实现，注意 stdout 纯净性与 invalid_input 契约
2. **注册**：`apps/gateway/src/registry.ts` 加一条 `entry(...)`（algorithmType 全局唯一；domain 必须与输出的领域一致；serviceId 按 runtime 自动归属）
3. **子进程测试**：`tests/algorithms.test.ts` 加用例——trace 合法性、结果正确性（对照参考实现）、invalid_input 拒绝；跨语言移植必须加 **parity 用例**（同输入下与原版事件流逐行一致，忽略 run_id）
4. **网关测试**：`apps/gateway/tests/gateway.test.ts` 视需要更新目录断言（条目数）与闭环用例
5. **完成**：前端下拉自动出现新条目（目录来自 `GET /api/algorithms`），无需改前端；预置 trace 可用 `npm run traces` 重新生成

## 5. 事件 → 渲染的边界

算法输出什么事件、payload 里放什么 index/id，决定了前端高亮什么。呈现映射（哪个事件染什么色）属于渲染器职责（`apps/web/src/App.svelte` 的 `*HighlightFor` 函数），算法只保证事件语义符合领域子规范——两者通过领域子规范对齐，不通过代码耦合。
