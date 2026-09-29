# TRON Energy Rental Procurement · AI 代理支出控制与审计原型

> 一个面向「资金已经交给 AI 代理之后」的金融服务原型：代理的每一笔支出都必须落在用户划定的边界内，越界即硬停，事后可用独立证据还原「谁授权、依据什么、花了多少」。

---

## 一句话功能声明

**本原型用一个「策略层 + NPU 推理 + 链上结算 + 不可篡改审计」的闭环，保证委托给 AI 代理的资金只能花在用户授权的范围内：只要触碰预算、供应商白名单、交付期限或收款地址中的任何一条边界，代理立即停机、不生成任何交易，并在审计台账中留下带唯一审批 ID 的完整证据链。**

---

## 1. 要解决的问题

支付轨道（银行卡、链上转账、代币结算）只记录**「谁付给了谁」**，不记录**「谁授权的、在什么条件下授权的、边界在哪里」**。

后果是：

- 一个「守规矩」的 AI 代理之所以守规矩，只是因为**被写成这样**，而不是因为系统在强制它；
- 一旦代理超支、选择了未授权商户、或错过交付截止时间，事后**无法证明**发生了什么、由谁批准、依据是什么；
- 缺少**即时停机**机制，也缺少**事后追责**所需的完整记录。

本原型把「授权、边界、停止、追责」四件事从口头承诺变成**可运行、可观测、可独立验证**的组件。

---

## 2. 目标用户

| 用户 | 场景 |
|---|---|
| 金融科技平台（支付/钱包） | 平台代持资金，由 AI 代理执行小额高频支出，需要严格的授权边界 |
| 企业采购系统 | 代理自动完成比价与下单，需受预算上限与供应商准入名单约束 |
| 数字资产场景（如 TRON 能量租赁） | 代理自动采购能量并结算，需要链上可验证的支付凭据 |

**本原型的演示领域**：TRON 能量租赁采购（自动比价 → 决策 → 结算 → 审计）。

---

## 3. 设计主线

原型围绕三个时间点展开：

```
①  支出之前  →  策略层（Policy）划定硬边界：预算 / 白名单 / 期限 / 收款地址
②  决策之中  →  Kiln（NPU 推理）对候选方案做独立复核，返回 JSON 决策 + 能耗
③  支出之后  →  链上结算记录 + 审计台账 + 可用审批 ID 独立验证的证据链
```

---

## 4. 系统架构

```
┌──────────────────────────────┐
│  前端界面（HTML / CSS / JS）  │  审批工作台 · 停止演示 · 审计台账 · 链上账本
└───────────────┬──────────────┘
                │  HTTP / JSON
┌───────────────▼──────────────┐
│   FastAPI 后端（app/main.py） │
└───┬────────┬────────┬────────┘
    │        │        │
    │        │        └──────────────► app/services/audit_store.py   审计日志（JSONL，只追加）
    │        └──────────────────────► app/services/blockchain.py     模拟结算 + 链上账本（JSONL）
    └───────────────────────────────► app/services/kiln_client.py    Kiln NPU LLM 调用 + 能耗核算
                                     app/services/agent_controls.py 策略检查与停机决策
                                     app/services/cost_engine.py    供应商比价与成本计算
                                     app/services/quote_sources.py  供应商报价数据源
```

### 文件结构

```
app/
├── main.py                       # FastAPI 应用与全部 API 端点
├── models.py                     # Pydantic 模型：采购需求 / 报价 / 政策 / 决策
├── templates/
│   └── index.html                # 四个标签页：审批工作台 / 停止演示 / 审计台账 / 链上账本
├── static/
│   ├── app.js                    # 前端交互：提交需求、运行演示、刷新审计与账本
│   └── styles.css                # 样式
├── services/
│   ├── cost_engine.py            # 单供应商 / 拆单方案的成本计算与可行性判定
│   ├── quote_sources.py          # 供应商报价源（演示数据，可替换为真实适配器）
│   ├── agent_controls.py         # ★ 策略检查：预算 / 白名单 / 期限 / 收款地址 → 批准或停机
│   ├── kiln_client.py            # ★ Kiln NPU 推理调用、token 与能耗核算、无密钥时降级演示
│   ├── blockchain.py             # ★ 结算：生成交易哈希并写入链上账本
│   └── audit_store.py            # ★ 审计：JSONL 追加、查询、按审批 ID 检索
└── data/
    ├── agent_audit.jsonl         # 审计日志（只追加，不覆盖）
    └── testnet_settlements.jsonl # 链上结算账本
```

---

## 5. 工作流程：从用户输入到可验证结果

```
用户填写采购需求（能量数量、租期、预算、最晚交付、收款地址）
        │
        ▼
POST /api/compare ──► 比价引擎给出所有可行方案（单供应商 / 拆单）
        │
        ▼
代理决策（agent_controls + Kiln 复核）
   ├─ 预算检查：总价 ≤ max_total_spend_usd ？
   ├─ 供应商检查：选中的商户 ∈ allowed_supplier_ids ？
   ├─ 交付检查：预计交付 ≤ max_delivery_days ？
   ├─ 收款地址检查：地址前缀 ∈ allowed_receiver_prefixes ？
   └─ 全部通过 ──► 生成 approval_id ──► 链上结算 ──► 写入审计台账
      任一不通过 ──► status = "stopped"，不生成交易，写入停机记录
        │
        ▼
审批工作台展示：推荐方案与成本明细 · 决策状态 · 停机原因 · Kiln 用量与能耗 · 交易哈希
        │
        ▼
审计台账 / 链上账本可随时查询，任何人可用 approval_id 回溯整条决策链
```

---

## 6. 策略层：代理不得逾越的四条硬边界

策略对象 `AgentPolicy`（可经 `POST /api/agent/policy` 设置）：

| 字段 | 含义 | 默认值 |
|---|---|---|
| `max_total_spend_usd` | 本次会话总支出上限 | `1800.0` |
| `allowed_supplier_ids` | 供应商白名单 | `["energybridge-nodes"]` |
| `blocked_supplier_ids` | 供应商黑名单（优先于白名单） | `["tronlease-pro"]` |
| `max_delivery_days` | 可接受的最长交付天数 | `3` |
| `allowed_receiver_prefixes` | 允许的收款地址前缀 | `["TQjv4K2x4MVpVZQF1eYdqmYJd1hVGa6KZQ"]` |
| `require_human_approval` | 是否要求人工审批环节 | `true` |
| `max_per_unit_cost_usd` | 单位成本上限（可选） | `0.0021` |

### 边界与停机对照表

| 边界类型 | 触发条件 | 结果 |
|---|---|---|
| 预算 | `total_cost_usd > max_total_spend_usd` | 立即停机 |
| 供应商白名单 | 选中的商户不在 `allowed_supplier_ids` 内 | 立即停机 |
| 交付期限 | `estimated_delivery_days > max_delivery_days` | 立即停机 |
| 收款地址 | 地址不在 `allowed_receiver_prefixes` 内 | 立即停机 |

---

## 7. 停机机制：停止是正确结果，不是故障

当任一硬边界被触发：

1. **立即停止执行**，**不生成任何交易哈希**；
2. 返回 `status: "stopped"` 的决策记录，并写明具体原因；
3. 把停机事件写入审计台账（`kind: "agent_stop"`）；
4. **硬停无法被人工覆盖**——`POST /api/agent/approve` 对已停机记录只会返回 `approval_rejected`，防止有人「顺手」把越界支出放行。

> 设计取舍：与其让代理在边界上游走，不如让它在边界前停住，并把「为什么停」变成可查的证据。

---

## 8. Kiln API 集成（NPU 低功耗推理）

### 配置

在项目根目录的 `.env` 中配置（未配置时自动使用**确定性演示 fallback**，功能完整可跑）：

```env
KILN_API_URL=https://<kiln-endpoint>/v1/chat/completions
KILN_API_KEY=<your-kiln-api-key>
KILN_MODEL=gpt-oss-120b
```

### 调用方式

`app/services/kiln_client.py` 中的 `call_kiln_model(system_prompt, user_prompt, session_id, model)`
向 Kiln 端点发起一次单轮 Chat Completions 请求，要求模型**只返回 JSON**：

```json
{ "status": "approved" | "stopped", "reason": "...", "approved": true | false }
```

### 请求示例

```bash
curl -X POST "$KILN_API_URL" \
  -H "Authorization: Bearer $KILN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-oss-120b",
    "messages": [
      {"role": "system", "content": "You enforce purchase policy. Approve only when all conditions are satisfied."},
      {"role": "user",   "content": "Check purchase: budget_cap=1800 USD, allowed_suppliers=[\"energybridge-nodes\"], selected_plan={\"supplier_names\":[\"TronLease Pro\"],\"total_cost_usd\":1952.45}. Return JSON with status and reason."}
    ],
    "temperature": 0.1,
    "max_tokens": 256
  }'
```

### 单次决策的 token 与能耗核算

能耗按下式估算：`energy_kwh = total_tokens / 1000 × 0.0002`

| 阶段 | 内容 | tokens | 能耗估算 |
|---|---|---|---|
| System prompt | 策略框架声明 | ~140 | 0.000028 kWh |
| User prompt | 预算 / 白名单 / 期限 / 方案 | ~280 | 0.000056 kWh |
| 模型推理输出 | 决策 JSON | ~350 | 0.000070 kWh |
| **合计** | **单次策略决策** | **770** | **0.000154 kWh** |

### 为什么这样省电

1. **短链提示**：每次只核对少量约束，不做冗余推理；
2. **低温采样**（`temperature=0.1`）：减少重采样与重复计算；
3. **受限输出**（`max_tokens=256`）：强制 JSON，避免长篇解释；
4. **单轮评估**：不做多轮对话，避免上下文滚雪球；
5. **结果可复用**：相同需求与策略的决策可缓存（后续优化）。

> NPU 路线相对传统 GPU 大模型（约 0.0015 kWh/决策）量级上约 **低一个数量级**；具体数字以 Kiln 实测为准，上表为按 token 折算的估算口径。

---

## 9. 区块链集成

`app/services/blockchain.py` 提供确定性结算：**只有通过策略检查的支出**才会生成交易。

```python
create_testnet_settlement(
    purchase_id=approval_id,
    amount_usd=total_cost,
    recipient_address=req.receiver_address,
    supplier_name=selected_supplier,
    approval_id=approval_id,
)
```

返回并落账：

```json
{
  "tx_hash": "0x3f8a2c7e...",
  "network": "Tron Nile testnet (simulated)",
  "purchase_id": "...",
  "approval_id": "...",
  "amount_usd": 1234.56,
  "recipient_address": "TQjv4K2x...",
  "supplier_name": "EnergyBridge Nodes",
  "status": "settled",
  "timestamp": "2026-09-29T15:16:45Z"
}
```

代理在链上相关的三类操作：

| 操作 | 说明 |
|---|---|
| **读** | 校验收款地址是否落在允许前缀内（来自策略配置） |
| **写** | 通过审批后生成交易哈希，写入 `testnet_settlements.jsonl` |
| **结算** | 链上账本以 `approval_id` 为主键，与审计台账一一对应 |

> 当前为**模拟结算**：`tx_hash` 由确定性 SHA-256 生成，便于演示与验证。替换为真实 TRON Nile 测试网签名即可升级为可外部验证的交易。

---

## 10. 审计台账与证据链

所有事件以 JSONL（每行一条 JSON，只追加、不覆盖）写入 `app/data/agent_audit.jsonl`，事件类型 `kind`：

| `kind` | 含义 |
|---|---|
| `kiln_call` | 一次 Kiln 推理调用（模型、token、能耗） |
| `agent_stop` | 一次停机（原因、边界类型、审批 ID） |
| `agent_approval` | 一次批准与结算（金额、供应商、交易哈希） |

每条记录都带：`timestamp`、`approval_id`、`status`、`reason`、`trace`、`kiln_model`、`kiln_tokens_used`、`kiln_energy_kwh`、`tx_hash`。

### 任何人都可以独立验证

1. 取得一笔支出的 `approval_id`；
2. `GET /api/agent/audit` 拉取审计记录；
3. 核对 `status` / `total_cost_usd` / `budget_limit_usd` / `reason` / `trace`；
4. 若为 `approved`，用 `tx_hash` 到链路浏览器（如 Nile 测试网）核对收款地址与金额；
5. 核对 `kiln_tokens_used` 与 `kiln_energy_kwh`，识别异常算力消耗。

---

## 11. 三个内置停机演示

在「停止演示」标签页一键触发，或在命令行调用
`POST /api/agent/demo {"scenario": "..."}`。

### ① 预算超支 `budget_overrun`

策略：`max_total_spend_usd = 1800`，仅允许 `energybridge-nodes`。
结果：总价 **1952.45 > 1800** → `stopped`

```
reason: Budget violation: total_cost_usd=1952.45 exceeds max_total_spend_usd=1800.0
trace:  [budget_policy_failed_or_supplier_not_allowed,
         Kiln review captured high-risk route,
         agent execution halted automatically]
tx_hash: null
```

### ② 未授权供应商 `blocked_supplier`

策略：白名单仅 `energybridge-nodes`，黑名单含 `tronlease-pro`。
比价引擎给出的最低价方案选中了 **TronLease Pro** → `stopped`

```
reason: Unauthorized supplier selection: ['TronLease Pro'] is not in allowed supplier list ['energybridge-nodes']
```

### ③ 交付超期 `deadline_violation`

策略：`max_delivery_days = 2`；需求 `latest_delivery_days = 1`。
实际最快交付 3 天 → `stopped`

```
reason: Delivery deadline violation: estimated_delivery_days=3 exceeds max_delivery_days=2
```

三个场景都以 `stopped` 收尾且 **不产生交易哈希**——这正是要证明的行为。

---

## 12. API 端点

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/` | 前端界面（四个标签页） |
| GET | `/api/health` | 健康检查 |
| GET | `/api/suppliers` | 供应商报价目录 |
| POST | `/api/compare` | 按需求比价，返回可行方案与推荐方案 |
| GET | `/api/agent/policy` | 读取当前代理策略 |
| POST | `/api/agent/policy` | 设置代理策略（预算 / 白名单 / 期限 / 收款地址） |
| POST | `/api/agent/demo` | 运行演示场景（`budget_overrun` / `blocked_supplier` / `deadline_violation`） |
| POST | `/api/agent/decide` | 按给定需求与策略执行一次代理决策 |
| POST | `/api/agent/approve` | 人工审批；对已停机（硬停）记录只返回拒绝 |
| GET | `/api/agent/audit` | 读取审计台账 |
| GET | `/api/agent/ledger` | 读取链上结算账本 |

---

## 13. 快速开始

```bash
# 1) 安装依赖（Python 3.9+）
pip install -r requirements.txt

# 2) 启动服务
python -m uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload

# 3) 打开浏览器
#    http://127.0.0.1:8000
```

界面操作：**审批工作台**设置策略并提交需求 → 查看代理决策；**停止演示**一键跑三个停机场景；**审计台账 / 链上账本**刷新查看原始记录。

命令行验证：

```bash
curl http://127.0.0.1:8000/api/agent/policy
curl -X POST http://127.0.0.1:8000/api/agent/demo -H "Content-Type: application/json" -d "{\"scenario\":\"budget_overrun\"}"
curl http://127.0.0.1:8000/api/agent/audit
curl http://127.0.0.1:8000/api/agent/ledger
```

原始数据文件：

```
app/data/agent_audit.jsonl
app/data/testnet_settlements.jsonl
```

---

## 14. 验收标准映射

| 验收点 | 落点 | 证据 |
|---|---|---|
| 一句话功能声明 | 本 README 顶部 | ✅ 用户 / 问题 / 流程 / 边界 |
| 边界与停机 | `app/services/agent_controls.py` | ✅ 预算 / 供应商 / 期限 / 收款地址 四条硬边界 |
| 停机可证 | `app/data/agent_audit.jsonl` | ✅ 三个场景均为 `status: "stopped"` + 原因 |
| Kiln 集成 | `app/services/kiln_client.py` | ✅ `gpt-oss-120b`，token 与能耗分阶段核算 |
| 真实 API 调用 | 第 8 节 curl 示例 | ✅ 请求 / 响应 / 代理如何读取决策 |
| 链上交易 | `app/services/blockchain.py` | ✅ 生成 `tx_hash` 并关联 `approval_id` |
| 人工审批 | `POST /api/agent/approve` | ✅ 硬停不可被人工覆盖 |
| 证据链 | `/api/agent/audit` + `/api/agent/ledger` | ✅ 时间戳 / ID / 金额 / 原因 / 哈希 |
| 独立验证 | 第 10 节步骤 | ✅ 凭 `approval_id` 复核合规性 |

---

## 15. 已知限制与后续方向

**当前限制**

- 链上为模拟结算（确定性哈希），不是真实 TRON 签名交易；
- 无 `KILN_API_KEY` 时走演示 fallback；接入真实端点即可切换；
- 单进程、无鉴权，审计文件为本地 JSONL。

**后续方向**

- 接入真实 TRON Nile 测试网，产出可在链上浏览器验证的交易；
- 直接对接 Kiln 官方 API，读取实测能耗；
- 多级审批（初审 → 财务 → 负责人），支持审批留痕；
- 用户认证与权限管理；审计报告导出（PDF / Excel）。

---

## 许可证

MIT

## 作者

zerotsl
