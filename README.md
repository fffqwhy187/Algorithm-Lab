# Algorithm Lab — 算法可视化实验室

用**统一事件协议**驱动算法可视化：算法只负责输出协议事件流，播放系统只负责按时间轴回放事件流。两者通过协议解耦，因此可以独立演进——新增数据结构、新增算法、甚至新增实现语言，都不需要触碰播放系统。

## 核心理念

1. **协议先行**：[执行事件协议 v1.0](protocol/spec/v1.md) 是唯一契约。算法（任何语言）stdin 吃一个 JSON 输入，stdout 吐 NDJSON 事件流；协议层校验后交给领域适配器，算法永不依赖 UI。
2. **双时间轴**：算法在后端毫秒级全速执行完毕，网关缓存完整事件流；前端按自己的速度播放。执行速度与播放速度彻底解耦。
3. **适配器缝**：每个数据结构对应一个 `DomainAdapter`，注册进 `DomainRegistry` 后播放器自动可用。第三~五阶段新增链表/树/图时，PlaybackController / Timeline / CheckpointManager 核心逻辑**零改动**（证据见 [架构文档](docs/architecture.md)）。

## 快速开始

```bash
npm install          # 安装全部 workspace 依赖
npm run dev          # 并行启动：算法网关(8787) + Web 前端(5173)
```

打开 <http://localhost:5173>：

- **预置用例**：下拉选择预置 trace 离线回放（无需网关）
- **实时执行**：在「实时执行（算法网关）」面板选择算法（含 Python 运行时）、编辑 JSON 输入、执行并回放，历史记录可点击重放

## 常用脚本

| 命令 | 作用 |
|---|---|
| `npm run dev` | 并行启动网关（8787）+ Web（5173），Vite 代理 `/api` |
| `npm run web` / `npm run gateway` | 单独启动前端 / 网关 |
| `npm test` | 全量测试（协议、引擎、时间轴、网关集成、算法子进程） |
| `npm run build` | 构建全部 workspace |
| `npm run traces` | 重新生成预置 trace 文件（`apps/web/public/traces/`） |
| `npm run validate` | 校验预置 trace 全部符合协议 |
| `npm run demo` | 终端版演示（Node only，不启浏览器） |

## 仓库地图

```
├─ protocol/                  # 协议规范（唯一的契约层）
│  ├─ spec/v1.md              #   主规范：消息结构、seq 语义、领域扩展规则
│  ├─ spec/domains/*.md       #   领域子规范：array/linked_list/binary_tree/graph
│  ├─ schemas/*.schema.json   #   JSON Schema（Draft-07）
│  └─ examples/               #   合法 + 非法样例（校验器测试数据）
├─ packages/                  # TypeScript 库（播放系统，算法不可见）
│  ├─ protocol-core/          #   parseTrace：协议校验（结构 → 序列 → 分层语义）
│  ├─ domain-api/             #   DomainAdapter / DomainRegistry 扩展缝
│  ├─ state-engine/           #   array 领域适配器（第一阶段引擎）
│  ├─ linked-list-engine/     #   linked_list 领域适配器
│  ├─ binary-tree-engine/     #   binary_tree 领域适配器（BST 孤儿生长语义）
│  ├─ graph-engine/           #   graph 领域适配器（只读遍历领域）
│  └─ timeline/               #   Player：play/pause/step/seek/reset + 检查点重放
├─ algorithms/                # 算法脚本（零依赖，stdin→stdout，与播放系统零耦合）
│  ├─ javascript/             #   Node 运行时（8 个：排序×5、链表、BST、BFS）
│  └─ python/                 #   Python 运行时（2 个：插入排序、链表反转）
├─ apps/
│  ├─ gateway/                # 算法网关（Hono, 8787）：run 记录、trace 存储、执行
│  ├─ web/                    # Svelte 5 前端：三来源回放 + 实时执行面板 + 4 渲染器
│  └─ demo/                   # 终端演示
├─ scripts/                   # trace 生成 / 校验 / dev 并行启动
└─ docs/                      # 架构、算法契约、网关 API、阶段验收
```

## 文档索引

| 文档 | 内容 |
|---|---|
| [protocol/spec/v1.md](protocol/spec/v1.md) | 执行事件协议主规范（NDJSON、消息结构、seq、错误语义） |
| [protocol/spec/domains/](protocol/spec/domains) | 四个领域的子规范（init model / 事件 / complete 语义） |
| [docs/architecture.md](docs/architecture.md) | 分层架构、七阶段演进、扩展缝证据 |
| [docs/algorithm-contract.md](docs/algorithm-contract.md) | 算法进程契约 + 新算法/新运行时接入指南 |
| [docs/gateway-api.md](docs/gateway-api.md) | 网关 HTTP API 参考、RunRecord 模型、设计决策 |
| [docs/phase-*-acceptance.md](docs) | 各阶段验收报告（扩展性验证结论） |

## 运行环境要求

- Node.js ≥ 20（TypeScript workspace、网关、JS 算法运行时）
- Python ≥ 3.10（可选——仅 Python 算法需要；网关按需调用 PATH 上的 `python`）
