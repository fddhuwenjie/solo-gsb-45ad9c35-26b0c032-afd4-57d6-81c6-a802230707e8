# 事务交错推演（Transaction Interleaving Playground）

用 Python + FastAPI + 原生 HTML/CSS/JavaScript 构建的事务调度可视化网页：
编辑初始键值与 2～3 条事务脚本，系统按**轮转调度**（依次从各事务队头取操作）
排出同一份全局调度，再分别在三种隔离级别下生成确定性轨迹并逐步展示。

## 隔离级别语义

| 隔离级别 | 读取规则 |
|---|---|
| **Read Uncommitted (RU)** | 读键上的**最新写入**，包括其他事务尚未提交的版本；写入方若随后 ABORT，该读列入「脏读清单」 |
| **Read Committed (RC)** | 每次读只取键上的**最新已提交版本**（不读未提交数据，亦不看自身未提交的写） |
| **Snapshot Isolation (SI)** | 读 **BEGIN 时的快照**，自身写入优先；BEGIN 之后同键若已被其他事务提交，提交时按**首提交者胜**整笔回滚 |

页面支持：

- 逐步 / 自动播放每一条调度操作，展示**读值、来源版本、写入者**与逐条解释；
- 每一步刷新**版本链**（初始 / 未提交 / 已提交 / 已回滚版本，读写位置高亮）与事务状态；
- 按**已提交事务的真实读写关系**生成三种依赖边：
  - `WR`：Tj 读到 Ti 已提交版本（Ti → Tj）；
  - `WW`：同键上 Ti 先写、Tj 后写且均提交（Ti → Tj）；
  - `RW`（反依赖）：Ti 读过某版本后，同键出现 Tj 的更新已提交版本（Ti → Tj）；
  - 图中发现环时，环上的边与对应的读/写操作红色高亮；
- 单列「**脏读清单**」：读到后来被 ABORT 的未提交数据；
- 内置四个一键加载示例：脏读、丢失更新（对比 RC 提交成环与 SI 回滚）、写偏序（SI 下双双提交但 RW 成环）、同键交错提交（先写者晚提交时的 RC 读值与依赖图）。

## 事务脚本语法

每条脚本一行，仅允许以下操作（大小写不敏感）：

```
BEGIN
READ key
WRITE key value
COMMIT
ABORT
```

- 必须以 `BEGIN` 开头，以 `COMMIT` 或 `ABORT` 结尾，结束语句之后不能再有操作；
- 值支持整数、小数、`true` / `false` / `null` 与字符串；
- 初始键值每行一个 `key=value`；
- 勾选「启用 T3」可使用 3 条事务。

## 同键交错提交示例（先写者晚提交）

页面中的「示例4 · 同键交错提交」演示以下交错（初始 `x=0, y=0`，
`y` 仅用于在轮转调度中占位）：

```
T1: BEGIN; WRITE x 1; READ y; COMMIT
T2: BEGIN; WRITE x 2; COMMIT
T3: BEGIN; READ y; READ y; READ x; COMMIT
```

轮转调度后的关键序列：T1 先写 `x=1`（v3），T2 后写 `x=2`（v4），
但 **T2 先提交（第 8 步）、T1 后提交（第 10 步）**，之后 T3 在第 11 步读 `x`。

| 隔离级别 | T3 读到 | 依据 |
|---|---|---|
| RU | **2**（T2） | 始终取版本链上最新写入 |
| RC | **1**（T1，最后提交者） | “最新已提交”按**提交时刻**判定，而非版本编号/写入顺序；提交前未提交值仍不可读 |
| SI | **0**（BEGIN 快照） | T3 开始时无任何提交版本；T1 因 T2 已先提交同键，按首提交者胜**整笔回滚** |

- **版本链保持写入顺序**：`x = v1(0, 初始) → v3(1, T1, 第10步提交) → v4(2, T2, 第8步提交)`，
  提交次序与链位置不同；RC 选 cseq 最大的 v3，而不是链尾的 v4。
- **依赖图（RC）**：`WR T1→T3`（T3 实际读到 T1 提交的版本）、
  `WW T1→T2`（反映真实写入顺序，与提交先后无关）；
  **不**会仅因 v4 编号较大就添加 `T3→T2` 的 RW 边——RW 反依赖只在
  “读完之后该键才有他人提交更新版本”时产生。
- SI 下回滚的 T1 不参与依赖图，其 v3 标记为已回滚、对任何隔离级别都不再可读。

## 一键启动（Docker Compose）

```bash
docker compose up --build
```

启动后访问：

- **http://localhost:8000** —— 推演网页
- http://localhost:8000/api/health —— 健康检查

后台停止：`docker compose down`。

## 本地直接运行（可选，无需 Docker）

需要 Python 3.11+：

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

然后访问 http://localhost:8000 。

## 接口

`POST /api/simulate`

```json
{
  "initial": "x=0\ny=0",
  "txns": [
    "BEGIN\nREAD x\nWRITE x 10\nCOMMIT",
    "BEGIN\nREAD x\nWRITE x 20\nCOMMIT"
  ]
}
```

返回三种隔离级别各自的 `steps`（含每步版本链快照、读值来源、事务状态）、
`dirty_reads` 与 `graph`（节点 / WR·WW·RW 边 / 环路径）。
