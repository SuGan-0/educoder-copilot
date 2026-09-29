# 设计：只提取答案模式 + 教学课程一键答题（含缺答案清单）

- 日期：2026-09-28
- 分支：`feat/webui-light`
- 状态：设计已与用户逐节确认（控制台、油猴两块均「按这个做」）

## 1. 背景与目标

工具现状：控制台（`npm run ui`）抓**课堂**（classrooms）的课程 → 实验 → 关卡，导出题目存档与答案；
篡改猴在关卡页按本地答案自动作答提交。本次要加两个能力：

1. **控制台「只提取答案」**：跑同一条抓取链路，但只导出答案文件，不写题目存档（题干 / 图片 / md / levels.jsonl）。
2. **篡改猴支持教学课程（`/paths/{id}`）**：面板里能选「课程 → 课程实验 → 关卡」并「一键答题」——从教学课程开始，按本地已有答案逐关自动作答。

配套（用户已拍板）：作答时**缺答案**的关卡——先判是不是虚拟机题，是就跳过；不是就把「课程 + 课程实验 + 关卡」上报到控制台，统一存成本地清单；之后可用这份清单做**定向一键答题**。

## 2. 范围

**做**

- 控制台：抓取选项加「只提取答案」；新增缺答案清单（文件 + 接口 + 面板）。
- 篡改猴：教学课程页匹配；三级选择面板；队列式一键答题；缺答案上报与「按清单再答」。

**不做（明确划界）**

- 控制台**不新增**「教学课程」来源（用户拍板「只做课堂」）。教学课程的答案来源＝课堂里同关卡标识的复用 / 油猴作答通过后自动回传 / 缺答案清单。
- 不做「整门课全自动」：用户选的是方案一（三级选择 + 队列）。
- 不改现有「一键作答并提交」的单关与实验内连续作答行为，包括「没答案时用编辑器当前内容兜底」。
- 不猜、不直连提交接口：提交仍走页面按钮点击（既有原则，见 `HANDOVER.md` §6.2）。

## 3. 术语

- **课堂**：`/classrooms/...` 下的课程（现有链路）。
- **教学课程**：`/paths/{id}` 形态的课程；结构 = 课时（stage）→ 课程实验（`item_type === 'Shixun'`）→ 关卡链。
- **关卡链**：同一实验内关卡用 `next_game` / `prev_game` 串起来；关卡标识只在实验内唯一，跨实验聚合用复合键 `课程__实验__L001`。
- **缺答案清单**：控制台 `exports/missing-answers.json`，记录「有题目但本地没有答案、且不是虚拟机题」的关卡。

## 4. 控制台设计

### 4.1 只提取答案模式

**入口（UI）**

- `extractor/ui/public/index.html`：`#panel-progress` 的 `.prog-ft` 里、`#opt-redownload` 之后加勾选 `#opt-answers-only`，文案「只提取答案（不写题目存档）」。
- `btn-start`（开始提取）与 `btn-continue`（继续提取）的 `runAction('start', …)` 载荷加 `answersOnly: 勾选值`。课程 / 实验 / 关卡的选择交互完全不变。

**引擎行为**（`extractor/lib/extract-engine.js`）

- start 处理里读 `payload.answersOnly` → `this.answersOnly`（每轮 start 重新赋值）。
- `captureLevel()` 在 `attachStarterCode` 之后、复用快路径之前插入分支：

  ```js
  if (this.answersOnly) {
    // 判据（是否通过）与内容（代码/选项）都在这份 payload 里，不写任何存档。
    await this.captureAnswer(task, payload, course, experiment);
    return payload.next_game || null;
  }
  ```

- 该分支下**跳过**：`findReusableLevel`（不写存档，复用比对无意义）、`attachImages`、`saveTask`（levels.jsonl）。
- 整轮结束的 `flushExperimentViews()`（生成 `实验.md` / metadata）在只提取答案模式下**跳过**。
- 与「重新下载已存在的关卡」勾选的关系：只提取答案模式不走复用路径，该勾选自然失效（不写存档，也无可复用），两个勾选同时勾上时以只提取答案为准。
- 计数与事件沿用现有「已保存 / 跳过 / 失败」三键，语义换成答案口径：
  - 已保存：答案已导出（内容没变、`mergeAnswer` 幂等未重写时日志仍打「已是最新，未重写」，计已保存）；
  - 跳过：该关未通过测评 / 不是可作答题型（`captureAnswer` 的 `skipped` 原文进日志）；
  - 失败：异常。
- `done` 事件照发；完成面板在只提取答案模式下把产物区文案改为「只提取答案：未写题目存档」，不列 md / 图片产物。

### 4.2 缺答案清单

**存储**：新模块 `extractor/lib/missing-store.js` + 数据文件 `exports/missing-answers.json`（相对 `EDUCODER_EXPORT_ROOT`）：

```json
{
  "updated_at": "2026-09-28T12:00:00.000Z",
  "items": [
    {
      "id": "稳定哈希(course|experiment|identifier|url)",
      "course": "示例课程B",
      "experiment": "第1课时：…",
      "level": "第2关 留出法",
      "level_id": "L002",
      "identifier": "zq75a64d2b22",
      "url": "https://www.educoder.net/tasks/…",
      "reason": "no-answer",
      "reported_at": "2026-09-28T12:00:00.000Z"
    }
  ]
}
```

- 模块接口：`readMissing()` / `addMissing(items)`（按 `id` 幂等去重，已存在则更新时间）/ `removeMissing(ids)` / `clearMissing()`；读写风格与 `answer-store.js` 一致（坏文件不炸，返回空）。
- 条目里的 `url` 取上报时的关卡页地址，供「按清单再答」直达。

**接口**（`extractor/ui/server.js`；鉴权口径与 `/api/answer/save` 一致：只校验 token，`Access-Control-Allow-Origin: *`，因为调用方是篡改猴扩展）：

- `GET /api/missing` → `{ ok, items }`
- `POST /api/missing` → body `{ action: 'add' | 'remove' | 'clear', items?: [...], ids?: [...] }` → `{ ok, items }`

**界面**：`index.html` 右侧栏新增可折叠面板 `#panel-missing`（沿用现有面板语汇）：

- 徽标显示条数；列表每行：课程 · 实验 · 关卡（+ 理由与时间）；
- 按钮「刷新」「清空」；空态文案说明来源（「一键答题遇到没有本地答案的关卡时会记录在这里」）。

## 5. 篡改猴设计

### 5.1 匹配与面板

- `@match` 增加 `https://www.educoder.net/paths/*`；「教学课程」区在 paths 页与关卡页都可用，默认折叠。
- 面板新增区块（沿用现有 `eas-*` CSS 语汇）：
  - 课程下拉 + 「刷新」；
  - 课程实验勾选列表（名称 + 已过/总数 + 虚拟机标记）；
  - 关卡勾选列表（按实验分组，名称 + 序号；默认全选，可取消）；
  - 按钮「一键答题（选中）」「按清单再答」「停止」；队列进度行（第 i/N 关 + 当前关卡名）。

### 5.2 数据来源与签名

- 身份：`GET https://data.educoder.net/api/users/get_user_info.json?school=1`（**免签名**，取 `login`）。
- 课程列表：`GET /api/users/{login}/courses.json?category=&status=processing&page=N&per_page=100&sort_by=updated_at&sort_direction=desc&username={login}&zzud={login}`（翻页到 `count`）。
- 课程实验：`GET /api/v2/stages.json?subject_id={课程id}&…`（确切参数以实现期探针抓到的为准）→ `data.stages[].items[]` 过滤 `item_type === 'Shixun'`，取 `stage_name / name / identifier / challenges_count / finished_challenges_count / is_jupyter / is_jupyter_lab`。
- 关卡：优先用「导航到 `/shixuns/{identifier}/challenges` 后页面自己发的那份」`/api/shixuns/{id}/challenges.json`（现有网络钩子已能观察）；拿不到再签名直取。
- **签名**：上述 data 接口都需要 `X-EDU-Timestamp` / `X-EDU-Signature` / `X-EDU-Type: pc`（见 `docs/ARCHITECTURE.md` §2、`extractor/lib/educoder-sign.js`）。脚本内置等价实现：
  - `signature = md5(base64('method=GET&ak=<AK>&sk=<SK>&time=<毫秒时间戳>'))`；
  - AK/SK 用与 extractor 相同的双重 base64 常量（`atob(atob(...))` 解码）；注释注明「网站改版时 extractor 与脚本两处一起更新」；
  - 内置一份紧凑 md5 实现（浏览器无原生 md5），**必须**用 `test/educoder-sign.test.js` 的向量做行为测试；
  - 请求走页面上下文 `unsafeWindow.fetch(url, { credentials: 'include', headers })`（与引擎在页面里发请求同路，已验证可行）；探针若发现跨域被拦，退化到 `GM_xmlhttpRequest`（补 `@connect`）。

### 5.3 队列状态机

- 队列 = 选中的「实验 × 关卡序号」展开成一串要作答的关卡；执行单位 = 关卡。
- **进入实验**：导航 `/shixuns/{identifier}/challenges` → 点入口按钮（文案候选沿用引擎那套：查看实战 / 继续挑战 / 开始挑战 / 继续学习 / 开始学习…）→ 落到关卡链；落点不在第 1 关时，优先点「上一关」回退，没有该控件就用观察到的 `prev_game` 回溯定位链首（实现期探针确认，见 §7）。
- **逐关推进**：页面上用现有「下一关」机制；本实验链走完进入下一个实验。
- **队列状态持久化**（GM 存储）：`{ mode: 'course' | 'missing', items, index, running }`。每次页面跳转会重载脚本，boot 时若队列在跑且当前页是队首 → 自动续跑；对不上 → 暂停并打日志（不盲目续跑）。队首匹配口径：优先比「关卡标识」（URL 最后一段）；没有标识时比「实验标识 + 关卡序号」（序号取观察到的 `challenge.position`）。「停止」按钮 / 手动离开页面即暂停。

### 5.4 跳过规则

只针对选中的关卡；未选中的关卡只走过场（点「下一关」），不提交、不判定。

| 顺序 | 条件 | 动作 |
|---|---|---|
| 1 | 虚拟机题：实验带 `is_jupyter` / `is_jupyter_lab`（建队列时整实验剔除），或**页面就绪后**既无代码编辑器、也无 `radio/checkbox` 选项 | 跳过 + 日志（不进清单） |
| 2 | 已通过：`game.final_score >= challenge.score`（字段就在脚本已观察的 `/api/tasks/{id}.json` 里） | 跳过 + 日志 |
| 3 | 无本地答案（`state.answers` 按「URL 标识 → 复合键」查不到） | 上报缺答案清单 + 日志，继续下一关 |
| 4 | 有答案 | 选择题 → 勾选；编程题 → 写 monaco（`window.Monaco` + `updateMonacoValue`）；提交 → 等结论（现有逻辑） |

- 通过 → 现有 `pushAnswerToConsole` 落盘；若该关在清单里 → `remove`。
- 未通过 / 拿不到结论 → **默认停下**（与 `strictAdvance` 同口径），日志指明课程 / 实验 / 关卡；修好答案后可直接重跑（已通过的会自动跳）。
- 旧行为边界：只有队列模式把「无答案」当记账跳过；原「一键作答并提交」保留「用编辑器当前内容兜底」。

### 5.5 缺答案清单联动

- 上报：`POST /api/missing`，`action:'add'`，条目含 `course / experiment / level / level_id / identifier / url / reason:'no-answer'`。
- 「按清单再答」：`GET /api/missing` → 以条目建队列（直达 `url`）→ 逐条处理（跳过规则仍生效：已通过当场 `remove`；补到答案并答过后 `remove`）。
- 面板显示清单条数；清单拉取失败只影响显示，不影响答题。

## 6. 错误处理与边界

| 场景 | 处置 |
|---|---|
| 只提取答案：关卡未通过 / 答案为空 | 计「跳过」，日志按 `captureAnswer` 的 `skipped` 原文 |
| 只提取答案：`rep_content.json` 取不到 | 与现状一致（`attachStarterCode` 打 warn 继续）——编程题答案可能缺，日志可见 |
| 缺答案清单文件损坏 | `missing-store` 返回空清单，不阻断；下次写入重建 |
| 清单接口无 token / 校验失败 | 与 `/api/answer/save` 同口径直接 `deny` |
| 油猴：签名被拒（-102） | 日志明确「签名失效（网站可能改版）」并暂停队列；**不**误报为登录过期 |
| 油猴：课程 / 实验 / 关卡列表读取失败 | 面板给错误行 + 可重试；不建空队列 |
| 油猴：页面跳转后队首对不上 | 队列暂停 + 日志 |
| 油猴：入口落到链中段且无「上一关」 | 按 `prev_game` 回溯；仍失败则该实验记 warn 跳过，不停整条队列 |

## 7. 实现前真机探针（前置任务，不猜）

新增只读探针 `tools/probe-teach-course.js`（沿用 `tools/probe-*.js` 风格：cookie 兜底登录、headless、只读），确认四件事：

1. 浏览器端带签名请求 `courses.json` / `stages.json` 从页面上下文发出能否通（HTTP 状态与 `status` 字段）；
2. 教学课程页 `stages.json` 的真实参数、课程实验入口按钮文案与点击后的落点 URL；
3. 关卡页上「上一关」控件是否存在；观察到的 `/api/tasks/{id}.json` 里 `prev_game` / `next_game` / `challenge.position` / `game.final_score` 是否可用；
4. 教学课程**关卡页 URL 形态**（三段式还是单段），供清单直达使用。

探针结果钉死以上细节后才开始写实现；若与设计冲突，先修订设计再动手。

## 8. 测试计划

- **引擎（只提取答案）**：行为测试（临时 export root + 假页面）：只写 `exports/answers/*.answers.json`；不写 `levels.jsonl`、不生成 `实验.md`、不下载图片；计数与日志口径正确；未通过关卡计「跳过」。
- **服务端 / 清单**：`missing-store` 单测（幂等 add、remove、clear、坏文件）；接口测试（token 鉴权、CORS 头、add / remove / clear 往返）。
- **篡改猴**：`vm` + 假 DOM 行为测试（拿真机抄来的响应样本）：课程 / 实验 / 关卡解析；队列构建（虚拟机实验剔除、默认全选）；跳过规则（已通过 / 无答案 / 虚拟机）；上报与移除载荷；boot 续跑匹配（对得上 / 对不上）；签名实现对齐 `test/educoder-sign.test.js` 向量。
- **渲染**：改 `index.html` 后跑 `node tools/probe-ui-render.js`，确保内联脚本无错。
- **真机端到端**：① 课堂跑一次「只提取答案」，核对只出 answers；② 教学课程跑「一键答题（选中）」，核对跳过 / 上报 / 回传；③ 用清单跑「按清单再答」，核对自动移除。
- 沿用项目铁律：与 DOM / 文案打交道的逻辑必须有行为测试 + 真机验证。

## 9. 约束与兼容

- 控制台**只出答案、不提交**（标题与说明不动）；提交永远在头歌页面上下文里由篡改猴做。
- 复合键 `课程__实验__L001`、`levelIds` 三级查找、答案文件格式（`docs/AI-ANSWER-FORMAT.md`）不变。
- 不引外部字体 / CDN（离线约束）；控制台保持「蓝图工程台」视觉语汇。
- 签名 AK/SK 与算法在 extractor 与篡改猴各有一份，注释互相指向；网站改版时同步更新。
- 篡改猴测试钩子（`__educoderAutosubmitTestHooks`）继续扩展；单测模式不启动面板、不装网络钩子。

## 10. 风险与未决

- **浏览器端签名请求 / CORS**：本设计唯一未在真机验证过的底层假设（§7.1 先验）。不通则退 `GM_xmlhttpRequest`；最坏退化为「只支持当前课程 + 观察页面请求」。
- **教学课程关卡链的页面控件**：入口按钮文案与「上一关」是否存在（§7.2 / 7.3），按探针结果定回溯方式。
- **跨实验跳转的重载**：队列状态存 GM 续跑；若平台在跳转时弹确认框（如未保存提示），需要额外处理（实现时验证）。