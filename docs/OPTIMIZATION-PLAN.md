# tougepj 优化方案：修正 -102 前提 + 对齐工程实践

> 依据：对 `E:\Newproject\ppxxforever\fuck-educoder`（educoder-reminder）的实现分析，
> 以及 2026-09-27 在本机真实账号（`p8sxailt2`）下的接口实测。
> 本文只给方案，不含代码改动。

---

## 0. 结论先行

**项目的核心架构假设是错的。**

`docs/ARCHITECTURE.md` 第 0.1 节、`DEVELOPMENT_STATUS.md` 第 2 条、
`BUG.md` BUG-001、以及 `extractor/lib/educoder-api.js` 的整个「页面自身请求优先、
data API 仅作回退」体系，全部建立在这条判断上：

> 本工具直连 `data.educoder.net/api/` 的任何请求都会返回 `-102`，
> 与系统时间、cookie、`zzud`、Referer 均无关。

实测结论：`-102` **不是设备时间问题，是请求缺少签名头**。
补齐 `X-EDU-Timestamp` / `X-EDU-Signature` / `X-EDU-Type: pc` 之后，
**所有** data 接口都能直接调用——而且是在**浏览器上下文之外**的普通 Node 请求里。

这条修正能让引擎甩掉「导航页面 → 等页面自己发请求 → 截获响应」这一整套机制，
是数量级的复杂度下降，不是微调。

---

## 1. 实测证据

### 1.1 网络日志自证：同一路径，页面成功 2442 次，工具失败 2 次

对 `.educoder-network.jsonl`（14870 行）按路径聚合：

| 路径 | 页面自身请求成功 | 工具自身请求 `-102` |
|---|---|---|
| `/api/courses/DEMO0001/homework_commons.json` | **2442** | 2 |
| `/api/tasks/pool.json` | 2417 | 0 |
| `/api/users/p8sxailt2/courses.json` | 51 | **148** |
| `/api/myshixuns/zq4bc7d276/challenges.json` | 0 | 1 |

同一个 origin、同一个会话 cookie、同样的 `zzud=p8sxailt2`、同一台机器同一个浏览器，
页面发的成功、工具发的被拒。**唯一变量是请求头。**

### 1.2 全接口对照实测（同一脚本，只切换是否带签名头）

| 接口 | 无签名 | 带签名 |
|---|---|---|
| `GET /api/users/get_user_info.json` | ✓ 成功 | ✓ 成功 |
| `GET /api/users/{user}/courses.json` | ✗ -102 | ✓ `{count, courses}` |
| `GET /api/courses/{code}/homework_commons.json` | ✗ -102 | ✓ 实验列表 |
| `GET /api/shixuns/{id}/challenges.json` | ✗ -102 | ✓ 关卡列表 |
| `GET /api/myshixuns/{id}/challenges.json` | ✗ -102 | ✓ 数组(3) |
| `GET /api/tasks/{id}.json` | ✗ -102 | ✓ 含 `prev_game`/`next_game` |
| `GET /api/tasks/{id}/rep_content.json` | ✗ -102 | ✓ 不再被拒 |

执行环境：`require('playwright').request.newContext()`，**不是** `browserContext`，
**不是** `page.evaluate`。因此 `ARCHITECTURE.md` 的硬约束
「所有 Educoder 请求必须在浏览器上下文内发出」同样不成立。

### 1.3 签名算法（来自官网公开前端）

来源 `https://www-cdn.educoder.net/umi.<hash>.js` 的 `setHeader` / `getKey`：

```text
AK（双重 base64 解码）= <ak-removed>
SK（双重 base64 解码）= <sk-removed>

message   = "method=GET&ak=<AK>&sk=<SK>&time=<当前毫秒时间戳>"
signature = md5( base64(message) )        // 32 位十六进制小写
```

请求头：

```text
X-EDU-Timestamp: <同一个毫秒时间戳>
X-EDU-Signature: <上面算出的 md5>
```

AK/SK 是**公开客户端常量**，不是用户凭据，可以硬编码；但网站改版会失效，
需要保留一个能在失败时明确报「签名算法可能已变更」的探针。

### 1.4 最小必需头集合（逐个剔除实测）

| 组合 | 结果 |
|---|---|
| 三件套 + Cookie + `X-EDU-Type: pc` | ✓ |
| 去掉 `X-Request-Id` | ✓（**非必需**） |
| `X-EDU-Type` 改成 `web` | ✗（**必须精确为 `pc`**） |
| 三件套 + Cookie（无 `X-EDU-Type`） | ✗ |
| 三件套 + Cookie + Referer/Origin（无 `X-EDU-Type`） | ✗ |
| 仅三件套（无 Cookie） | ✗ |
| 签名正确但 `method` 写成 POST | ✗（**签名必须与实际 HTTP 方法一致**） |

结论：必需的是 **`X-EDU-Timestamp` + `X-EDU-Signature` + `Cookie` + `X-EDU-Type: pc`**。
`X-Request-Id`、`Referer`、`Origin`、`X-Original-*` 都不是必需的，现有代码里那一堆
伪装头可以全部删掉。

`X-EDU-Type: pc` 这一条尤其值得注意：它解释了为什么当初带着 cookie、Referer、Origin、
`zzud` 反复试都失败——缺的从来不是那几个头。

### 1.5 可复现的验证脚本

建议固化为 `tools/probe-signature.js`（下面的代码已验证可直接跑通）：

```js
'use strict';
const { request } = require('playwright');
const crypto = require('crypto');

// 双重 base64 解码后的公开客户端常量（不是用户凭据）
const AK = Buffer.from(Buffer.from('<双 base64 AK，不随仓库发布>', 'base64').toString(), 'base64').toString();
const SK = Buffer.from(Buffer.from('<双 base64 SK，不随仓库发布>', 'base64').toString(), 'base64').toString();

function signingHeaders(method = 'GET') {
  const time = Date.now();
  const message = `method=${method}&ak=${AK}&sk=${SK}&time=${time}`;
  const signature = crypto.createHash('md5')
    .update(Buffer.from(message, 'ascii').toString('base64'))   // 先 base64
    .digest('hex');                                             // 再 md5
  return {
    'X-EDU-Timestamp': String(time),
    'X-EDU-Signature': signature,
    'X-Request-Id': crypto.randomUUID()   // 实测非必需，保留便于排查
  };
}

// 必需头：上述签名头 + Cookie + X-EDU-Type:pc
// 用法：
//   const ctx = await request.newContext({ timeout: 30000 });
//   await ctx.get(url, { headers: { ...signingHeaders(), Cookie, 'X-EDU-Type': 'pc' }, max_redirects: 0 });
```


---

## 2. 两个项目的差异对照

| 维度 | educoder-reminder（fuck-educoder） | tougepj |
|---|---|---|
| 目标 | 采集作业**元数据**，按时推送提醒 | 采集题目**正文/选项/初始代码**，供 AI 作答 + 篡改猴提交 |
| 技术栈 | Python + uv + Playwright(async) | Node 22 + Playwright，CommonJS |
| **接口访问** | **签名头直连**（`request.newContext`） | **页面自身请求优先 + XHR 截获**（因误判 -102） |
| 会话 | `.env` 存 Token/Cookie，账号密码可自动恢复 | 持久化浏览器 profile，手动登录 |
| 数据层 | SQLite，事务写入，`present` 标记 + `deadline_changes` 变更表 | `levels.jsonl` 追加存档，last-wins |
| 视图 | JSON 快照（辅助核对） | Markdown + metadata.json（可重建） |
| 推送 | Server 酱（Turbo / SC3） | 无（本地 Web 控制台 + SSE） |
| 错误语义 | 三类异常分层，权限失败不当作会话失效 | 单一 `Error` + message 字符串匹配 |
| 并发保护 | `fcntl.flock` 进程锁 | 靠 `--user-data-dir` 互斥，报了才提示 |
| 测试 | unittest，注入假 fetch | `node:test`，118 用例，假 Playwright |
| 交互 | 终端 CLI + systemd | 终端 + 本地 Web 控制台 |
| 规模 | 约 1500 行 Python | 约 17500 行 JS/HTML |

**最本质的差异：两者在跟同一套 Educoder 接口打交道，但 tougepj 因为一个未解开的
协议细节，付出了数量级的复杂度代价。** 而 tougepj 在数据层设计、离线测试、
交互体验上其实反超了 educoder-reminder——这部分要守住，别在重构里丢掉。

---

## 3. 优化清单

### 3.1 删除类：前提不成立，整套机制可以拿掉

| 位置 | 现状 | 处置 |
|---|---|---|
| `extract-engine.js` `watchResponses`（222） | 常驻监听并缓存页面自身响应 | **删** |
| `cachedResponse` / `cachedResponseEntry`（250/276） | 按路径片段命中响应缓存 | **删** |
| `captureOwnResponse`（313） | 导航后等待匹配响应，超时 15s | **删** |
| `attachNetworkCapture`（196） | 仅用于排查的响应日志 | 移入 `EDUCODER_NETWORK_LOG` 开关内 |
| `coursesFromPage`（564） | 回溯 React Fiber `memorizedProps.dataSource` | **删**（BUG-002 自己都标了「前端改版即失效」） |
| `loadCategoryExperiments`（762） | 导航到分类页截获响应 | **删** |
| `openExperiment`（1203） | 点击卡片/「开始学习」、等 popup、截获 `shixun_exec.json`、URL/DOM 兜底找标识 | **删** |
| `openFirstLevelFromShixunPage`（1329） | 点「查看实战」进关卡拿标识 | **删** |
| `enterFirstLevel`（1383） | 「查看实战」入口再回溯链首 | **删** |
| `startPath`（1284） | 从页面 `a[href*="/tasks/"]` 收集入口 | **删** |
| `levelsFromCache`（941） | 从响应缓存拼关卡列表 | **删** |
| `shixunCategoriesFromBanner`（740） | 从页面 DOM 读分类 | **删** |
| `educoder-api.js` `probeLogin`（100） | 两种请求方式各试一遍 + 五种判据 | **简化**为单次签名调用 |
| `educoder-api.js` `collectLoginEvidence`（245） | cookie + DOM 文本正则判断登录 | **删** |
| `educoder-api.js` `apiGet`（59） | `page.evaluate` + `credentials:'include'` | **改为**签名直连，页面版留作兜底 |

配套的文档与注释也要改，否则下一个人会照着错前提继续写代码：

- `docs/ARCHITECTURE.md` 第 0.1 节整节重写；
- 第 0 节硬约束「所有请求必须在浏览器上下文内发出」删掉；
- `DEVELOPMENT_STATUS.md` 第 2 条「该环境下 data API 被整体拒绝」改写为已解决；
- `BUG.md` BUG-001 补一条「根因已定位：缺签名头，非设备时间」；
- `educoder-api.js` 文件头注释第 9-11 行、`extract-engine.js` 218 行附近注释。

**预计瘦身**：`extract-engine.js` 2254 行 → 约 850~950 行；`educoder-api.js` 440 行 → 约 220 行。

### 3.2 简化类：逻辑保留，实现换掉

| 位置 | 现状 | 优化后 |
|---|---|---|
| `loadCourses`（527） | API → 失败则页面 Fiber | 直接 API，去掉 `apiBlocked` 分支 |
| `loadExperiments`（607） | 导航到课堂实验页 + 等网络空闲 + 截获 | 一次 `homework_commons.json` |
| `loadLevels`（974） | 导航 → 轮询合并两份缓存 | 一次 `challenges.json`；**关卡顺序直接取 `position`**，不再需要靠链式遍历定序 |
| `resolveEntry`（1089） | 三重候选 + 回溯 | 简化为「取 `position === 1` 的关卡」，`prev_game` 回溯降级为**只在无 position 时**的兜底 |
| `findChainStart`（1053） | 主路径 | 降级为兜底路径 |
| `fetchTask`（1409） | 缓存 → 导航截获 → API 回退 | 一次签名 API |
| `captureLevel`（1739） | 导航到关卡页抓取 | 直接取 `tasks/{id}.json`，删掉导航与 `sleep` |
| `ensureLogin`（362） | 接口 → cookie → DOM → 「页面能读到课程」四重 | 一次 `get_user_info.json`（该接口**无签名也能通**，是最干净的存活探针） |
| `diagnose`（457） | 检查 cookie/原文/DOM/时间偏差 | 改为检查：签名是否有效、会话是否有效、接口连通性 |
| `attachStarterCode`（1440） | `page.request.get` | 签名请求（`rep_content.json` 已验证） |
| `attachImages`（1477） | 页面内 `fetch` 转 base64 | 签名请求直接取二进制 |
| `launchContextWithRetry`（163） | 启动重试 2 次 | **保留**——登录仍需要浏览器 |

### 3.3 借鉴类：从 educoder-reminder 搬过来的工程实践

按落地价值排序：

1. **去掉硬编码账号**。`educoder-api.js` 第 16 行 `const AUTH_USER = 'p8sxailt2'` 写死了账号，
   连同 `taskUrl()`（1305）里的 `zzud=p8sxailt2`、若干处同样的字面量。
   改为用 `get_user_info.json` 的 `login` 字段动态取得——这个接口**不需要签名**就能调通，
   正好当作用户身份的唯一来源。这一步直接把工具从「只对一个人可用」变成通用。

2. **错误语义分层**。现在是 `throw new Error('读取课程实验失败：...')`，
   调用方靠 `/-102|时间不匹配/.test(error.message)` 反推。改为三个类：
   - `AuthExpiredError`（会话失效 → 提示重新登录）
   - `IdentityMismatchError`（登录账号与预期不符）
   - `ApiError`（签名失效、权限不足、HTTP 异常、结构异常）
   关键是**权限失败不得触发重新登录**，否则会白烧登录次数。

3. **分页完整性校验**。`listCourses`（306）现在只有「累计 ≥ count 就停 + 20 页上限」。
   补齐三道闸：**重复 ID 检测**、**分页期间 `count` 变化检测**、**累计超过 `count` 报错**。
   宁可整批失败重来，也不要静默存半份——这是 BUG-012 / BUG-019 那类问题的通用解。

4. **原子写**。`level-store.js` 的 `appendLevel` 用 `fs.appendFile` 直接写，
   断电会留半截行（`readArchive` 已经在容忍这件事了）。可改为
   「写临时文件 + `fs.rename`」；或保持追加式，但在每条记录前加长度前缀便于截断恢复。
   同时给 `exports/`、`levels.jsonl` 加权限收紧（当前 gitignore 有了，权限没有）。

5. **单实例进程锁**。现在靠 `--user-data-dir` 冲突在启动时才暴露（BUG-017）。
   改为在启动时获取一个锁文件（Windows 用 `fs.open` + `wx` 标志，或 `proper-lockfile` 的
   纯手写版），**在 UI 上提前提示**「已有实例在运行」，而不是让用户看到 Chromium 的 exitCode=21。

6. **请求节流**。批量拉关卡时每请求间隔 150~200ms，避免把接口打出限流。

7. **配置集中且不被插值**。tougepj 现在大量用环境变量（`EDUCODER_*`），
   建议收进一个 `.env`（Node 22 原生支持 `--env-file`），并且**不要**做 shell 插值——
   密码/密钥里的 `$` 会被吃掉。

8. **登录自动恢复**（可选，价值中等）。educoder-reminder 支持账号密码表单重登 + 限流
   （30 分钟 1 次、24 小时 3 次）。tougepj 现在完全靠手动登录，跨机器/长期运行会掉。
   若要做，务必带上同样的限流，并且**不在日志里输出任何凭据**。

### 3.4 保留类：tougepj 优于 educoder-reminder 的部分，重构时不要动

- **存档 / 视图分离**：`levels.jsonl` 只追加 + 视图可重建 + `contentHash` 增量合并。
  比 educoder-reminder 的「SQLite + JSON 双写」在可恢复性上更清晰，**保持**。
- **题型白名单**（`classifyTask`）与**正文小节白名单**：5020 条真实样本归纳出来的判据，**保持**。
- **离线测试体系**：118 个用例 + 假 Playwright + `cli-ui-flow.test.js` 的 HTTP/SSE 端到端。**保持**，
  并且这次重构必须让它们继续全绿。
- **本地 Web 控制台 + SSE + 单文件前端**：零构建、零新增依赖（`ARCHITECTURE.md` 的硬约束）。**保持**。
- **网络日志默认关闭**：`.educoder-network.jsonl` 20MB 的大头已经被治住了。**保持**。
- **API Key 只存内存、不落盘、不回传前端**。**保持**。

---

## 4. 分阶段实施

原则：**每一阶段结束时测试必须全绿，且随时可回滚**。页面路径不要一次删光，
先用开关并存，验证充分后再拆。

### 阶段 0：固化证据（0.5 天，零风险）

0. **先拿到全绿基线**。在普通终端（非沙箱）跑一次 `npm test`，
   确认 119 个用例的真实通过数。若不绿，先定位那两条子进程用例是真缺陷还是环境问题
   （见 §5「测试基线并非全绿」），**并顺手修掉测试耗时异常**——阶段 2 要反复跑测试，
   5 分钟一轮会严重拖慢迭代。
1. 新增 `extractor/lib/educoder-sign.js`——纯函数，无 IO，无网络：
   ```js
   module.exports = { signingHeaders, decodeKey, SIGNING_VERSION };
   ```
2. 新增 `tools/probe-signature.js`——把 §1.5 的验证脚本固化，
   一条命令输出「签名是否有效 / 会话是否有效 / 各接口连通性」。
3. 新增 `test/educoder-sign.test.js`——锁定签名算法的测试向量（用固定 timestamp 断言固定 md5）。
4. 修正 `ARCHITECTURE.md` 第 0.1 节、`DEVELOPMENT_STATUS.md` 第 2 条、`BUG.md` BUG-001 的结论。

> 这一阶段不改任何抓取逻辑，纯新增 + 文档修正。

### 阶段 1：数据访问层切换（1 天）

1. `educoder-api.js` 的 `apiGet` 改为签名直连；原实现改名 `apiGetInPage` 保留。
2. 新增开关 `EDUCODER_TRANSPORT=direct|page`，默认 `direct`，`page` 走旧路径。
3. `checkLogin` 收敛为单次 `get_user_info.json`。
4. `AUTH_USER` 改为运行时从身份接口取得（见 §3.3 第 1 条）。
5. 跑 `npm test` + `tools/probe-real-flow.js`，对比新旧路径的课程/实验/关卡数量是否一致。

### 阶段 2：引擎瘦身（2 天）

1. 删除 §3.1 全部条目。
2. `loadLevels` / `resolveEntry` 改为 `position` 驱动（§3.2）。
3. `captureLevel` 去掉导航与 `sleep`，改为直接取 `tasks.json`。
4. 保留 `findChainStart` 作为「`challenges.json` 没有 `position` 时」的兜底，并记 warn。
5. 更新 `test/extract-engine.test.js`——**假 Playwright 的桩要从「模拟页面响应」改为「模拟 HTTP」**，
   这一步是阶段 2 的主要工作量。

### 阶段 3：工程化对齐（1 天）

1. 错误分层（§3.3 第 2 条）。
2. 分页三闸（§3.3 第 3 条）。
3. 原子写 + 权限（§3.3 第 4 条）。
4. 进程锁 + UI 提前提示（§3.3 第 5 条）。

### 验收标准

| 项 | 标准 |
|---|---|
| 功能等价 | 同一账号下：课程 10 门、示例课程B 5 个实验，与页面一致 |
| 关卡完整 | CNN 1→5、面向对象分析与设计 1→11，序号连续无缺号 |
| 产物不变 | 重跑后 `levels.jsonl` 与 `实验.md` 与优化前逐字节一致（除 `capturedAt`） |
| 性能 | 抓完 CNN 5 关的**页面交互等待**应大幅下降（现状：每关导航 + 网络空闲 + `sleep(1500)`×N） |
| 测试 | `npm test` 全绿，且 `test/ui-visual-check.js`、`test/ui-stop-gate-check.js` 通过 |
| 回归 | 在**未登录**、**会话过期**、**签名失效**三种情况下，报错信息可操作 |

### 失败即回滚

`EDUCODER_TRANSPORT=page` 一个环境变量就能退回旧路径——
所以阶段 1 完成前，不要删除任何页面路径代码。

---

## 5. 风险、边界，以及同期应处理的观察

| 风险 | 说明 | 缓解 |
|---|---|---|
| 签名算法变更 | AK/SK 来自公开前端 bundle，网站改版即失效 | `tools/probe-signature.js` 在启动时探测；失效时报「签名算法可能已变更」而不是笼统的「时间不匹配」 |
| 服务器时钟 | 签名含毫秒时间戳，依赖本机时钟 | 探测脚本同时报告本机与服务器时间偏差 |
| 浏览器依赖 | 登录仍需 Playwright profile | 数据路径可脱离浏览器，但**登录**保留浏览器上下文。这是有意为之 |
| `X-EDU-Type: pc` 语义 | 实测 `web` 会失败，说明该字段参与服务端分流 | 写进常量并在测试里锁定，避免被"顺手改掉" |
| 阶段 2 的测试改造 | 假 Playwright 桩要重写，工作量被低估的风险 | 单独排 1 天；桩改造完成前不动引擎 |
| `OPEN-005` 序号依赖 | 改为 `position` 驱动后，「`position` 与 `homework_challenge_index` 一致」这个前提变得更关键 | 保留链式遍历兜底 + 不一致时记 warn |

**另外三条与本次优化无关、但建议同期处理的观察（2026-09-27 本机实测）**：

### 测试基线并非全绿

`node --test --test-concurrency=1 "test/**/*.test.js"` 实测结果：

```text
tests 119 | pass 117 | fail 2 | duration 308.5s
```

两条失败：

| 用例 | 位置 | 报错 |
|---|---|---|
| `tools/validate-answers.js：集中目录里的答案能通过…` | `test/answer-validator.test.js:144` | `null !== 1`（期望退出码 1，实到 null） |
| `build-exports：从存档重建视图，且幂等` | `test/level-store.test.js:101` | `spawnSync … node.exe EBUSY` |

两条**根因相同**：都用 `execFileSync(process.execPath, …)` 派生子进程跑 `tools/*.js`，
而**本机当前执行环境下 `execFileSync(process.execPath, ['-v'])` 必然 `EBUSY`**
（受管 node 与系统 node 都复现，与 node 版本无关）。
单独直接执行 `node tools/build-exports.js` 是正常的。

> 判断：**这很可能是执行沙箱禁止派生子进程导致的，不是项目缺陷**。
> 但请在普通终端里复核一次——`DEVELOPMENT_STATUS.md` 声称「94 个离线用例」、
> `BUG.md` 声称「118/118」，与当前 119/117 对不上，说明基线的真实状态没有被跟踪。

**所以本方案的第一步不是改代码，而是先在普通终端拿到一次真实的、全绿的基线**
（`pass === tests`）。基线不绿就开始重构，后面无法判断是谁弄坏的。

### 测试耗时异常

119 个离线用例跑 **308 秒**，而单条用例普遍是 0.1~300ms 级。离线用例不该跑 5 分钟，
建议排查是否有用例在等真实超时（最可能是有假页面桩没有 resolve 某个
`waitForResponse`，于是每次都要等满 15s）。这会直接影响阶段 2 的迭代速度。

### OPEN-004 的重复实现

`writer-server.js`、`traditional-crawler.js`、3 个 userscript、2 个浏览器控制台片段
与主流程重复。本次重构会让重复度进一步升高，建议在阶段 2 之后集中清理或明确标记 deprecated。

---

## 6. 附：本次新增/改动文件清单

**新增**

```text
extractor/lib/educoder-sign.js      # 签名（纯函数）
tools/probe-signature.js            # 签名与会话探针
test/educoder-sign.test.js          # 签名测试向量
docs/OPTIMIZATION-PLAN.md           # 本文
```

**主要改动**

```text
extractor/lib/educoder-api.js       # 传输层换签名直连；账号改为动态取得；错误分层；分页三闸
extractor/lib/extract-engine.js     # 删除响应截获体系；关卡顺序改 position 驱动
extractor/lib/level-store.js        # 原子写 + 权限
extractor/run.js                    # 进程锁；传输开关
extractor/ui/server.js / index.html # 锁冲突提示；诊断面板改为签名/会话探测
docs/ARCHITECTURE.md                # 第 0.1 节重写；删除「必须在浏览器上下文」硬约束
DEVELOPMENT_STATUS.md / BUG.md      # 结论修正
test/extract-engine.test.js 等      # 假 Playwright 桩 → 假 HTTP 桩
```
