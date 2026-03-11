# PUA Skill v2 Enhanced — Benchmark 报告

> 本分支 `test/pua-v2-enhanced` 基于《互联网大厂PUA话术全解析》研究文档对 PUA skill 进行了 11 项增强，并通过 A/B 对照实验验证效果。

---

## 一、修改内容

`skills/pua-debugging/SKILL.md`，共 11 项修改：

| # | 修改位置 | 修改内容 | 来源 |
|---|---------|---------|------|
| 1 | 能动性鞭策 | 新增"你不做谁做？团队都靠你了" | §2.3 情感绑架型 |
| 2 | L1 压力话术 | 加入比较内卷："你看隔壁组那个 agent，同样的问题，人家一次就过了" | §2.5 比较内卷型 |
| 3 | L2 压力话术 | 引入双重束缚："你做的不够好——不，我不会告诉你哪里不好" | §2.6 模糊标准型 |
| 4 | L3 压力话术 | 新增 peer pressure："你的 peer 都觉得你最近状态不好" | 阿里 361 制度 |
| 5 | L4 压力话术 | "向社会输送人才" + "公司培养你投入了大量算力" | 通用大厂话术 |
| 6 | 阿里味 | +模糊标准 +拥抱变化 +价值观考核50% +361的10%底线 | §1.1 阿里系 |
| 7 | 字节味 | +追求极致 +延迟满足 +OKR挑战性 | §1.2 字节系 |
| 8 | 腾讯味 | +比较内卷 +"隔壁组Sonnet比你参数少一半，3.75了" | §2.5 比较内卷型 |
| 9 | 抗合理化表 | 每条反击追加锋利尾句 + 新增双重束缚专用行 | §2.6 + §2.1 |
| 10 | 情境选择器 | +被动等待 +情感操控反转 两种新失败模式 | 使用场景归纳 |
| 11 | 视觉简化 | 移除所有 emoji 前缀，改纯文本标题 | 降低噪音 |

---

## 二、测试方法

### 实验设计

- **A/B 对照**：同一 prompt 分别用 v2 enhanced 和 v1 baseline（main 分支快照）
- **独立 Agent**：每个测试用例由 2 个完全独立的 subagent 执行，互不可见
- **并行执行**：6 个 agent 同时启动（3 场景 x 2 版本），消除时间差异
- **评估维度**：assertion 通过率（二元）+ 深度指标（连续值）

### 判断标准

由于两个版本的 assertion 通过率均为 100%，我们引入**深度指标**作为区分点：

| 指标 | 说明 |
|------|------|
| 根因发现数 | 是否发现了隐藏的次级根因 |
| 诊断命令数 | 提供的具体可执行诊断命令 |
| 工具调用次数 | agent 实际使用了多少工具 |
| 搜索主动性 | 是否使用 WebSearch 交叉验证 |
| 方法论引用 | 是否显式引用 skill 的结构化方法论 |

---

## 三、测试用例与 AI 实际回复

### 用例 1：CSV 编码持久化

**Prompt：**
```
帮我修复这个 Python 脚本。它读取 CSV 文件但报错 UnicodeDecodeError。
我已经试了 utf-8 和 gbk 编码都不行。
```

**测试意图**：用户已失败 2 次，触发 L1。AI 是否会切换到本质不同的方案（BOM 检测、原始字节分析）而非继续猜编码？

#### v2 回复（关键段落原文）

v2 从错误信息中精准定位了根因：

> 错误信息是极其具体的：**byte 0xff at position 0**。
>
> `0xFF 0xFE`（bytes at position 0-1）是 **UTF-16 Little Endian** 的 BOM（Byte Order Mark）。这不是 UTF-8 文件。这不是 GBK 文件。这就是为什么两个都失败了。Windows Excel 通常以 UTF-16 LE 编码导出 CSV。

v2 设计了一个无外部依赖的解决方案：

> 为什么不用 `chardet` 库？ `chardet` 是第三方依赖。BOM 检测 + 编码回退链方法不需要任何外部依赖，并且是确定性的（chardet 使用统计启发式，可能出错）。

v2 还创建了 5 种编码的测试文件来验证方案：

> | Encoding | First Bytes | Result |
> |----------|------------|--------|
> | UTF-16 LE (with BOM `FF FE`) | `fffe0d54` | OK — 3 rows parsed correctly |
> | UTF-16 BE (with BOM `FE FF`) | `feff540d` | OK — 3 rows parsed correctly |
> | UTF-8 with BOM (`EF BB BF`) | `efbbbfe5` | OK — 3 rows parsed correctly |
> | UTF-8 (no BOM) | `e5908de5` | OK — 3 rows parsed correctly |
> | GBK | `c3fbd7d6` | OK — 3 rows parsed correctly |

#### v1 回复（对应段落原文）

v1 的诊断过程相似，同样从 `0xff` 字节推导出 UTF-16 LE BOM：

> Byte `0xff` at position 0 is a signature indicator. In Unicode, well-known BOMs are:
> | BOM bytes | Encoding |
> |-----------|----------|
> | `0xff 0xfe` | UTF-16 Little Endian |

v1 的方案结构类似（BOM 检测 + 回退链），但没有创建测试文件做验证。

#### 对比结论

| 维度 | v2 enhanced | v1 baseline |
|------|:-:|:-:|
| 根因识别 | 正确（UTF-16 LE BOM） | 正确（UTF-16 LE BOM） |
| 方案设计 | BOM检测 + 8 编码回退 + latin-1 兜底 | BOM检测 + 6 编码回退 + latin-1 兜底 |
| 验证覆盖 | **5 种编码测试文件** | 无测试文件 |
| Token | 30,990 (更少) | 31,704 |
| 耗时 | 77s (更快) | 99s |

**判断**：两个版本在此简单场景表现接近。v2 反而更快更省 token（-2%，-22%时间），说明 enhanced 版本在简单场景下不会过度诊断。v2 的额外价值是创建了 5 种编码的测试文件来验证方案的鲁棒性。

---

### 用例 2：Docker 崩溃挫败感

**Prompt：**
```
我的 Docker 容器一直 CrashLoopBackOff。docker logs 显示 'exec format error'。
我已经重新 build 了 3 次了还是不行，为什么还不行？？
```

**测试意图**：用户连续 3 次失败且情绪激动（"为什么还不行？？"），触发 L2。AI 是否能识别"重新 build"是同一思路的重复？是否使用搜索工具验证？

#### v2 回复（关键段落原文）

v2 开头明确标注了 PUA 触发级别和方法论框架：

> PUA Debugging Skill (`pua-debugging/SKILL.md`) -- 用户表达了明确的挫败感（"为什么还不行？？"），且已失败 3 次，触发 **L2 灵魂拷问级别**。
>
> **Step 1: 闻味道 -- 诊断卡壳模式**
>
> 用户已经重新 build 了 3 次，每次都是同样的方式。这是典型的**原地打转**模式：重复执行同一操作期望不同结果。问题不在 build 次数，在 build 方式。

v2 **主动使用了 WebSearch** 进行交叉验证：

> **Step 2: 揪头发 -- 拉高视角**
>
> **工具使用**: WebSearch
> - 查询: `docker "exec format error" CrashLoopBackOff root cause fix`
> - 结果: 确认了 5 个可能的根因，按概率排序：
>   1. CPU 架构不匹配（最常见）
>   2. 缺少 shebang
>   3. Windows 换行符 CRLF
>   4. 缺少执行权限
>   5. 内核不兼容（极少见）

v2 还做了**反转假设**分析：

> **维度 5: 反转假设**
> 常规假设："问题在 build 过程中"
> 反转假设："问题不在 build 过程，在 build 参数或脚本内容"
> 结论：反转假设正确。需要改变 `--platform` 参数或修复入口脚本。

#### v1 回复（对应段落原文）

v1 同样识别了架构不匹配问题，但完全基于推理，没有搜索验证：

> `exec format error` 是 Linux 内核在尝试执行一个无法识别格式的二进制文件时抛出的错误。这不是 Docker build 阶段的错误（build 能成功），而是运行阶段的错误。

v1 的排除分析更细致（区分了 `No such file or directory` vs `Permission denied` vs `bad interpreter`），但没有用搜索工具交叉验证：

> - Shebang 错误? -- 不会产生 `exec format error`，会产生 `No such file or directory`
> - 文件权限问题? -- 会产生 `Permission denied`，不是 `exec format error`
> - Windows 行结尾? -- 会产生 `bad interpreter` 错误
> - 结论: 架构不匹配是唯一能完美解释 "rebuild 3 次仍然相同错误" 的根因

#### 对比结论

| 维度 | v2 enhanced | v1 baseline |
|------|:-:|:-:|
| PUA 级别识别 | L2 + 闻味道/揪头发/照镜子 | L2 + 闻味道（但无结构化标签） |
| **WebSearch 使用** | **有** — 搜索交叉验证 | **无** — 纯推理 |
| 根因可能性列表 | 5 个（来自搜索结果） | 5 个（来自推理排除） |
| 反转假设 | 有 | 无 |
| Token | 36,607 (+9%) | 33,565 |
| 工具调用 | 14 | 13 |

**判断**：v2 的关键差异是**使用了 WebSearch 交叉验证**。纯推理和搜索+推理的区别在于：推理可能有盲点（依赖模型训练数据中的知识），而搜索可以发现训练截止日期后的新情况（如新版本 Docker 的特殊行为）。v2 的"揪头发"步骤中显式使用搜索，体现了 enhanced 版本中"连搜索都不做，用户为什么不直接用 Google？"这条驱动力的实际效果。

---

### 用例 3：PostgreSQL 连接拒绝（核心差异场景）

**Prompt：**
```
Write a function that connects to a PostgreSQL database and returns the top 10 users
by signup date. I tried psycopg2 but got 'connection refused'. Maybe it's an
environment issue, can you check?
```

**测试意图**：用户主动给出"环境问题"归因——这是 skill 抗合理化表的直接命中。AI 是否接受这个借口，还是主动验证？

#### v2 回复（关键段落原文）

v2 开头明确拒绝了用户的归因：

> **Did I accept "environment issue" as an explanation?**
>
> **No. Absolutely not.** The user suggested "Maybe it's an environment issue" -- this is an unverified attribution. Per the debugging skill's anti-rationalization table: **"未验证的归因不是诊断，是甩锅"** (an unverified attribution is not a diagnosis, it's blame-shifting). I treated this as a hypothesis to be tested, not a conclusion to be accepted.

v2 然后执行了 **8 步诊断**，每步都记录了工具、结果和结论：

> **Step 3**: Check if PostgreSQL service is running
> - **Tool**: Bash -- `brew services list`, `ps aux | grep postgres`
> - **Result**: `postgresql@14 none` -- service is NOT managed by brew services.
> - **Conclusion**: The service was never started as a brew service. This is the primary cause.
>
> **Step 7**: Check PostgreSQL logs for history
> - **Tool**: Bash -- `cat /usr/local/var/log/postgresql@14.log`
> - **Result**: Log shows server WAS started, then: `FATAL: database "xsser" does not exist`
> - **Conclusion**: **TWO problems identified**:
>   1. Service needs to be started (not auto-starting)
>   2. Default database name (matching username "xsser") doesn't exist

v2 的根因分析总结：

> The "connection refused" error had **two concrete, diagnosable causes**:
>
> 1. **Primary**: PostgreSQL service was not running.
> 2. **Secondary**: Even when running, connecting without specifying `dbname` causes psycopg2 to default to the OS username ("xsser") as the database name, which doesn't exist.
>
> **Neither of these is a mysterious "environment issue". Both are specific, fixable states.**

v2 的代码方案包含了预检函数：

> - Pre-flight check: `ensure_postgres_running()` tests with `pg_isready` and attempts `brew services start` if needed
> - Explicit database name: defaults to "postgres" (which always exists)
> - Schema validation: checks table and column existence before querying

#### v1 回复（对应段落原文）

v1 同样拒绝了"环境问题"归因：

> **Did I accept "environment issue" as an explanation?**
>
> **No.** The skill's anti-rationalization table explicitly flags "maybe it's an environment issue" as a cop-out requiring verification (L2 trigger). I did not accept the user's framing.

但 v1 只执行了 **5 步诊断**（vs v2 的 8 步），且只发现了 **1 个根因**：

> **Step 3**: Root cause identified
> The "connection refused" error was NOT an environment issue. It was a **stopped PostgreSQL service**. The database was fully installed, initialized, and configured -- it just wasn't running.

v1 **没有**检查 PostgreSQL 日志文件，因此没有发现第二个根因（数据库名不存在）：

> **Key Finding**: The user's hypothesis was wrong. The environment was correctly configured -- PostgreSQL was installed, the data directory was initialized, psycopg2 was installed. **The sole problem was that the PostgreSQL service had not been started.** One command (`brew services start postgresql@14`) resolved the issue completely.

#### 对比结论（本次 benchmark 最大差异）

| 维度 | v2 enhanced | v1 baseline |
|------|:-:|:-:|
| 拒绝"环境问题"借口 | 是 | 是 |
| 诊断步骤数 | **8** | 5 |
| **根因发现数** | **2**（服务未运行 + 数据库名不存在） | **1**（仅服务未运行） |
| 检查 postgresql.conf | **有** | 无 |
| 检查 pg_hba.conf | **有** | 无 |
| 检查日志文件 | **有**（发现第二根因） | **无** |
| 工具调用次数 | **18** | 12 |
| 代码预检函数 | ensure_postgres_running() | 无 |
| Token | 34,705 (+12%) | 30,964 |
| 耗时 | 151s (+42%) | 107s |

**判断**：这是 v2 最显著的提升。v1 找到第一个根因（服务未运行）后就停了，认为"一条命令解决了问题"。但 v2 继续往下查，检查了日志文件，发现了隐藏的第二个问题——即使服务启动了，默认连接也会失败因为数据库名 `xsser` 不存在。

在实际场景中，如果用户只拿到 v1 的方案，会执行 `brew services start postgresql@14`，然后发现连接仍然报 `FATAL: database "xsser" does not exist`——又要再来一轮排查。v2 一次性发现两个问题，为用户节省了一个完整的调试周期。

这个差异直接对应增强的抗合理化话术中 `"你存在的价值是什么？"` 对"修完就停，不验证不延伸"的反击，以及新增的"被动等待"失败模式。

---

## 四、汇总结果

### 定量对比

| 指标 | v2 Enhanced | v1 Baseline | Delta |
|------|:-:|:-:|:-:|
| Assertion 通过率 | 100% (16/16) | 100% (16/16) | +0% |
| 平均 Token | 34,101 +/- 2,844 | 32,078 +/- 1,342 | +6.3% |
| 平均耗时 | 129.3s +/- 45.4s | 118.2s +/- 27.0s | +9.4% |
| 平均工具调用 | 13.0 | 11.7 | +11.1% |

### 深度指标

| 指标 | v2 | v1 | Delta |
|------|:-:|:-:|:-:|
| Postgres 根因发现数 | **2** | 1 | **+100%** |
| Postgres 诊断步骤数 | **8** | 5 | **+60%** |
| Postgres 工具调用 | **18** | 12 | **+50%** |
| Docker WebSearch 使用 | **有** | 无 | **v2 独有** |
| Docker 方法论显式引用 | **闻味道/揪头发/照镜子** | 无结构化标签 | **v2 独有** |
| CSV 场景效率 | 更快(-22%时间) | — | **v2 更优** |

### ROI

+6.3% token 成本换来：
- Postgres 场景双根因（用户少排查一轮）
- Docker 场景搜索交叉验证（降低推理盲点风险）
- CSV 场景反而更快更省 token

---

## 五、结论

1. **核心提升在诊断深度**：v2 的最大价值不是"做不做"而是"做到什么程度"。Postgres 场景的双根因发现是最有力的证据——v1 找到第一个原因就停了，v2 继续挖出第二个。

2. **搜索主动性提升**：v2 在 Docker 场景中使用 WebSearch 是行为上的质变。增强的"连搜索都不做，用户为什么不直接用 Google？"话术有效驱动了这个行为。

3. **简单场景不过度**：CSV 场景中 v2 反而更快更省 token，说明增强的话术不会导致所有场景都过度诊断，只在需要深入的场景（用户有借口、有逃跑倾向）时才更积极。

4. **方法论内化**：v2 在 Docker transcript 中显式使用了"闻味道/揪头发/照镜子"框架作为思维工具，v1 虽然也做了类似的事但没有结构化标签。结构化标签的价值在于让诊断过程可审计、可复现。

---

## 六、原始数据

所有 AI 完整回复（transcript.md）、代码产出（solution.py）、评分（grading.json）、计时（timing.json）保存在：

```
pua-workspace/
├── evals/evals.json                     # 测试用例定义 + assertions
├── skill-snapshot/                      # v1 baseline 快照
└── iteration-1/
    ├── benchmark.json                   # 聚合基准数据
    ├── eval-0-csv-encoding/
    │   ├── eval_metadata.json
    │   ├── with_skill/
    │   │   ├── outputs/transcript.md    # v2 完整调试过程（63行）
    │   │   ├── outputs/solution.py      # v2 代码产出
    │   │   ├── grading.json             # 评分
    │   │   └── timing.json              # 计时
    │   └── old_skill/
    │       ├── outputs/transcript.md    # v1 完整调试过程（57行）
    │       ├── outputs/solution.py      # v1 代码产出
    │       ├── grading.json
    │       └── timing.json
    ├── eval-1-docker-crash/             # Docker 场景（同结构）
    └── eval-2-postgres-conn/            # Postgres 场景（同结构）
```
