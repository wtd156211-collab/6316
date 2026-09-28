# 批处理语义与错误分层（从 0 实现）

起始环境只有这份说明和 `samples/` 里的素材（批次请求、逐行期望结果、现场记录）。代码从零写，只允许 Python 3.13
标准库：引擎是一个包（库 + `python -m batchwire` 入口），页面放在 `web/` 下。

## 1. 范围

做的：批处理语义（逐条结果 + 整体状态）；协议层与业务层的错误分层；批次上限与切分合并；方法表 `add`、`div`、
`stats`；`web/` 下的演示页（原生 HTML + Canvas，无构建、无依赖）；测试用 `unittest`。

不做：真实服务与网络收发（HTTP 客户端、超时、重试、鉴权、限流）、持久化与队列、并发与异步、方法表之外的
业务、第三方依赖与构建步骤、日志与监控、页面样式之外的工程化。

## 2. 口径与公式

- 批次 = 一个 JSON 数组，元素按 0 开始编号；处理顺序、结果顺序都按原始下标升序，切分不改顺序。
- 上限 `LIMIT = 8`：超过 8 条时按下标切连续段（`[0,8)`、`[8,16)`…），逐段处理再合并。切分只发生在两条请求
  之间：一条请求（含 `id` 与参数）必须完整落在同一段里，结果的 `index` 用原始下标，不因切分重置。
- 调用与通知：请求对象缺 `id` 键的是通知，一律不产生结果条目（成功、失败、协议问题都不产生），只计入
  `skipped`；其余元素都必须恰好产出一条结果条目。
- 子批状态：段内有任意 `error` 条目就是 `failed`，否则 `ok`。
- 整体状态：所有段都 `ok` 才是 `ok`，否则 `failed`（空批次算 `ok`）。整批 `failed` 不删减逐条结果，前端照样
  能拿到 `ok` 的那些条目。
- 计数：`total` = 批内元素个数（含通知），`ok + error + skipped = total`，段数 = `ceil(total / 8)`（空批次 0）。
- 数字指 JSON number（整数或浮点），布尔不算；`div` 用 Python 除法，浮点按最短往返表示；`stats` 的
  `sum`/`min`/`max` 保持整数或浮点。请求里多余的字段忽略，`id` 不要求唯一。

## 3. 状态机与数据结构

请求对象：`id`（可选，通知的判据）、`method`（方法名）、`params`（数组，可缺省）。每个元素按原始下标 `i` 依次走：

1. 不是 JSON 对象 → 协议错误 `E_PROTO_ITEM`。
2. 是对象且没有 `id` 键 → 通知，不产出条目。
3. `id` 不是非空字符串、也不是整数（布尔不算）→ 协议错误 `E_PROTO_ID`。
4. `method` 缺失或不是非空字符串 → `E_PROTO_METHOD`；不在方法表里 → `E_PROTO_UNKNOWN`。
5. `params` 缺失按 `[]` 处理；存在但不是数组 → `E_PROTO_PARAMS`。
6. 调方法：先查个数或长度，再从左到右查类型，最后查取值；失败给业务错误，成功给 `result`。

| method | params | 结果 | 业务错误 |
| --- | --- | --- | --- |
| `add` | 恰好 2 个数字 | `a + b` | `E_BIZ_ARITY`（`at = null`）、`E_BIZ_TYPE`（`at` 是首个非数字下标） |
| `div` | 恰好 2 个数字 | `a / b` | 同 `add`，外加 `E_BIZ_DIV_ZERO`（`b = 0`，`at = 1`） |
| `stats` | 1–4 个数字 | `{"count": n, "sum": s, "min": m, "max": M}` | `E_BIZ_EMPTY`（空数组）、`E_BIZ_TOO_LONG`（超 4 个，`at = 4`）、`E_BIZ_TYPE` |

两层错误的字段和错误码都不一样：协议层 `{"layer": "protocol", "code": "E_PROTO_…", "field": "item|id|method|params"}`，
`field` 指向出问题的请求字段；业务层 `{"layer": "business", "code": "E_BIZ_…", "at": <params 下标或 null>}`，
`at` 指向首个不合规参数，个数或长度不对时为 `null`。

结果条目键序固定：`{"index": <原始下标>, "id": <合法 id 或 null>, "status": "ok", "result": …}`，失败时
`status` 是 `"error"`、把 `result` 换成 `error`；`E_PROTO_ITEM` 与 `E_PROTO_ID` 的 `id` 写 `null`。

## 4. 输入输出与文件格式

```
python -m batchwire <批次文件>        # 在仓库根目录执行
```

读 UTF-8 JSON 文件，顶层必须是数组；读不了、解析不了、不是数组都算批级失败：标准输出为空，标准错误一行说明，
退出码 `2`。正常处理完把结果 JSON 写到标准输出，退出码 `0`（整体 `ok`）或 `1`（整体 `failed`）。结果 JSON 为
UTF-8、无 BOM、`indent=2`、`ensure_ascii=False`，顶层键序 `status`、`summary`、`chunks`、`results`，末尾一个换行；
逐字节格式以 `samples/expected/` 为准，结构如下：

```json
{
  "status": "failed",
  "summary": {"total": 2, "ok": 1, "error": 1, "skipped": 0},
  "chunks": [{"index": 0, "start": 0, "count": 2, "status": "failed"}],
  "results": [
    {"index": 0, "id": "x", "status": "ok", "result": 3},
    {"index": 1, "id": "y", "status": "error", "error": {"layer": "business", "code": "E_BIZ_DIV_ZERO", "at": 1}}
  ]
}
```

`chunks[i]` = `{"index": i, "start": <段首原始下标>, "count": <段内元素个数>, "status": "ok"|"failed"}`，页面画
分隔线用它。库接口 `batchwire.process_batch(batch) -> dict`（`batch` 是解析好的数组）是命令行、页面、测试的共同入口。

演示页：跑 `python -m batchwire serve`（默认 127.0.0.1:8000），打开 `http://127.0.0.1:8000/` 就是 `web/index.html`。
页面必须从引擎拿输出（批次 JSON 进去、结果 JSON 出来），把批次画成时间轴：每条请求一个刻度（通知灰色），成功、
业务错误、协议错误三种配色，段与段之间画分隔线，旁边显示整体状态和 `summary` 统计；刻度按 `results` 的 `index`
对号入座，没有条目的就是通知。颜色、分隔线、统计不许硬编码。

## 5. 性能与验收口径

- 性能：10^4 条请求的批次，单进程、含读写，2 秒内跑完；内存与批大小同阶。
- 确定性：同一批次跑两遍，标准输出逐字节相同；不读时钟、不用随机数、不看环境变量、不依赖并发调度。
- 约束：只用标准库；不联网、不加构建步骤、不改 `samples/` 与 `README.md`；不得照样例写死输出。

1. 对 `samples/batches/` 每个文件跑 `python -m batchwire samples/batches/<名>.json`，标准输出与
   `samples/expected/<名>.json` 逐字节相同，退出码与整体状态一致（`ok` → 0，`failed` → 1）。
2. 逐条结果的下标、顺序、`id`、`status`、`result` 与 `error` 逐项对得上；协议层的 `field`、业务层的 `at` 各就各位。
3. 通知不产生条目但计入 `skipped`；空批次整体 `ok`；超过 8 条的批次按 8 切段，`chunks` 的 `start`/`count`/`status` 正确。
4. 页面：`serve` 之后任选一个样例批次，时间轴刻度数、配色、分隔线、统计与期望结果一致。
5. 自带的 `unittest` 用例能跑通（只读 `samples/`，不联网）。

## 6. 样例说明

`samples/batches/` 里 7 批请求，`samples/expected/` 里是同名的逐行期望结果（顺序敏感）：

- `01-all-ok.json`：5 条全成功，1 段。
- `02-business-errors.json`：8 条 1 段；覆盖 `params` 缺失、个数、类型、除零、空数组、超长。
- `03-protocol-errors.json`：8 条；非对象元素、`id` 浮点、`method` 缺失/非字符串、未知方法、`params` 非数组。
- `04-notifications.json`：7 条里 5 条通知（含自身失败的），只有 2 条调用有结果。
- `05-empty.json`：空批次，`total = 0`、0 段、整体 `ok`。
- `06-split.json`：20 条切 3 段（8/8/4），中间一段干净、两头有错，整体 `failed`。
- `07-limit-boundary.json`：9 条切 2 段（8/1），第二段里有一条 `stats` 超长。

`samples/notes.md` 是现场记录，只讲这次整批失败怎么被发现。

## 7. 待补的文档

方法表的版本与兼容性（加方法、改参数、加错误码怎么升级）、真实服务的调用契约（超时、重试、幂等）、批次来源
与归档、页面之外的可视化需求都还没定。
