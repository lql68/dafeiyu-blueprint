# 07 · 时间感知与时间连续性

> 「她知道现在几点，但不知道**上一次说话是什么时候**；记忆里存着时间戳，但那是**写入时刻**不是**事情发生的时刻**。」

这个目录记录从「有时间戳」走到「有时间感」的整个过程 —— 两轮只读审计、一条设计核查、四次交付。

---

## 为什么需要

审计（`阶段3-A-时间化记忆-只读审计报告.md`）查出来的三个病：

| 病 | 数据 |
|---|---|
| **`event_time` 几乎全空** | 1449 条 atom 里只有 24 条有（1.7%），其中 13 条 `\|created_at − event_time\| < 2 秒` —— 等于把记录时间当事件时间 |
| **主链路没有时间过滤** | `search_memories(query, k, session_id, persona_id)` 就四个参数，四个检索器都没有时间位 |
| **零时区、零单调时钟** | 全项目 `ZoneInfo` / `pytz` / `astimezone` / `monotonic` 各 0 处 |

根因在 `atom_classifier.py`：

```python
_TIME_INDICATORS = re.compile(r"明天|昨天|今天|上午|下午|晚上|…")
# 「下午在看书」→ has_time=True（命中「下午」），但 _parse_event_time() 返回 None
# → 落到 if has_action: return EPISODIC, 0.75, None —— 事件时间丢掉
```

还有一个语义 bug：`mapping = {"昨天": -86400}` → `now + offset` 是「现在减 24 小时」，**不是昨天零点**。精度 ±1 天，却存成秒级 float。

---

## 做了什么

### 1. `temporal.py` —— 统一时钟

`dafeiyu-core/src/dafeiyu/kernel/temporal.py`（588 行，零第三方依赖）

```python
t.now()                       # 2026-11-17 20:30:00+08:00（带时区）
t.now_utc()                   # epoch 秒，持久化用这个
t.mono()                      # 单调时钟，算耗时用这个（不受系统改时间影响）
t.band()                      # 早晨/上午/中午/下午/傍晚/晚上/深夜 七档
t.weekday_cn()                # 周一…周日
t.elapsed(ts)                 # 回拨返回 0.0（elapsed_raw 保留负值）
t.fmt_gap(3700)               # "1 小时 1 分"
t.offline(last_active_ts)     # {seconds, raw, text, is_restart, is_long, clock_went_back}
t.context(last_active_ts)     # ≤60 字，给 prompt 用
```

**时区策略**：`DFY_TZ` 环境变量，缺省 `Asia/Shanghai`。`zoneinfo` 不可用时退到固定偏移表 —— **表里只放全年无夏令时的时区**（`Asia/Shanghai: 8.0`、`Asia/Tokyo: 9.0`、`UTC: 0.0` 等 11 个）。有夏令时的一个都不放，找不到就抛 `TzError`，**不猜**。

**两种时钟不许混**（`TestTwoClocksNeverMixed`）：

| | 用途 | 特征 |
|---|---|---|
| `now_utc()` | 持久化、比较、算日期边界 | 会被系统改时间影响 |
| `mono()` | 算耗时、超时 | 不受系统改时间影响，但**不能持久化** |

把 `mono()` 喂给 `elapsed()` 会算出几十年；`mono_elapsed()` 在系统改时间后照常准。

### 2. `parse_cn_time()` —— 中文时间解析

四档精度 + 三档时序：

```python
t.parse_cn_time("昨天下午在看书")
# {'start': 1793..., 'end': 1793..., 'precision': 'exact',
#  'status': 'past', 'label': '昨天下午', 'source': 'day+band', 'tz': 'Asia/Shanghai'}
```

| 表达 | precision | 说明 |
|---|---|---|
| `2026年11月10日` | `exact` | |
| `5月30日` | `approximate` | 年份未说明，取「今年或明年最近一次」 |
| `昨天 / 前天 / 大前天 / 今天 / 明天 / 后天 / 大后天` | `exact` | 配时段词时取时段区间 |
| `上周三 / 下周二` | `exact` | 周一起算 |
| `上周 / 本周 / 下下周` | `exact` | 整周区间 |
| `上个月 / 下个月` | `exact` | 跨年用 `y*12+m` 索引运算 |
| `去年夏天` / `今年冬天` | `approximate` | 季度区间；冬天跨年 `(y,12,1)-(y+1,3,1)` |
| `前年 / 去年 / 今年` | `exact` | 整年 |
| `前段时间（30天）/ 最近（14天）/ 前几天（7天）` | `relative` | |
| `小时候 / 以前 / 有一天` | `unknown` | |
| **纯时段词「下午」** | `unknown` | label 写「下午（具体哪天没说）」，**不默认成今天** |

`query_range(text)` → `(start, end) | (None, None)`。**`precision == unknown` 时不返回区间** —— 「下午」不筛，避免漏掉别的天。

**明确不做**：不给历史记忆补造时间；不把 `relative` 升级成 `exact`；`autobiography.from_ts/to_ts = 0` 一直表示未知。

### 3. 时间化记忆 —— 两列 + 一个过滤器

`memory_atoms` 表 **19 列 → 22 列**（`ALTER TABLE ADD COLUMN`，幂等，两轮验证）：

```
event_time_end  REAL
time_precision  TEXT  DEFAULT 'unknown'
temporal_status TEXT  DEFAULT 'unknown'
+ CREATE INDEX idx_atoms_event_time ON memory_atoms(event_time)
```

**存量 1449 条全部 `unknown`** —— 不批量猜测、不回填。

`filter_by_event_time(results, start_ts, end_ts)` 三分类：

| 类 | 判据 |
|---|---|
| `matched` | 区间有重叠（**左闭右开**：`ev_end < start_ts` 才算排除，点事件在区间起点要算命中） |
| `unknown` | `precision == "unknown"` 或没有 `atom_event_time`，**保留不删**，排在 `matched` 后面 |
| `excluded` | 明确时间但不在区间 |

**只认 `atom_event_time`**（atom 层从文本解析出来的），document 层的 `source_time_start` **不冒充事件时间**。

### 4. 主链路接入

两条通路都接了：

| 通路 | 接入点 |
|---|---|
| QQ 对话 | `memory_recall.py:263` 先 `_tp_query_range(query)` 再传 `start_ts/end_ts` |
| 网页通话 | `/recall` 里 `self._temporal().query_range(q)` |

过滤在 `search_memories()` **末尾、两路合并之后**做一次 —— 文档层和 atom 层都在里面。

**缓存 key 必须带时间**：`_search_cache_key(query, k, session_id, persona_id, start_ts, end_ts)`，否则「上个月」会命中「上周」的缓存。

**发现过一个只能靠端到端测出来的 bug**：`atom_time` 的收集原本写在 `if atom.parent_memory_id not in existing_ids:` **里面** —— hybrid 已命中的文档拿不到 `atom_event_time`，时间过滤对主命中永远无效。25 个单测全过，因为它测的是函数本身，不是接进来的数据。

### 5. `scheduler.py` —— 后台任务预算

```python
s = Scheduler(root)
if s.allow("proactive", min_gap=0, max_per_day=6):
    ...
    s.done("proactive", ok=True, note="说了一句")
```

失败指数退避 `60 × 2^(n-1)`，封顶 1 小时；成功清零。`pause()` / `resume()` 全局暂停。
状态 JSON **原子写**（`.tmp` → `os.replace`），坏文件不崩。

接到五个 loop：`world_walk` / `reading` / `world_health` / **`proactive`（6/天）** / **`heartbeat`**。
`school` / `lymph` / `observe` 没接 —— 各有等价预算（`daily_cap=5` / 睡眠+6h 门控 / 纯读不调模型）。

---

## 踩过的七个坑

| # | 坑 | 现象 |
|---|---|---|
| 1 | **「去年夏天」被「去年」抢先** | 整年规则先匹配 → 季节块必须挪到整年之前 |
| 2 | **「大后天」被「后天」抢先** | dict 迭代顺序问题 → `sorted(_DAY_WORDS, key=lambda kv: -len(kv[0]))` 按词长降序 |
| 3 | **左边界写成 `<=`** | 点事件落在区间起点被排除 → 改 `ev_end < start_ts` |
| 4 | **`atom_time` 只在补充分支收** | 主命中的文案永远进 unknown（只有端到端能发现） |
| 5 | **`AtomStore.insert()` 会重算 `created_at`** | 测试里只 override `expires_at` 没用，要连 `created_at` 一起 UPDATE |
| 6 | **迁移不能 import 插件** | `atom_store` → `core.models` → `core.managers` → `memory_engine` → `storage.atom_store` 循环导入 → 直接执行 DDL |
| 7 | **PowerShell `-Body` 按 GBK 发中文** | 服务端收乱码 → `query_range` 解析不出 → `time_filter` 恒 `null`。正解 `[Text.Encoding]::UTF8.GetBytes($json)`（**不是代码 bug，是验证方法错了**） |

---

## 测试

```
dafeiyu-core                248 passed
  test_temporal.py          107
  test_temporal_memory.py    56
  test_scheduler.py          17
  （其余既有）
livingmemory                 87 passed
  test_memory_atom.py        62
  test_p3_event_time.py      25
端到端六场景                 39 / 39 PASS
```

`pytest tests/` 全量有 2 个文件会卡死 —— `test_faiss_path_bridge.py` / `test_faiss_async_persist.py`：**`faiss.write_index()` 在 Windows + 中文路径下挂住**（`faiss_path_bridge.py` 本身就是绕开它的）。与时间系统无关，逐个文件跑其余 73 个全过。

---

## 文档索引

| 文档 | 内容 |
|---|---|
| [阶段2-交付报告](阶段2-交付报告.md) | 时间底座接进 `voice_server`：`_time_bit()` + `/temporal` |
| [阶段2.5-交付报告](阶段2.5-交付报告.md) | QQ 通路：AstrBot 本来就在注入日期，插件只补昼夜（`on_llm_request` 钩子），58 条实测日志 |
| [阶段3-A-时间化记忆-只读审计报告](阶段3-A-时间化记忆-只读审计报告.md) | 语料分布、`event_time` 1.7% 的根因、已有的四个时间能力、四个缺失 |
| [阶段3-B-设计核查报告](阶段3-B-设计核查报告.md) | 主链路四个检索器逐个签名、`source_time_*` 是消息时间、测试缺 8 项 |
| [阶段3-交付报告](阶段3-交付报告.md) | 七处改动、DB 迁移、13 类解析能力、A–H 八节 |
| [阶段3-验收补充报告](阶段3-验收补充报告.md) | 全量测试定位（faiss 卡死）+ A/B 证明 9 个失败是历史遗留 + 端到端 4 项 + 修掉 `atom_time` 那个真 bug |
| [V1.0-交付报告](V1.0-交付报告.md) | 七模块总验收、六场景 39 项、调度器、成本控制、启动与回滚 |

代码在 [`lql68/dafeiyu-core`](https://github.com/lql68/dafeiyu-core)：
`src/dafeiyu/kernel/temporal.py` / `scheduler.py` + 三个测试文件。
