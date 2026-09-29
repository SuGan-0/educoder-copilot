# 实施计划：只提取答案 + 教学课程一键答题（含缺答案清单）

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 控制台加「只提取答案」模式与缺答案清单；篡改猴支持教学课程（`/paths/{id}`）的「课程 → 课程实验 → 关卡」三级选择与队列式一键答题；缺答案的关卡上报清单、可「按清单再答」。

**Architecture:** 控制台侧：引擎在 `captureLevel` 里加 `answersOnly` 分支（只调 `captureAnswer`，跳过存档/图片/视图），新增 `missing-store` 模块 + `/api/missing` 路由 + 面板。篡改猴侧：内嵌签名实现（与 `extractor/lib/educoder-sign.js` 同源）直连教学课程接口做三级发现；队列状态存 GM 存储、跨页面跳转续跑；作答复用现有「填答案 → 提交 → 等结论」函数。

**Tech Stack:** Node 22（`node:test` + 内置 `fetch`）、Playwright（既有假页面夹具）、Tampermonkey（`GM_*` + `unsafeWindow`）、原生 JS（无构建步骤）。

**依据的设计文档:** `docs/superpowers/specs/2026-09-28-answers-only-and-teach-course-queue-design.md`（已提交 `054578a`）

**执行顺序:** Phase A（控制台，可独立交付）→ Phase B（真机探针，钉死 Phase C 的常量）→ Phase C（篡改猴）。

## 文件结构（先锁定边界）

| 文件 | 动作 | 职责 |
|---|---|---|
| `extractor/lib/missing-store.js` | 新建 | 缺答案清单的读写/合并（唯一碰 `exports/missing-answers.json` 的地方） |
| `extractor/lib/extract-engine.js` | 修改 | `captureLevel` 增加 answersOnly 分支；`flushExperimentViews` 加护栏；构造器接受 `answersOnly` |
| `extractor/run.js` | 修改 | start 动作把 `payload.answersOnly` 传给引擎 |
| `extractor/ui/server.js` | 修改 | 新增 `GET/POST /api/missing`（token 口径与 `/api/answer/save` 一致） |
| `extractor/ui/public/index.html` | 修改 | 「只提取答案」勾选 + 载荷接线；缺答案清单面板（HTML + JS 接线） |
| `test/missing-store.test.js` | 新建 | missing-store 单测（幂等去重 / 增删清 / 坏文件自愈） |
| `test/missing-api.test.js` | 新建 | `/api/missing` 端到端（真 HTTP + 假 export root + token 鉴权） |
| `test/extract-engine.test.js` | 修改 | 追加「只提取答案只写 answers」用例 |
| `test/ui-wiring.test.js` | 修改 | 静态接线断言（勾选 / 面板 / 载荷） |
| `tools/probe-teach-course.js` | 新建 | 只读真机探针（签名请求 / 课程实验入口 / 关卡链控件 / 关卡 URL 形态） |
| `userscript/educoder-autosubmit.user.js` | 修改 | `@match` 教学课程页；签名请求；三级选择面板；队列状态机；上报 / 再答 |
| `test/userscript-sign.test.js` | 新建 | 内嵌 md5 / 签名的向量测试 |
| `test/userscript-teach.test.js` | 新建 | 发现层解析、队列构建、跳过规则、上报载荷的行为测试 |
| `HANDOVER.md` | 修改 | 收尾：新增能力的入口、探针结论、坑位 |

---

# Phase A — 控制台

## Task A1: missing-store（缺答案清单的落盘）

> **已实现并评审通过**（提交 `9a303c3` + 评审修复 `437c011`）。评审修复让实现与本节示例代码有三处差异，**以提交为准**：
> ① `itemId` 用 `identifier || url`（优先关卡标识，避免同一关卡因 URL 查询串不同落成两条）；
> ② `addMissing` / `removeMissing` 容忍「单条对象 / 单个 id」入参，并在返回值里带 `skipped`（非对象条目计数，不静默）；
> ③ 测试文件共 **7 个用例**（新增「不同 URL 形态收敛为一条」「结构漂移与垃圾入参」；用例 1 追加 refreshed 断言、用例 3 追加「无改动不写盘」、用例 4 夹具自洽）。

**Files:**
- Create: `extractor/lib/missing-store.js`
- Test: `test/missing-store.test.js`

- [ ] **Step 1: 写失败测试**

新建 `test/missing-store.test.js`：

```js
'use strict';

/**
 * 缺答案清单的落盘与合并（离线，不触网）。
 *
 * 判据：同一条目重复上报不新增（幂等）、增/删/清各自生效、坏文件自愈。
 * 运行：node --test test/missing-store.test.js
 */
const test = require('node:test');
const assert = require('node:assert/strict');
const fs = require('node:fs');
const path = require('node:path');
const os = require('node:os');

const {
  missingPathFor,
  readMissingDocument,
  addMissing,
  removeMissing,
  clearMissing
} = require('../extractor/lib/missing-store');

function withTempRoot() {
  return fs.mkdtempSync(path.join(os.tmpdir(), 'touge-missing-'));
}

const ITEM_A = {
  course: '机器学习',
  experiment: '第1课时：模型评估',
  level: '第2关 留出法',
  level_id: 'L002',
  identifier: 'zq75a64d2b22',
  url: 'https://www.educoder.net/tasks/HCEAP94S/88000001/zq75a64d2b22?coursesId=HCEAP94S',
  reason: 'no-answer'
};

test('addMissing：写入文件，字段齐全且 id 稳定不重复', async () => {
  const root = withTempRoot();
  const first = await addMissing(root, [ITEM_A]);
  assert.equal(first.ok, true);
  assert.equal(first.added, 1);

  const document = JSON.parse(fs.readFileSync(missingPathFor(root), 'utf8'));
  assert.equal(document.items.length, 1);
  assert.match(document.items[0].id, /^[0-9a-f]{16}$/, 'id 应是稳定哈希');
  assert.equal(document.items[0].level, '第2关 留出法');
  assert.ok(document.items[0].reported_at, '应记录上报时间');

  const again = await addMissing(root, [ITEM_A]);
  assert.equal(again.added, 0, '同一关卡重复上报不得新增');
  assert.equal(again.items.length, 1);
});

test('addMissing：缺「课程/实验/关卡名」的条目直接丢弃', async () => {
  const root = withTempRoot();
  const result = await addMissing(root, [{ identifier: 'x', url: 'https://example.com' }]);
  assert.equal(result.added, 0);
  assert.deepEqual((await readMissingDocument(root)).items, []);
});

test('removeMissing：按 id 删除；未知 id 不报错也不写盘', async () => {
  const root = withTempRoot();
  const added = await addMissing(root, [ITEM_A]);
  const id = added.items[0].id;

  const removed = await removeMissing(root, [id]);
  assert.equal(removed.removed, 1);
  assert.deepEqual((await readMissingDocument(root)).items, []);

  const again = await removeMissing(root, ['不存在']);
  assert.equal(again.removed, 0);
});

test('clearMissing：清空全部并返回清掉的条数', async () => {
  const root = withTempRoot();
  await addMissing(root, [ITEM_A, { ...ITEM_A, level: '第3关 F1', identifier: 'other' }]);
  const cleared = await clearMissing(root);
  assert.equal(cleared.cleared, 2);
  assert.deepEqual((await readMissingDocument(root)).items, []);
});

test('坏文件自愈：非法 JSON 视为空清单，后续写入重建', async () => {
  const root = withTempRoot();
  fs.writeFileSync(missingPathFor(root), 'not json at all', 'utf8');
  const empty = await readMissingDocument(root);
  assert.deepEqual(empty.items, []);
  const added = await addMissing(root, [ITEM_A]);
  assert.equal(added.added, 1, '坏文件不该阻断写入');
  assert.equal(JSON.parse(fs.readFileSync(missingPathFor(root), 'utf8')).items.length, 1);
});
```

- [ ] **Step 2: 运行测试，确认失败**

Run: `node --test test/missing-store.test.js`
Expected: FAIL — `Cannot find module '../extractor/lib/missing-store'`

- [ ] **Step 3: 写实现**

新建 `extractor/lib/missing-store.js`：

```js
'use strict';

/**
 * 缺答案清单的落盘（篡改猴上报 → 控制台保存）。
 *
 * 用途：教学课程「一键答题」遇到「没有本地答案、且不是虚拟机题」的关卡时，
 * 篡改猴把「课程 / 实验 / 关卡 / 关卡 URL」上报到这里；补上答案后可照清单定向再答。
 *
 * 规范位置：`exports/missing-answers.json`（跨课程汇总，单文件）。
 * 规则：同一条目（课程 + 实验 + 关卡标识/URL）幂等去重；重复上报只刷新时间。
 * 读写风格与 lib/answer-store.js 一致：坏文件不炸、返回空清单，下次写入重建。
 */

const fs = require('node:fs/promises');
const path = require('node:path');
const crypto = require('node:crypto');

/** 清单文件的规范位置。 */
function missingPathFor(exportRoot) {
  return path.join(exportRoot, 'missing-answers.json');
}

/** 条目标识：课程/实验/关卡标识（或 URL）/关卡名任一不同即视为不同条目。 */
function itemId(item) {
  const raw = [item.course, item.experiment, item.identifier || '', item.url || '', item.level].join('|');
  return crypto.createHash('sha1').update(raw).digest('hex').slice(0, 16);
}

/** 只保留规范字段，避免把上报方的临时字段写进文件。 */
function normalizeItem(raw) {
  if (!raw || typeof raw !== 'object') return null;
  const item = {
    course: String(raw.course || '').trim(),
    experiment: String(raw.experiment || '').trim(),
    level: String(raw.level || '').trim(),
    level_id: String(raw.level_id || raw.levelId || '').trim(),
    identifier: String(raw.identifier || '').trim(),
    url: String(raw.url || '').trim(),
    reason: String(raw.reason || 'no-answer').trim()
  };
  // 课程/实验是「归属」、关卡名是「叫什么」：缺了就没法补答案，直接丢弃。
  if (!item.course || !item.experiment || !item.level) return null;
  return {
    ...item,
    id: itemId(item),
    reported_at: String(raw.reported_at || '').trim() || new Date().toISOString()
  };
}

/** 读清单；文件不存在 / 坏文件 / 结构不对都返回空清单（不抛）。 */
async function readMissingDocument(exportRoot) {
  try {
    const parsed = JSON.parse(await fs.readFile(missingPathFor(exportRoot), 'utf8'));
    if (!parsed || typeof parsed !== 'object' || !Array.isArray(parsed.items)) {
      return { updated_at: '', items: [] };
    }
    return { updated_at: String(parsed.updated_at || ''), items: parsed.items.filter((item) => item && item.id) };
  } catch {
    return { updated_at: '', items: [] };
  }
}

async function writeMissingDocument(exportRoot, document) {
  await fs.mkdir(exportRoot, { recursive: true });
  await fs.writeFile(missingPathFor(exportRoot), JSON.stringify(document, null, 2), 'utf8');
}

/**
 * 追加条目（幂等）。
 * @returns {Promise<{ok:boolean, added:number, refreshed:number, items:Array}>}
 */
async function addMissing(exportRoot, items) {
  const document = await readMissingDocument(exportRoot);
  const byId = new Map(document.items.map((item) => [item.id, item]));
  let added = 0;
  let refreshed = 0;
  for (const raw of Array.isArray(items) ? items : []) {
    const item = normalizeItem(raw);
    if (!item) continue;
    const existing = byId.get(item.id);
    if (existing) {
      // 重复上报：只刷新时间（同一关卡可能被多轮答题反复报到）
      if (existing.reported_at !== item.reported_at) {
        existing.reported_at = item.reported_at;
        refreshed += 1;
      }
      continue;
    }
    byId.set(item.id, item);
    added += 1;
  }
  if (!added && !refreshed) return { ok: true, added: 0, refreshed: 0, items: document.items };
  document.items = [...byId.values()].sort((a, b) => String(a.reported_at).localeCompare(String(b.reported_at)));
  document.updated_at = new Date().toISOString();
  await writeMissingDocument(exportRoot, document);
  return { ok: true, added, refreshed, items: document.items };
}

/** 按 id 删除；没有任何改动时不写盘。 */
async function removeMissing(exportRoot, ids) {
  const document = await readMissingDocument(exportRoot);
  const wanted = new Set((Array.isArray(ids) ? ids : []).map((id) => String(id || '')));
  const before = document.items.length;
  document.items = document.items.filter((item) => !wanted.has(item.id));
  const removed = before - document.items.length;
  if (removed) {
    document.updated_at = new Date().toISOString();
    await writeMissingDocument(exportRoot, document);
  }
  return { ok: true, removed, items: document.items };
}

/** 清空全部。 */
async function clearMissing(exportRoot) {
  const document = await readMissingDocument(exportRoot);
  const cleared = document.items.length;
  if (cleared) await writeMissingDocument(exportRoot, { updated_at: new Date().toISOString(), items: [] });
  return { ok: true, cleared, items: [] };
}

module.exports = { missingPathFor, itemId, readMissingDocument, addMissing, removeMissing, clearMissing };
```

- [ ] **Step 4: 运行测试，确认通过**

Run: `node --test test/missing-store.test.js`
Expected: PASS（6 个用例全过）

- [ ] **Step 5: 提交**

```bash
git add extractor/lib/missing-store.js test/missing-store.test.js
git commit -m "feat(store): 缺答案清单的落盘模块（幂等去重/增删清/坏文件自愈）"
```

---

## Task A2: `/api/missing` 接口

> **已实现并评审通过**（提交 `6f59c38` + 评审修复 `8a1380c`）。评审修复：缺 `items`/`ids` **字段**时返回 400（不误伤 store 对「单条对象/单个 id」的容忍）；测试另含 PUT 405、坏 JSON 400、>64KB 413、clear 后 GET 为空四条断言。

**Files:**
- Modify: `extractor/ui/server.js`（顶部 require + 路由块，插在 `/api/answer/save` 之前）
- Test: `test/missing-api.test.js`

- [ ] **Step 1: 写失败测试**

新建 `test/missing-api.test.js`：

```js
'use strict';

/**
 * 缺答案清单 HTTP 接口的端到端测试（真实 HTTP，假 export root）。
 *
 * 契约：GET /api/missing 取清单；POST /api/missing { action: add | remove | clear }。
 * 调用方是篡改猴扩展（Origin 不是本机），因此与 /api/answer/save 同口径：只校验 token。
 *
 * 运行：node --test test/missing-api.test.js
 */
const test = require('node:test');
const assert = require('node:assert/strict');
const fs = require('node:fs');
const path = require('node:path');
const os = require('node:os');

const ROOT = path.join(__dirname, '..');
const ITEM = {
  course: '机器学习',
  experiment: '第1课时：模型评估',
  level: '第2关 留出法',
  level_id: 'L002',
  identifier: 'zq75a64d2b22',
  url: 'https://www.educoder.net/tasks/HCEAP94S/88000001/zq75a64d2b22',
  reason: 'no-answer'
};

test('缺答案清单接口：add / 幂等 / remove / clear 往返，且校验 token', async (t) => {
  const exportRoot = fs.mkdtempSync(path.join(os.tmpdir(), 'touge-missing-api-'));
  const previous = process.env.EDUCODER_EXPORT_ROOT;
  process.env.EDUCODER_EXPORT_ROOT = exportRoot;
  const { startUiServer } = require(path.join(ROOT, 'extractor/ui/server.js'));
  const ui = await startUiServer({ open: false });
  t.after(async () => {
    await ui.close();
    if (previous === undefined) delete process.env.EDUCODER_EXPORT_ROOT;
    else process.env.EDUCODER_EXPORT_ROOT = previous;
    fs.rmSync(exportRoot, { recursive: true, force: true });
  });

  const token = new URL(ui.url).searchParams.get('token');
  const call = async (method, body) => {
    const response = await fetch(`http://127.0.0.1:${ui.port}/api/missing?token=${token}`, {
      method,
      headers: { 'Content-Type': 'application/json' },
      body: body === undefined ? undefined : JSON.stringify(body)
    });
    return { status: response.status, payload: await response.json() };
  };

  const empty = await call('GET');
  assert.equal(empty.status, 200);
  assert.deepEqual(empty.payload.items, []);

  const added = await call('POST', { action: 'add', items: [ITEM, { course: '', experiment: '', level: '' }] });
  assert.equal(added.status, 200);
  assert.equal(added.payload.added, 1);
  assert.equal(added.payload.skipped, 1, '被丢弃的条目要如实回给调用方，不能静默');

  const addedAgain = await call('POST', { action: 'add', items: [ITEM] });
  assert.equal(addedAgain.payload.added, 0, '同一关卡重复上报不得新增');
  assert.equal((await call('GET')).payload.items.length, 1);

  const id = (await call('GET')).payload.items[0].id;
  const removed = await call('POST', { action: 'remove', ids: [id] });
  assert.equal(removed.payload.removed, 1);
  assert.equal((await call('GET')).payload.items.length, 0);

  await call('POST', { action: 'add', items: [ITEM] });
  const cleared = await call('POST', { action: 'clear' });
  assert.equal(cleared.payload.cleared, 1);
  assert.equal((await call('GET')).payload.items.length, 0, 'clear 之后清单必须真的为空');

  const wrongMethod = await fetch(`http://127.0.0.1:${ui.port}/api/missing?token=${token}`, { method: 'PUT' });
  assert.equal(wrongMethod.status, 405, '非 GET/POST 必须 405');

  const badJson = await fetch(`http://127.0.0.1:${ui.port}/api/missing?token=${token}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: '{not json'
  });
  assert.equal(badJson.status, 400);

  const tooLarge = await fetch(`http://127.0.0.1:${ui.port}/api/missing?token=${token}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ action: 'add', items: [{ padding: 'x'.repeat(70 * 1024) }] })
  });
  assert.equal(tooLarge.status, 413, '超过 64KB 必须 413');

  const missingItems = await call('POST', { action: 'add' });
  assert.equal(missingItems.status, 400);
  assert.match(String(missingItems.payload.error), /items/);

  const unknown = await call('POST', { action: '???', items: [] });
  assert.equal(unknown.status, 400);
  assert.match(String(unknown.payload.error), /未知 action/);

  const denied = await fetch(`http://127.0.0.1:${ui.port}/api/missing?token=wrong`);
  assert.equal(denied.status, 403, '错 token 必须被拒（与既有接口同口径）');
});
```

- [ ] **Step 2: 运行测试，确认失败**

Run: `node --test test/missing-api.test.js`
Expected: FAIL — `empty.status` 是 404（路由还不存在）

- [ ] **Step 3: 加 require 与路由**

在 `extractor/ui/server.js` 第 24 行（`const { mergeAnswer } = require('../lib/answer-store');`）下面加一行：

```js
const { readMissingDocument, addMissing, removeMissing, clearMissing } = require('../lib/missing-store');
```

在 `if (pathname === '/api/answer/save') {` **之前**插入路由块：

```js
    if (pathname === '/api/missing') {
      // 与 /api/answers、/api/answer/save 同样的鉴权口径：只校验 token（调用方是篡改猴扩展）。
      res.setHeader('Access-Control-Allow-Origin', '*');
      if (!tokenMatches(pickToken(req, requestUrl, null))) return deny(res);
      const root = exportRootDir();
      if (method === 'GET') {
        const document = await readMissingDocument(root);
        return sendJson(res, 200, { ok: true, items: document.items, updated_at: document.updated_at });
      }
      if (method !== 'POST') return sendJson(res, 405, { ok: false, error: '方法不允许' });
      const raw = await readBody(req);
      if (raw.tooLarge) return sendJson(res, 413, { ok: false, error: '请求体超过 64KB 上限' });
      let payload = null;
      try {
        payload = JSON.parse(raw.text || '{}');
      } catch {
        payload = null;
      }
      if (!payload || typeof payload !== 'object' || Array.isArray(payload)) {
        return sendJson(res, 400, { ok: false, error: '请求体必须是 JSON 对象' });
      }
      if (payload.action === 'add') {
        if (payload.items === undefined || payload.items === null) {
          return sendJson(res, 400, { ok: false, error: 'items 不能为空' });
        }
        const result = await addMissing(root, payload.items);
        return sendJson(res, 200, { ok: true, added: result.added, skipped: result.skipped, items: result.items });
      }
      if (payload.action === 'remove') {
        if (payload.ids === undefined || payload.ids === null) {
          return sendJson(res, 400, { ok: false, error: 'ids 不能为空' });
        }
        const result = await removeMissing(root, payload.ids);
        return sendJson(res, 200, { ok: true, removed: result.removed, items: result.items });
      }
      if (payload.action === 'clear') {
        const result = await clearMissing(root);
        return sendJson(res, 200, { ok: true, cleared: result.cleared, items: [] });
      }
      return sendJson(res, 400, { ok: false, error: `未知 action：${String(payload.action)}` });
    }
```

- [ ] **Step 4: 运行测试，确认通过**

Run: `node --test test/missing-api.test.js`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add extractor/ui/server.js test/missing-api.test.js
git commit -m "feat(ui): /api/missing 接口（取清单 / add / remove / clear，token 鉴权与既有口径一致）"
```

---

## Task A3: 引擎「只提取答案」模式

**Files:**
- Modify: `extractor/lib/extract-engine.js`（构造器 ~L61、`captureLevel` ~L2188、`flushExperimentViews` ~L2028）
- Test: `test/extract-engine.test.js`（在文件末尾追加用例，复用该文件既有的假 Playwright 夹具）

- [ ] **Step 1: 写失败测试**

在 `test/extract-engine.test.js` 末尾追加：

```js
test('引擎：只提取答案模式只写 answers，不写 levels.jsonl / md / 图片', async (t) => {
  const fake = installFakePlaywright();
  // 本地夹具：conv 已通过（有代码答案），pool 未通过（必须计「跳过」而不是失败）
  const tasks = {
    conv: {
      challenge: {
        subject: 'conv',
        homework_challenge_index: 1,
        with_code_file: true,
        path: 'conv.py',
        score: 100,
        task_pass: '### 任务描述\n\n实现卷积。'
      },
      game: { final_score: 100 },
      next_game: 'pool',
      chooses: []
    },
    pool: {
      challenge: {
        subject: 'pool',
        homework_challenge_index: 2,
        with_code_file: true,
        path: 'pool.py',
        score: 100,
        task_pass: '### 任务描述\n\n实现池化。'
      },
      game: { final_score: 0 },
      next_game: null,
      chooses: []
    }
  };
  fake.state.ownResponses = [
    ['api/courses/HCEAP94S/homework_commons.json', JSON.stringify(EXPERIMENTS)],
    ['api/myshixuns/zq3a1b66ee/challenges.json', JSON.stringify([
      { identifier: 'conv', subject: 'conv', homework_challenge_index: 1, with_code_file: true, chooses: [] },
      { identifier: 'pool', subject: 'pool', homework_challenge_index: 2, with_code_file: true, chooses: [] }
    ])],
    ...Object.entries(tasks).map(([id, payload]) => [`api/tasks/${id}.json`, JSON.stringify(payload)])
  ];
  const restoreApi = patchApiForFakes();
  const { ExtractEngine } = freshEngineModule();
  const exportRoot = withTempExportRoot();
  const events = [];
  t.after(() => {
    restoreApi();
    fake.restore();
    fs.rmSync(exportRoot, { recursive: true, force: true });
  });

  const engine = new ExtractEngine({
    exportRoot,
    profileDir: path.join(exportRoot, 'profile'),
    pageSettleMs: 0,
    answersOnly: true,
    onEvent: (event) => events.push(event),
    control: {
      waitForCourse: async (courses) => courses[0],
      waitForExperiments: async (experiments) => [experiments[0]],
      waitForLevels: async (levels) => levels
    }
  });

  await engine.run();

  // ① 答案文件照写：只有已通过的 conv（代码内容来自 rep_content，与「已提交的代码」同口径）
  const answerPath = path.join(exportRoot, 'answers', '示例课程B__CNN.answers.json');
  const document = JSON.parse(fs.readFileSync(answerPath, 'utf8'));
  assert.deepEqual(document.answers.map((item) => item.level_id), ['L001']);
  assert.equal(document.answers[0].kind, 'code');
  assert.match(String(document.answers[0].answer), /print\("starter"\)/);

  // ② 题目存档一律不写：连实验目录都不该出现（levels.jsonl / 实验.md / assets 都在它下面）
  const experimentDir = path.join(exportRoot, '示例课程B', 'CNN');
  assert.equal(
    fs.existsSync(experimentDir),
    false,
    `只提取答案不得创建实验目录，实际内容：${fs.existsSync(experimentDir) ? fs.readdirSync(experimentDir).join(',') : '(不存在)'}`
  );

  // ③ 计数口径：已通过 = 已保存（答案已导出）；未通过 = 跳过
  const done = events.filter((event) => event.type === 'done').pop();
  assert.ok(done, '应发出 done');
  assert.equal(done.saved, 1, '已通过关卡应计「已保存」（答案已导出）');
  assert.equal(done.skipped, 1, '未通过关卡应计「跳过」');
});
```

- [ ] **Step 2: 运行测试，确认失败**

Run: `node --test --test-name-pattern="只提取答案" test/extract-engine.test.js`
Expected: FAIL — 引擎还不认 `answersOnly`：实验目录存在（写满了 levels.jsonl / md）

- [ ] **Step 3: 改引擎（三处）**

① 构造器：在 `this.skipExisting = options.skipExisting !== false;`（约 L61）下面加：

```js
    /** 只提取答案：不写题目存档（levels.jsonl / md / 图片），只导出 answers。 */
    this.answersOnly = options.answersOnly === true;
```

> 注意：**不要**在 `resetForNextRun()` 里重置它——`run()` 内部会先调 `resetForNextRun()`，
> 重置会把 `run.js` 刚设好的值抹掉。每轮 start 都由 `run.js` 重新赋值（见 Task A4）。

② `captureLevel()`：在 `await this.attachStarterCode(page, task, payload, experiment, gameIdentifier);` **之后**、
`const reusable = await this.findReusableLevel(task, course, experiment);` **之前**插入：

```js
    // 只提取答案：链路照跑（答案判据与内容都在这份 payload 里），但不写任何题目存档。
    // 复用快路径在这里没有意义（不写存档，也就无所谓「复用」），image/md 一律跳过。
    if (this.answersOnly) {
      const result = await this.captureAnswer(task, payload, course, experiment);
      if (result && result.ok) {
        if (result.changed === false) {
          this.stats.existing = (this.stats.existing || 0) + 1;
        } else {
          this.stats.saved += 1;
        }
        this.emit({
          // 复用既有事件类型（前端白名单/「已提取内容」面板零改动即可正确归组）：
          // changed === false 表示答案本来就是最新的（未重写盘），与正常抓取的「复用」同口径。
          type: result.changed === false ? 'reused' : 'saved',
          status: result.changed === false ? 'reused' : 'saved',
          path: path.relative(this.exportRoot, result.answerPath || ''),
          directory: path.dirname(result.answerPath || ''),
          title: task.title,
          taskType: '答案',
          level: task.level,
          levelCount: 0,
          images: 0
        });
      } else if (result && result.skipped) {
        this.stats.skipped += 1;
        this.log(`只提取答案：${levelId(task)} ${result.skipped}`);
      } else {
        this.stats.failed += 1;
        this.log(`只提取答案失败（${levelId(task)}）：${(result && result.error) || '未知原因'}`, 'warn');
      }
      this.progress({ levelName: task.level, levelType: task.type });
      return payload.next_game || null;
    }
```

③ `flushExperimentViews()`：在 `async flushExperimentViews() {` 之后第一行加护栏：

```js
    // 只提取答案模式不生成任何题目视图（实验.md / metadata）。
    if (this.answersOnly) return;
```

- [ ] **Step 4: 运行测试，确认通过**

Run: `node --test --test-name-pattern="只提取答案" test/extract-engine.test.js`
Expected: PASS

再跑一遍整个引擎测试文件确认没有回归：

Run: `node --test test/extract-engine.test.js`
Expected: PASS（原有用例全过）

- [ ] **Step 5: 提交**

```bash
git add extractor/lib/extract-engine.js test/extract-engine.test.js
git commit -m "feat(engine): 只提取答案模式——只导出 answers，不写 levels.jsonl/md/图片"
```

---

## Task A4: 「只提取答案」接线（run.js + 控制台勾选）

**Files:**
- Modify: `extractor/run.js`（start 动作 ~L290）
- Modify: `extractor/ui/public/index.html`（勾选 ~L787；两个 start 载荷 ~L2455 / ~L2462）
- Test: `test/ui-wiring.test.js`

- [ ] **Step 1: 写失败测试**

在 `test/ui-wiring.test.js` 末尾追加：

```js
test('控制台：只提取答案的勾选、载荷与引擎接线都在', () => {
  assert.match(HTML, /id="opt-answers-only"/, 'index.html 必须有「只提取答案」勾选');
  assert.match(HTML, /answersOnly: Boolean\(/, '开始/继续提取的载荷必须带上 answersOnly');
  const runSource = fs.readFileSync(path.join(ROOT, 'extractor', 'run.js'), 'utf8');
  assert.match(runSource, /engine\.answersOnly = payload\.answersOnly === true/, 'run.js 必须把 answersOnly 传给引擎');
});
```

- [ ] **Step 2: 运行测试，确认失败**

Run: `node --test test/ui-wiring.test.js`
Expected: FAIL — 找不到 `id="opt-answers-only"`

- [ ] **Step 3: 改 run.js**

在 `engine.skipExisting = payload.skipExisting !== false;`（约 L290）下面加一行：

```js
      engine.answersOnly = payload.answersOnly === true;
```

并把该分支结尾的返回改成（约 L295）：

```js
      return { ok: true, skipExisting: engine.skipExisting, answersOnly: engine.answersOnly };
```

- [ ] **Step 4: 改 index.html（勾选 + 两个载荷）**

① 在 `#opt-redownload` 那个 `<label>` 之后插入同样的一个勾选：

```html
          <label class="hint" style="display:flex;align-items:center;gap:6px;cursor:pointer">
            <input type="checkbox" id="opt-answers-only">
            <span>只提取答案（不写题目存档）</span>
          </label>
```

② 「继续提取」（`btn-continue`）与「开始提取」（`btn-start`）两个处理器里，把

```js
      runAction('start', {
        skipExisting: !($('opt-redownload') && $('opt-redownload').checked)
      }, '继续提取');
```

```js
      var redownload = $('opt-redownload');
      runAction('start', { skipExisting: !(redownload && redownload.checked) }, '开始提取');
```

分别改成：

```js
      var answersOnlyToggle = $('opt-answers-only');
      runAction('start', {
        skipExisting: !($('opt-redownload') && $('opt-redownload').checked),
        answersOnly: Boolean(answersOnlyToggle && answersOnlyToggle.checked)
      }, '继续提取');
```

```js
      var redownload = $('opt-redownload');
      var answersOnlyToggle = $('opt-answers-only');
      runAction('start', {
        skipExisting: !(redownload && redownload.checked),
        answersOnly: Boolean(answersOnlyToggle && answersOnlyToggle.checked)
      }, '开始提取');
```

- [ ] **Step 5: 运行测试 + 渲染探针，确认通过**

Run: `node --test test/ui-wiring.test.js`
Expected: PASS

Run: `node tools/probe-ui-render.js`
Expected: 无内联脚本报错（照既有输出口径，出现 error/未捕获异常即失败）

- [ ] **Step 6: 提交**

```bash
git add extractor/run.js extractor/ui/public/index.html test/ui-wiring.test.js
git commit -m "feat(ui): 抓取选项加「只提取答案（不写题目存档）」并接通引擎"
```

---

## Task A5: 缺答案清单面板

**Files:**
- Modify: `extractor/ui/public/index.html`（面板 HTML 插在 `#panel-answer` 的 `</section>` 之后；
  JS 插在 `/* ------------------------------ wiring ------------------------------ */` 之前；
  `bind()` 内接线；启动区加首次加载）
- Test: `test/ui-wiring.test.js`

- [ ] **Step 1: 写失败测试**

在 `test/ui-wiring.test.js` 末尾追加：

```js
test('控制台：缺答案清单面板与接口接线都在', () => {
  assert.match(HTML, /id="panel-missing"/, '必须有缺答案清单面板');
  assert.match(HTML, /id="badge-missing"/, '面板徽标必须存在（显示条数）');
  assert.match(HTML, /'\/api\/missing\?token='/, '必须走 /api/missing + token');
  assert.match(HTML, /function renderMissing\(/, '必须有渲染函数');
  assert.match(HTML, /action: 'clear'/, '「清空」必须调用接口的 clear');
});
```

- [ ] **Step 2: 运行测试，确认失败**

Run: `node --test test/ui-wiring.test.js`
Expected: FAIL — 找不到 `id="panel-missing"`

- [ ] **Step 3: 加面板 HTML**

在 `#panel-answer` 的 `</section>` 之后、`<section class="panel" id="panel-recent">` 之前插入：

```html
      <section class="panel" id="panel-missing">
        <div class="panel-hd">
          <span class="step">缺</span>
          <h2>缺答案清单</h2>
          <span class="spacer"></span>
          <button class="btn btn-mini" id="btn-missing-refresh" type="button">刷新</button>
          <span class="badge mono" id="badge-missing">0</span>
          <button class="btn btn-mini" id="btn-missing-clear" type="button">清空</button>
        </div>
        <div class="missing-body">
          <p class="hint">
            篡改猴在教学课程「一键答题」时，遇到<b>没有本地答案、且不是虚拟机题</b>的关卡会记到这里。
            补上答案后，在篡改猴面板点「按清单再答」即可定向过关。
          </p>
          <div class="list" id="list-missing" data-empty="true" data-empty-text="暂无缺答案记录"></div>
        </div>
      </section>
```

> 说明：内容短、列表自身滚动，所以**不做折叠**（与 AI 面板不同，不需要省版面）。

- [ ] **Step 4: 加界面逻辑与接线**

在 `/* ------------------------------ wiring ------------------------------ */` 之前插入：

```js
  /* ------------------------------ 缺答案清单 ------------------------------ */

  /** 渲染清单：行结构与「已提取内容」保持一致，复用既有样式类。 */
  function renderMissing(items) {
    var list = $('list-missing');
    var badge = $('badge-missing');
    if (!list) return;
    var rows = (items || []).map(function (item) {
      return el('div', { class: 'extracted-item' }, [
        el('span', { class: 'mono', text: item.level || '（未命名关卡）' }),
        el('span', { class: 'extracted-tag', text: item.reason || 'no-answer' }),
        el('span', { class: 'hint', text: [item.course, item.experiment].filter(Boolean).join(' · ') }),
        el('span', { class: 'hint mono', text: item.reported_at ? formatTime(item.reported_at) : '' })
      ]);
    });
    if (badge) badge.textContent = String((items || []).length);
    list.replaceChildren.apply(list, rows);
    list.dataset.empty = rows.length ? 'false' : 'true';
  }

  async function loadMissing() {
    try {
      var response = await fetch('/api/missing?token=' + encodeURIComponent(TOKEN), {
        headers: { Accept: 'application/json' }
      });
      var payload = await response.json();
      if (!payload || !payload.ok) {
        toast('读取缺答案清单失败', 'error');
        return;
      }
      renderMissing(payload.items);
    } catch (error) {
      toast('读取缺答案清单失败：' + error.message, 'error');
    }
  }

  async function clearMissingList() {
    if (!window.confirm('清空缺答案清单？（只清记录，不动任何答案文件）')) return;
    try {
      var response = await fetch('/api/missing?token=' + encodeURIComponent(TOKEN), {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ action: 'clear' })
      });
      var payload = await response.json();
      if (payload && payload.ok) {
        renderMissing([]);
        toast('缺答案清单已清空', 'ok');
      } else {
        toast('清空失败：' + ((payload && payload.error) || response.status), 'error');
      }
    } catch (error) {
      toast('清空失败：' + error.message, 'error');
    }
  }
```

在 `bind()` 里（与 `$('btn-refresh')` 等接线同一处）加：

```js
    $('btn-missing-refresh').addEventListener('click', loadMissing);
    $('btn-missing-clear').addEventListener('click', clearMissingList);
```

在启动处（找到 `connectEvents();` 调用，紧随其后）加：

```js
  loadMissing();
```

- [ ] **Step 5: 运行测试 + 渲染探针**

Run: `node --test test/ui-wiring.test.js`
Expected: PASS

Run: `node tools/probe-ui-render.js`
Expected: 无内联脚本报错

- [ ] **Step 6: 提交**

```bash
git add extractor/ui/public/index.html test/ui-wiring.test.js
git commit -m "feat(ui): 缺答案清单面板（可看/可刷新/可清空）"
```

---

# Phase B — 真机探针（先钉死再写篡改猴）

## Task B1: `tools/probe-teach-course.js`

**Files:**
- Create: `tools/probe-teach-course.js`

**目的（只读，不提交、不写 exports）：** 回答四个问题，答案决定 Phase C 的常量与分支：

1. 浏览器端**带签名**请求 `courses.json` / `stages.json` 能否通（HTTP 状态 + `status` 字段）；
2. 教学课程页 `stages.json` 的真实 URL/参数；课程实验入口按钮的**文案**与点击后的**落点 URL**；
3. 关卡页有没有「上一关」控件；`/api/tasks/{id}.json` 里 `prev_game` / `next_game` / `challenge.position` / `game.final_score` 是否可用；
4. 教学课程**关卡页 URL 形态**（三段式 `/tasks/a/b/c` 还是单段 `/tasks/{id}`）。

- [ ] **Step 1: 写探针**

新建 `tools/probe-teach-course.js`：

```js
/**
 * 诊断：教学课程（/paths/{id}）一站式摸底（**只读**：不提交、不写 exports）。
 *
 * 回答四个问题：
 *   1) 浏览器端带签名请求 courses.json / stages.json 能否通；
 *   2) 课程实验入口按钮的文案与落点 URL；
 *   3) 关卡页是否有「上一关」；task JSON 的 prev_game/next_game/position/final_score 是否可用；
 *   4) 教学课程关卡页 URL 形态（三段式还是单段）。
 *
 * 用法：node tools/probe-teach-course.js [课程页URL]
 */
const fs = require('node:fs');
const os = require('node:os');
const path = require('node:path');
const { chromium } = require('playwright');
const { signingHeaders, EDU_TYPE } = require('../extractor/lib/educoder-sign');

const PROFILE_DIR = path.join(__dirname, '..', '.educoder-browser');
const COOKIE_FILE = path.join(__dirname, '..', '..', 'tougepj', '.educoder-cookie.json');

async function openBrowser() {
  try {
    return await chromium.launchPersistentContext(PROFILE_DIR, {
      headless: true,
      channel: 'chromium',
      viewport: { width: 1440, height: 900 }
    });
  } catch {
    const saved = JSON.parse(fs.readFileSync(COOKIE_FILE, 'utf8'));
    const browser = await chromium.launch({ channel: 'chromium', headless: true });
    const context = await browser.newContext({ viewport: { width: 1440, height: 900 }, userAgent: saved.userAgent || undefined });
    await context.addCookies(
      String(saved.cookie || '')
        .split(';')
        .map((part) => part.trim())
        .filter(Boolean)
        .map((part) => {
          const index = part.indexOf('=');
          return { name: part.slice(0, index), value: part.slice(index + 1), domain: '.educoder.net', path: '/', secure: true };
        })
    );
    context.__ownedBrowser = browser;
    return context;
  }
}

/** 带签名的页面内 GET：与引擎 apiGet 同路（签名头在 Node 侧算好，进页面里发）。 */
async function signedGet(page, url) {
  const headers = { ...signingHeaders({ method: 'GET' }), 'X-EDU-Type': EDU_TYPE, Accept: 'application/json' };
  return page.evaluate(async ({ target, header }) => {
    try {
      const response = await fetch(target, { credentials: 'include', headers: header });
      const text = await response.text();
      return { status: response.status, body: text.slice(0, 3000) };
    } catch (error) {
      return { status: 0, body: String(error.message || error) };
    }
  }, { target: url, header: headers });
}

const readJson = (result) => {
  try {
    return JSON.parse(result.body);
  } catch {
    return null;
  }
};

(async () => {
  const context = await openBrowser();
  const page = context.pages()[0] || (await context.newPage());
  const observed = [];
  page.on('response', (response) => {
    const url = response.url();
    if (/\/api\//.test(url)) observed.push(url.replace('https://data.educoder.net', '').replace('https://www.educoder.net', ''));
  });

  try {
    await page.goto('https://www.educoder.net/', { waitUntil: 'domcontentloaded' }).catch(() => {});
    await page.waitForLoadState('networkidle', { timeout: 20000 }).catch(() => {});

    // ① 身份（免签名）
    const identity = await signedGet(page, 'https://data.educoder.net/api/users/get_user_info.json?school=1');
    const login = (readJson(identity) || {}).login || '';
    console.log(`身份：HTTP ${identity.status}，login=${login || '(未取到)'}`);

    // ② 课程列表（带签名）
    const coursesUrl = `https://data.educoder.net/api/users/${login}/courses.json?category=&status=processing&page=1&per_page=15&sort_by=updated_at&sort_direction=desc&username=${login}&zzud=${login}`;
    const coursesResult = await signedGet(page, coursesUrl);
    const courses = readJson(coursesResult) || {};
    console.log(`\n课程列表（带签名）：HTTP ${coursesResult.status}，status=${courses.status}，count=${courses.count}`);
    const first = Array.isArray(courses.courses) ? courses.courses[0] : null;
    if (first) console.log('  第一门课程字段：', JSON.stringify({ id: first.id, name: first.name, keys: Object.keys(first).slice(0, 14) }));

    // 无签名对照（预期 -102，验证「必须带签名」这一前提）
    const unsigned = await page.evaluate(async (target) => {
      const response = await fetch(target, { credentials: 'include' });
      const text = await response.text();
      return { status: response.status, body: text.slice(0, 200) };
    }, coursesUrl);
    console.log(`课程列表（无签名对照）：HTTP ${unsigned.status}，body=${unsigned.body.replace(/\s+/g, ' ').slice(0, 120)}`);

    // ③ 课程结构 stages.json（带签名）
    let subjectId = first && first.id;
    const target = process.argv[2] || '';
    if (target) {
      const match = /\/paths\/(\w+)/.exec(target);
      if (match) subjectId = match[1];
    }
    if (subjectId) {
      for (const origin of ['https://data.educoder.net', 'https://www.educoder.net']) {
        const stagesResult = await signedGet(page, `${origin}/api/v2/stages.json?subject_id=${subjectId}`);
        const stages = readJson(stagesResult) || {};
        const items = stages && stages.data && Array.isArray(stages.data.stages) ? stages.data.stages : [];
        const shixuns = items.flatMap((stage) => (stage.items || []).filter((item) => item.item_type === 'Shixun'));
        console.log(`\n课程结构 stages.json（${origin}）：HTTP ${stagesResult.status}，status=${stages.status}，课时=${items.length}，课程实验=${shixuns.length}`);
        if (shixuns[0]) console.log('  第一个课程实验：', JSON.stringify({ identifier: shixuns[0].identifier, name: shixuns[0].name, challenges: shixuns[0].challenges_count, jupyter: shixuns[0].is_jupyter }));
        if (shixuns.length) break;
      }
    }

    // ④ 打开课程页：看页面自己请求了哪些接口、课时展开后的入口控件
    if (target) {
      observed.length = 0;
      await page.goto(target, { waitUntil: 'domcontentloaded' }).catch(() => {});
      await page.waitForLoadState('networkidle', { timeout: 30000 }).catch(() => {});
      await page.waitForTimeout(3000);
      console.log(`\n课程页自身请求（节选）：\n  ${observed.filter((url) => /stages|course|shixun/i.test(url)).slice(0, 12).join('\n  ')}`);
      const opened = await page.evaluate(() => {
        const header = document.querySelector('.ant-collapse-header');
        if (header) header.click();
        return Boolean(header);
      });
      await page.waitForTimeout(3000);
      const entries = await page.evaluate(() => {
        const nodes = [...document.querySelectorAll('.ant-collapse-item')].slice(0, 3);
        return nodes.map((node) => ({
          text: (node.textContent || '').replace(/\s+/g, ' ').trim().slice(0, 60),
          controls: [...node.querySelectorAll('button, a[href], [class*="btn"]')]
            .map((el) => `${(el.textContent || '').replace(/\s+/g, ' ').trim().slice(0, 18)} → ${el.getAttribute('href') || el.className.slice(0, 30)}`)
            .slice(0, 8)
        }));
      });
      console.log(`课时展开：${opened ? '已点击' : '没有 .ant-collapse-header'}`);
      console.log(entries.map((item) => `- ${item.text}\n    ${item.controls.join('\n    ')}`).join('\n'));
    }

    // ⑤ 打开一个课程实验页：入口按钮 + 落点 URL + 关卡页控件
    const shixunId = process.env.PROBE_SHIXUN || '';
    if (shixunId) {
      observed.length = 0;
      await page.goto(`https://www.educoder.net/shixuns/${shixunId}/challenges`, { waitUntil: 'domcontentloaded' }).catch(() => {});
      await page.waitForLoadState('networkidle', { timeout: 20000 }).catch(() => {});
      await page.waitForTimeout(2000);
      console.log(`\n实训页请求：\n  ${observed.slice(0, 12).join('\n  ')}`);
      const buttons = await page.evaluate(() => [...document.querySelectorAll('button, a')]
        .map((el) => (el.textContent || '').replace(/\s+/g, ' ').trim())
        .filter((text) => text && text.length <= 12)
        .slice(0, 12));
      console.log('实训页按钮文案：', JSON.stringify(buttons));

      const clicked = await page.evaluate(() => {
        const wanted = /查看实战|继续挑战|开始挑战|继续学习|开始学习|进入关卡/;
        const node = [...document.querySelectorAll('button, a')].find((el) => wanted.test(el.textContent || ''));
        if (!node) return null;
        node.click();
        return (node.textContent || '').trim();
      });
      await page.waitForTimeout(4000);
      console.log(`点击入口：${clicked || '(没找到入口按钮)'} → 当前 URL：${page.url()}`);
      const taskUrl = page.url();
      const taskJson = observed.find((url) => /\/api\/tasks\/[^/]+\.json/.test(url)) || '';
      console.log(`关卡页 URL 形态：${/\d/.test(taskUrl) ? '见上（判断几段式）' : taskUrl}；观察到的 task JSON：${taskJson || '(无)'}`);

      // 直接取一份 task JSON，看关键字段
      const identifier = taskUrl.split('/').filter(Boolean).pop();
      if (taskUrl.includes('/tasks/') && identifier) {
        const payloadResult = await signedGet(page, `https://data.educoder.net/api/tasks/${identifier}.json`);
        const payload = readJson(payloadResult) || {};
        console.log('task JSON 关键字段：', JSON.stringify({
          HTTP: payloadResult.status,
          prev_game: payload.prev_game,
          next_game: payload.next_game,
          position: payload.challenge && payload.challenge.position,
          homework_challenge_index: payload.challenge && payload.challenge.homework_challenge_index,
          score: payload.challenge && payload.challenge.score,
          final_score: payload.game && payload.game.final_score
        }));
        const hasPrev = await page.evaluate(() => [...document.querySelectorAll('button, a, [class*="next"], [class*="prev"]')]
          .some((el) => /上一关|上一题/.test((el.textContent || '') + (el.getAttribute('title') || ''))));
        console.log(`页面上是否存在「上一关」：${hasPrev ? '有' : '没有'}`);
        fs.writeFileSync(path.join(os.tmpdir(), 'touge-task-sample.json'), JSON.stringify(payload).slice(0, 20000), 'utf8');
      }
    } else {
      console.log('\n（要测课程实验入口与关卡链：把 PROBE_SHIXUN=<课程实验 identifier> 再跑一次）');
    }
  } finally {
    await context.close().catch(() => {});
    if (context.__ownedBrowser) await context.__ownedBrowser.close().catch(() => {});
  }
})().catch((error) => {
  console.error('诊断失败：', error.message);
  process.exitCode = 1;
});
```

- [ ] **Step 2: 运行探针并记录结论**

Run: `node tools/probe-teach-course.js <你的教学课程页URL>`
再跑一次课程实验部分：`$env:PROBE_SHIXUN="<课程实验 identifier>"; node tools/probe-teach-course.js <同一课程页URL>`

Expected（把实际输出记进本计划下方的「探针结论」小节 + BUG.md）:
- 带签名 courses.json / stages.json `status: 0`；无签名对照是 `-102`；
- 课程实验入口按钮的真实文案；
- 关卡页 URL 形态与「上一关」有无、task JSON 四个字段是否可用。

- [ ] **Step 3: 按结论修订 Phase C（若有出入）**

若探针结论与 Phase C 的假设冲突（例如入口按钮文案不在候选列表、无「上一关」且 `prev_game` 不可用、关卡 URL 非「标识在最后一段」），**先停下来改设计/计划，再继续 Phase C**。若一致，继续。

- [ ] **Step 4: 提交**

```bash
git add tools/probe-teach-course.js
git commit -m "tools(probe): 教学课程摸底探针（签名请求/入口按钮/关卡链/URL 形态）"
```

## 探针结论（2026-09-29 真机实测，已完成）

- **签名**：带签名请求全部可用；无签名对照 `courses.json` 返回 `{"status":-102,...}`（签名必须带）。
- **课程来源**：`courses.json` 是**课堂**列表（`subject_id`/`subject_identifier` 全为 `null`），**不是**教学课程；
  教学课程入口是 `/paths/{identifier}`（identifier 为字符串，如 `zq3026ca`；首页里的推荐课也有数字型 `/paths/13007`）。
  未找到「我的教学课程列表」接口（首页 `home/index.json` 只有推荐内容）→ 面板课程来源 = **当前 /paths 页自动识别 + 本地缓存**。
- **课程信息**：`GET /api/paths/{identifier}.json?identifier=&id=`（带签名）→ `{ id: 10084, name: "OSS Development 开源软件技术" }`。
- **课程结构（stages）**：`GET /api/v2/stages.json?tab=&stage_id=&stage_item_id=&show_course_section=false&subject_id={identifier}&keywords=&zzud={login}`
  → `status:0`、课时 16、课程实验 15（页面自身发同一 URL；`zzud` 用账号 login）。
- **课程实验入口按钮文案：`继续挑战`**（同页还有「跳过，我自己来 / 预览」，不在候选模式里，不会误点）；
  点击后落点 `https://www.educoder.net/tasks/zq2e3c066d33`。
- **关卡页 URL 形态：单段式 `/tasks/{关卡标识}`**（无查询串）→ `currentLevelId()` 取最后一段即可；回退链首用「替换最后一段」。
- **「上一关」控件：没有**（该实验只有 1 关，落点即第 1 关）→ 回退链首按 `prev_game` + URL 替换实现（原后备路径转正）。
- **task JSON 字段可用**：`prev_game=null / next_game=null / position=1 / homework_challenge_index=1 / score=100 / final_score=0`；
  **多关链未实测**（该课程每个课程实验都只有 1 关）——字段与课堂同构，`next_game` 机制留到 C7 真机验证。
- **实训页自身请求**：`/api/shixuns/{zqbda7a6}/challenges.json?id={id}`（脚本直取的 URL 形状正确）。

---

# Phase C — 篡改猴

> 全部改动集中在 `userscript/educoder-autosubmit.user.js`（+ 两个新测试文件）。
>
> **探针后修订（2026-09-29，以下四条覆盖后文示例代码里的旧写法）：**
> ① 课程列表：`courses.json` 是课堂列表，**不能**用作教学课程来源。课程来源 = 当前 `/paths/{identifier}` 页自动识别 + GM 本地缓存；
>    课程信息走 `GET /api/paths/{identifier}.json?identifier=&id=`；课程实验走带 `zzud` 的 stages URL（见探针结论）。
> ② 入口按钮真机文案是「继续挑战」——已在 `TEACH_ENTRY_PATTERNS` 候选里，无需改；关卡页是单段式 `/tasks/{id}`。
> ③ 页面上没有「上一关」控件 → 回退链首用 `prev_game` + URL 替换（`urlForLevelIdentifier`），这从后备路径转为**主路径**。
> ④ 后端若返回 HTML（被登录墙拦住）等非 JSON，`signedGet` 的 `readJson` 会得到 null —— 面板要显示"读取失败"而不是空列表（沿用现有错误分支）。
>
> 动手前先把现有结构读一遍：`KEYS` / `state` / `el()` / `log()` / `consoleEndpoint()` /
> `findButton()` / `readAnswer()` / `answerChoiceLevel()` / `findCodeEditor()` /
> `waitForSubmitResult()` / `pushAnswerToConsole()` / `passedAnswerPayload()` /
> `observeTaskJson()` / `buildPanel()` / `boot()` / 测试钩子导出块。
> **现状行为不动**：原有「一键作答并提交」、连续作答、编辑器内容兜底都保持原样。

## Task C1: 内嵌签名实现（教学课程接口的前置）

**Files:**
- Modify: `userscript/educoder-autosubmit.user.js`（`@match` / 新增签名段 / 测试钩子）
- Test: `test/userscript-sign.test.js`

- [ ] **Step 1: 写失败测试**

新建 `test/userscript-sign.test.js`：

```js
'use strict';

/**
 * 篡改猴内嵌签名实现的向量测试（离线）。
 * 判据直接抄 test/educoder-sign.test.js 的硬向量：AK/SK 常量或 base64/md5 顺序一旦改错，这里必然红。
 *
 * 运行：node --test test/userscript-sign.test.js
 */
const test = require('node:test');
const assert = require('node:assert/strict');
const fs = require('node:fs');
const path = require('node:path');
const vm = require('node:vm');

const SOURCE = fs.readFileSync(path.join(__dirname, '..', 'userscript', 'educoder-autosubmit.user.js'), 'utf8');

function loadHooks() {
  const hooks = {};
  const fakeDocument = {
    querySelectorAll: () => [],
    querySelector: () => null,
    getElementById: () => null,
    createElement: () => ({ style: {}, append() {}, addEventListener() {}, setAttribute() {} }),
    head: { append() {} },
    body: { append() {} }
  };
  const location = {
    pathname: '/tasks/x',
    search: '',
    origin: 'https://www.educoder.net',
    href: 'https://www.educoder.net/tasks/x'
  };
  const sandbox = {
    window: {
      __educoderAutosubmitTestHooks: hooks,
      btoa: (value) => Buffer.from(String(value), 'binary').toString('base64'),
      atob: (value) => Buffer.from(String(value), 'base64').toString('binary'),
      document: fakeDocument,
      location
    },
    document: fakeDocument,
    location,
    navigator: { userAgent: 'test' },
    URL,
    URLSearchParams,
    console,
    setTimeout,
    clearTimeout,
    GM_getValue: (_key, fallback) => fallback,
    GM_setValue: () => {},
    GM_deleteValue: () => {},
    GM_registerMenuCommand: () => {},
    GM_xmlhttpRequest: () => {}
  };
  vm.createContext(sandbox);
  new vm.Script(SOURCE).runInContext(sandbox);
  return hooks;
}

test('篡改猴签名：与 extractor 的硬向量逐一对齐', () => {
  const hooks = loadHooks();
  assert.equal(typeof hooks.signature, 'function', '测试钩子应暴露 signature');
  assert.equal(hooks.md5('abc'), '900150983cd24fb0d6963f7d28e17f72');
  assert.equal(hooks.signature('GET', 1758956400000), '5b853aab244a2be19154aaabc0a65fdb');
  assert.equal(hooks.signature('POST', 1758956400000), 'ba80d114b7b1b5863e7bd2527cce1281');
  assert.equal(hooks.signature('GET', 0), 'b9a70328b377bb453efca5ace76f4776');
});

test('篡改猴签名：请求头三件套齐全，X-EDU-Type 必须精确是 pc', () => {
  const hooks = loadHooks();
  const headers = hooks.buildSignedHeaders('GET', 1758956400000);
  assert.equal(headers['X-EDU-Timestamp'], '1758956400000');
  assert.equal(headers['X-EDU-Signature'], '5b853aab244a2be19154aaabc0a65fdb');
  assert.equal(headers['X-EDU-Type'], 'pc');
});
```

- [ ] **Step 2: 运行测试，确认失败**

Run: `node --test test/userscript-sign.test.js`
Expected: FAIL — `hooks.signature` 不是函数

- [ ] **Step 3: 实现（改头部 + 新增签名段 + 钩子）**

① `@match` 加一行、版本号升一位：

```
// @version      0.2.0
// @match        https://www.educoder.net/tasks/*
// @match        https://www.educoder.net/shixuns/*
// @match        https://www.educoder.net/paths/*
```

② 在「工具」区（`const sleep = ...` 附近）插入签名段：

```js
  /* ---------------------------------------------------------------- 签名请求 */

  /**
   * 教学课程相关的接口（课程列表 / 课程结构 / 关卡列表）走 data.educoder.net，
   * 和引擎一样**必须带签名头**，否则会被 -102 拒绝——错误文案写的是「设备时间不匹配」，
   * 别被误导（详见 extractor/lib/educoder-sign.js 的说明）。
   *
   * 算法与常量与 extractor **同源**；网站改版时两边一起更新：
   * 抓 www-cdn.educoder.net/umi.<hash>.js，搜 setHeader / getKey，替换下面两个常量。
   */
  const ENCODED_AK = '<双 base64，不随仓库发布：放 GM 存储或本地密钥文件>';
  const ENCODED_SK = '<双 base64，不随仓库发布：放 GM 存储或本地密钥文件>';

  /** base64 编/解码走页面上下文（脚本沙箱里不保证有 btoa/atob）。 */
  function base64EncodeAscii(text) {
    const host = pageWindow();
    if (host && typeof host.btoa === 'function') return host.btoa(text);
    throw new Error('页面上下文没有 btoa，无法计算签名');
  }

  function base64Decode(value) {
    const host = pageWindow();
    if (host && typeof host.atob === 'function') return host.atob(value);
    throw new Error('页面上下文没有 atob，无法解码签名常量');
  }

  function decodeKey(encoded) {
    try {
      return base64Decode(base64Decode(encoded));
    } catch (error) {
      return '';
    }
  }

  const SIGN_AK = decodeKey(ENCODED_AK);
  const SIGN_SK = decodeKey(ENCODED_SK);

  /** 紧凑 md5（输入只按 ASCII 字节处理：签名串先 base64，天然是 ASCII）。 */
  function md5(str) {
    const S = [
      7, 12, 17, 22, 7, 12, 17, 22, 7, 12, 17, 22, 7, 12, 17, 22,
      5, 9, 14, 20, 5, 9, 14, 20, 5, 9, 14, 20, 5, 9, 14, 20,
      4, 11, 16, 23, 4, 11, 16, 23, 4, 11, 16, 23, 4, 11, 16, 23,
      6, 10, 15, 21, 6, 10, 15, 21, 6, 10, 15, 21, 6, 10, 15, 21
    ];
    // K[i] = floor(abs(sin(i+1)) * 2^32)：用定义式现算，避免抄常数表抄错。
    const K = Array.from({ length: 64 }, (_, i) => Math.floor(Math.abs(Math.sin(i + 1)) * 4294967296) | 0);
    const bytes = [];
    for (let i = 0; i < str.length; i++) bytes.push(str.charCodeAt(i) & 0xff);
    const bitLength = bytes.length * 8;
    bytes.push(0x80);
    while (bytes.length % 64 !== 56) bytes.push(0);
    for (let i = 0; i < 8; i++) bytes.push((bitLength / Math.pow(2, i * 8)) & 0xff);
    const words = [];
    for (let i = 0; i < bytes.length; i += 4) {
      words.push(bytes[i] | (bytes[i + 1] << 8) | (bytes[i + 2] << 16) | (bytes[i + 3] << 24));
    }
    let a = 0x67452301 | 0;
    let b = 0xefcdab89 | 0;
    let c = 0x98badcfe | 0;
    let d = 0x10325476 | 0;
    const leftRotate = (value, bits) => (value << bits) | (value >>> (32 - bits));
    for (let offset = 0; offset < words.length; offset += 16) {
      const start = { a, b, c, d };
      for (let i = 0; i < 64; i++) {
        let f;
        let g;
        if (i < 16) { f = (b & c) | (~b & d); g = i; }
        else if (i < 32) { f = (d & b) | (~d & c); g = (5 * i + 1) % 16; }
        else if (i < 48) { f = b ^ c ^ d; g = (3 * i + 5) % 16; }
        else { f = c ^ (b | ~d); g = (7 * i) % 16; }
        const tmp = d;
        d = c;
        c = b;
        b = (b + leftRotate((a + f + K[i] + words[offset + g]) | 0, S[i])) | 0;
        a = tmp;
      }
      a = (start.a + a) | 0;
      b = (start.b + b) | 0;
      c = (start.c + c) | 0;
      d = (start.d + d) | 0;
    }
    const hex = (value) => {
      let out = '';
      for (let i = 0; i < 4; i++) out += (((value >>> (i * 8)) & 0xff) + 0x100).toString(16).slice(1);
      return out;
    };
    return hex(a) + hex(b) + hex(c) + hex(d);
  }

  /** signature = md5(base64(`method=GET&ak=..&sk=..&time=<毫秒时间戳>`))。 */
  function signature(method, time) {
    const text = `method=${String(method || 'GET').toUpperCase()}&ak=${SIGN_AK}&sk=${SIGN_SK}&time=${time}`;
    return md5(base64EncodeAscii(text));
  }

  function buildSignedHeaders(method = 'GET', now = Date.now()) {
    const time = String(now);
    return {
      'X-EDU-Timestamp': time,
      'X-EDU-Signature': signature(method, time),
      'X-EDU-Type': 'pc'
    };
  }

  /**
   * 带签名 GET：请求走页面上下文（与引擎在页面里发请求同路，cookie 由页面自动带上）。
   * -102 时抛出人话（网站改版 → 更新常量），不误报成登录过期。
   */
  async function signedGet(url) {
    const host = pageWindow();
    if (!host || typeof host.fetch !== 'function') throw new Error('页面上下文不可用（无法发带签名请求）');
    const response = await host.fetch(url, {
      credentials: 'include',
      headers: { Accept: 'application/json', ...buildSignedHeaders('GET') }
    });
    const text = await response.text();
    let payload = null;
    try {
      payload = JSON.parse(text);
    } catch (error) {
      payload = null;
    }
    if (!response.ok) throw new Error(`请求失败（HTTP ${response.status}）：${url}`);
    if (payload && Number(payload.status) === -102) {
      throw new Error('签名被拒（-102）：网站可能改版，需更新脚本与 extractor/lib/educoder-sign.js 里的签名常量');
    }
    return payload;
  }
```

③ 测试钩子导出块（`Object.assign(window.__educoderAutosubmitTestHooks, {...})`）里追加：

```js
      md5,
      signature,
      buildSignedHeaders,
```

- [ ] **Step 4: 运行测试，确认通过**

Run: `node --test test/userscript-sign.test.js`
Expected: PASS（2 个用例）

- [ ] **Step 5: 提交**

```bash
git add userscript/educoder-autosubmit.user.js test/userscript-sign.test.js
git commit -m "feat(userscript): 内嵌签名实现（与 extractor 同源）+ 教学课程页匹配"
```

---

## Task C2: 发现层（课程 / 课程实验 / 关卡）

**Files:**
- Modify: `userscript/educoder-autosubmit.user.js`（新增发现层段 + 钩子；扩展 `observeTaskJson`）
- Test: `test/userscript-teach.test.js`

- [ ] **Step 1: 写失败测试**

新建 `test/userscript-teach.test.js`（vm + 假 DOM；响应样本取自 2026-09-28 真机实录，已裁剪）：

```js
'use strict';

/**
 * 教学课程「发现层 + 队列」的行为测试（离线）。
 * 覆盖：三种真实响应结构的解析、队列构建（默认全选/虚拟机剔除）、每关动作判定、
 *       缺答案上报载荷、队首匹配与 URL 回退、已通过判定。
 *
 * 运行：node --test test/userscript-teach.test.js
 */
const test = require('node:test');
const assert = require('node:assert/strict');
const fs = require('node:fs');
const path = require('node:path');
const vm = require('node:vm');

const SOURCE = fs.readFileSync(path.join(__dirname, '..', 'userscript', 'educoder-autosubmit.user.js'), 'utf8');

function loadHooks({ pathname = '/tasks/HCEAP94S/88000001/zq75a64d2b22', search = '' } = {}) {
  const hooks = {};
  const fakeDocument = {
    querySelectorAll: () => [],
    querySelector: () => null,
    getElementById: () => null,
    createElement: () => ({ style: {}, append() {}, addEventListener() {}, setAttribute() {} }),
    head: { append() {} },
    body: { append() {} }
  };
  const location = {
    pathname,
    search,
    origin: 'https://www.educoder.net',
    href: `https://www.educoder.net${pathname}${search}`
  };
  const sandbox = {
    window: {
      __educoderAutosubmitTestHooks: hooks,
      btoa: (value) => Buffer.from(String(value), 'binary').toString('base64'),
      atob: (value) => Buffer.from(String(value), 'base64').toString('binary'),
      document: fakeDocument,
      location
    },
    document: fakeDocument,
    location,
    navigator: { userAgent: 'test' },
    URL,
    URLSearchParams,
    console,
    setTimeout,
    clearTimeout,
    GM_getValue: (_key, fallback) => fallback,
    GM_setValue: () => {},
    GM_deleteValue: () => {},
    GM_registerMenuCommand: () => {},
    GM_xmlhttpRequest: () => {}
  };
  vm.createContext(sandbox);
  new vm.Script(SOURCE).runInContext(sandbox);
  return hooks;
}

const STAGES_SAMPLE = {
  status: 0,
  data: {
    stages: [
      {
        stage_name: '第一课时：开源软件及其开发概述（课堂 lecture）',
        items: [
          {
            item_type: 'Shixun',
            name: '开源软件及其开发概述（课堂 lecture）',
            identifier: 'zqbda7a6',
            challenges_count: 1,
            finished_challenges_count: 0,
            is_jupyter: false,
            is_jupyter_lab: false
          },
          { item_type: 'Attachment', name: 'lecture11.pdf' }
        ]
      },
      {
        stage_name: '第七课时：CI/CD（上机实习）',
        items: [
          {
            item_type: 'Shixun',
            name: 'CI/CD（上机实习）',
            identifier: 'zqbf6766',
            challenges_count: 3,
            finished_challenges_count: 1,
            is_jupyter: true,
            is_jupyter_lab: false
          }
        ]
      }
    ]
  }
};

test('发现层：课程信息与当前路径识别', () => {
  const hooks = loadHooks({ pathname: '/paths/zq3026ca' });
  assert.equal(hooks.parseCurrentPathIdentifier(), 'zq3026ca');
  assert.deepEqual(
    hooks.parsePathCourse('zq3026ca', { id: 10084, name: 'OSS Development 开源软件技术' }),
    { identifier: 'zq3026ca', id: 10084, name: 'OSS Development 开源软件技术' }
  );
});

test('发现层：课程实验摊平且只留 Shixun（含虚拟机标记与进度）', () => {
  const hooks = loadHooks();
  const rows = hooks.parseCourseExperiments(STAGES_SAMPLE);
  assert.equal(rows.length, 2);
  assert.deepEqual(rows[0], {
    stageName: '第一课时：开源软件及其开发概述（课堂 lecture）',
    name: '开源软件及其开发概述（课堂 lecture）',
    identifier: 'zqbda7a6',
    challenges: 1,
    finished: 0,
    jupyter: false
  });
  assert.equal(rows[1].jupyter, true, 'is_jupyter 必须被识别为虚拟机');
});

test('发现层：关卡列表两种结构都认，并按序号排序', () => {
  const hooks = loadHooks();
  const fromList = hooks.parseChallenges({
    challenge_list: [
      { name: '第二关', position: 2, challenge_id: 9 },
      { name: '第一关', position: 1, challenge_id: 8 }
    ]
  });
  assert.deepEqual(fromList.map((item) => item.position), [1, 2]);
  assert.equal(fromList[1].name, '第二关');
  assert.equal(hooks.parseChallenges([{ name: '唯一关', position: 1 }]).length, 1);
});

test('队列构建：默认全选、按勾选过滤、虚拟机实验剔除', () => {
  const hooks = loadHooks();
  const experiments = hooks.parseCourseExperiments(STAGES_SAMPLE);
  const levelsByExperiment = {
    zqbda7a6: [{ name: '第1关', position: 1 }, { name: '第2关', position: 2 }]
  };
  const all = hooks.buildTeachQueue(experiments, { experiments: ['zqbda7a6', 'zqbf6766'], levels: {} }, levelsByExperiment);
  assert.equal(all.total, 2, '默认 = 该实验全部关卡');
  assert.deepEqual(all.skippedJupyter, ['CI/CD（上机实习）']);
  const partial = hooks.buildTeachQueue(experiments, { experiments: ['zqbda7a6'], levels: { zqbda7a6: [2] } }, levelsByExperiment);
  assert.deepEqual(partial.experiments[0].positions, [2], '勾选了的按勾选来');
});

test('每关动作判定：未选中 → 虚拟机 → 已通过 → 无答案 → 作答', () => {
  const hooks = loadHooks();
  assert.equal(hooks.teachLevelAction({ selected: false, hasEditor: true, hasChoice: false, passed: false, hasAnswer: true }), 'unselected');
  assert.equal(hooks.teachLevelAction({ selected: true, hasEditor: false, hasChoice: false, passed: false, hasAnswer: true }), 'vm');
  assert.equal(hooks.teachLevelAction({ selected: true, hasEditor: true, hasChoice: false, passed: true, hasAnswer: false }), 'passed');
  assert.equal(hooks.teachLevelAction({ selected: true, hasEditor: true, hasChoice: false, passed: false, hasAnswer: false }), 'missing');
  assert.equal(hooks.teachLevelAction({ selected: true, hasEditor: true, hasChoice: false, passed: false, hasAnswer: true }), 'answer');
});

test('缺答案上报载荷：字段齐全、编号是 L00N', () => {
  const hooks = loadHooks();
  const item = hooks.buildMissingItem({
    course: '机器学习',
    experiment: '第1课时',
    levelName: '第2关 留出法',
    position: 2,
    identifier: 'zq75a64d2b22',
    url: 'https://www.educoder.net/tasks/HCEAP94S/88000001/zq75a64d2b22'
  });
  assert.equal(item.level_id, 'L002');
  assert.equal(item.reason, 'no-answer');
  assert.equal(item.identifier, 'zq75a64d2b22');
});

test('队首匹配：序号命中即续跑、不命中不续；handoff 超过 90 秒视为过期', () => {
  const hooks = loadHooks();
  const queue = {
    mode: 'course',
    running: true,
    handoffAt: Date.now(),
    experimentIndex: 0,
    experiments: [{ identifier: 'x', positions: [1, 2] }],
    expected: { identifier: '', position: 0 }
  };
  assert.equal(
    hooks.queueMatchesCurrent(queue, { identifier: 'zq75a64d2b22', position: 2, href: 'https://www.educoder.net/tasks/a/b/zq75a64d2b22' }),
    true
  );
  assert.equal(
    hooks.queueMatchesCurrent(queue, { identifier: 'other', position: 9, href: 'https://www.educoder.net/tasks/other' }),
    false
  );
  assert.equal(hooks.queueFresh({ ...queue, handoffAt: Date.now() - 120000 }), false);
});

test('URL 回退：把最后一段换成上一关标识，查询串保留', () => {
  const hooks = loadHooks({
    pathname: '/tasks/HCEAP94S/88000001/zq75a64d2b22',
    search: '?coursesId=HCEAP94S'
  });
  assert.equal(
    hooks.urlForLevelIdentifier('abc123'),
    'https://www.educoder.net/tasks/HCEAP94S/88000001/abc123?coursesId=HCEAP94S'
  );
});

test('已通过判定：与 extractor 的 levelPassed 同口径（满分才通过）', () => {
  const hooks = loadHooks();
  assert.equal(hooks.levelPassedOf({ challenge: { score: 100 }, game: { final_score: 100 } }), true);
  assert.equal(hooks.levelPassedOf({ challenge: { score: 100 }, game: { final_score: 0 } }), false);
  assert.equal(hooks.levelPassedOf({}), false);
});
```

- [ ] **Step 2: 运行测试，确认失败**

Run: `node --test test/userscript-teach.test.js`
Expected: FAIL — `hooks.parseCourses` 不是函数

- [ ] **Step 3: 实现（发现层 + 判定函数 + 扩展 observeTaskJson + 钩子）**

① 在签名段之后新增发现层：

```js
  /* ------------------------------------------------------------ 教学课程发现层 */

  const TEACH_ORIGIN = 'https://data.educoder.net';

  /** /api/paths/{identifier}.json → { identifier, id, name }（教学课程信息）。 */
  function parsePathCourse(identifier, payload) {
    return {
      identifier: String(identifier || ''),
      id: Number(payload && payload.id) || 0,
      name: String((payload && payload.name) || '').trim()
    };
  }

  /** 当前页面是不是教学课程页（/paths/{identifier}）。 */
  function parseCurrentPathIdentifier() {
    const match = /^\/paths\/(\w+)/.exec(String(location.pathname || ''));
    return match ? match[1] : '';
  }

  /** /api/v2/stages.json → 摊平成课程实验列表（只要 Shixun，且必须有 identifier）。 */
  function parseCourseExperiments(payload) {
    const stages = payload && payload.data && Array.isArray(payload.data.stages) ? payload.data.stages : [];
    const out = [];
    for (const stage of stages) {
      for (const item of Array.isArray(stage.items) ? stage.items : []) {
        if (String(item.item_type || '') !== 'Shixun') continue;
        if (!item.identifier) continue;
        out.push({
          stageName: String(stage.stage_name || '').trim(),
          name: String(item.name || item.shixun_name || '').trim(),
          identifier: String(item.identifier),
          challenges: Number(item.challenges_count) || 0,
          finished: Number(item.finished_challenges_count) || 0,
          jupyter: Boolean(item.is_jupyter || item.is_jupyter_lab)
        });
      }
    }
    return out;
  }

  /** 关卡列表：shixuns / myshixuns 两种端点结构都认（challenge_list 或数组）。 */
  function parseChallenges(payload) {
    const rows = Array.isArray(payload)
      ? payload
      : payload && Array.isArray(payload.challenge_list)
        ? payload.challenge_list
        : payload && Array.isArray(payload.challenges)
          ? payload.challenges
          : [];
    return rows
      .map((item, index) => ({
        name: String(item.name || item.subject || '').trim() || `第${index + 1}关`,
        position: Number(item.position ?? item.homework_challenge_index ?? 0) || index + 1,
        identifier: String(item.identifier || '')
      }))
      .sort((a, b) => a.position - b.position);
  }

  /** 教学课程没有「我的课程列表」接口（courses.json 是课堂，实测 subject_id 全为 null）：
   *  课程来源 = 当前 /paths/{identifier} 页 + 本地缓存；名字走 /api/paths/{identifier}.json。 */
  function readTeachCourseCache() {
    const cached = GM_getValue(KEYS.teachCourses, []);
    return Array.isArray(cached) ? cached : [];
  }

  function cacheTeachCourse(course) {
    if (!course || !course.identifier) return;
    const list = readTeachCourseCache().filter((item) => item.identifier !== course.identifier);
    list.unshift(course);
    GM_setValue(KEYS.teachCourses, list.slice(0, 20));
  }

  async function ensureTeachLogin() {
    if (state.teachLogin) return state.teachLogin;
    const identity = await signedGet(`${TEACH_ORIGIN}/api/users/get_user_info.json?school=1`);
    state.teachLogin = identity && identity.login ? String(identity.login) : '';
    if (!state.teachLogin) throw new Error('拿不到账号 login：确认已登录头歌');
    return state.teachLogin;
  }

  async function loadPathCourse(identifier) {
    const payload = await signedGet(
      `${TEACH_ORIGIN}/api/paths/${encodeURIComponent(identifier)}.json`
      + `?identifier=${encodeURIComponent(identifier)}&id=${encodeURIComponent(identifier)}`
    );
    return parsePathCourse(identifier, payload);
  }

  async function loadTeachExperiments(identifier) {
    const login = await ensureTeachLogin();
    // 参数照抄真机（页面自身请求的形态），少了 show_course_section/zzud 实测会 404。
    const url = `${TEACH_ORIGIN}/api/v2/stages.json`
      + `?tab=&stage_id=&stage_item_id=&show_course_section=false`
      + `&subject_id=${encodeURIComponent(identifier)}&keywords=&zzud=${encodeURIComponent(login)}`;
    const payload = await signedGet(url);
    return parseCourseExperiments(payload);
  }

  async function loadTeachChallenges(identifier) {
    const payload = await signedGet(
      `${TEACH_ORIGIN}/api/shixuns/${encodeURIComponent(identifier)}/challenges.json?id=${encodeURIComponent(identifier)}`
    );
    return parseChallenges(payload);
  }

  /* ------------------------------------------------------------ 教学课程判定（纯函数） */

  /**
   * 「已通过测评」：得分达到满分（与 extractor/lib/answer-store.js 的 levelPassed 同口径）。
   * 判据字段就在脚本已经在观察的 /api/tasks/{id}.json 里（challenge.score / game.final_score）。
   */
  function levelPassedOf(payload) {
    const max = Number(payload && payload.challenge && payload.challenge.score);
    const got = Number(payload && payload.game && payload.game.final_score);
    return Number.isFinite(max) && max > 0 && Number.isFinite(got) && got >= max;
  }

  /**
   * 队列里「当前这一关怎么处理」：顺序即优先级（未选中 → 虚拟机 → 已通过 → 无答案 → 作答）。
   * @returns {'unselected'|'vm'|'passed'|'missing'|'answer'}
   */
  function teachLevelAction({ selected, hasEditor, hasChoice, passed, hasAnswer }) {
    if (!selected) return 'unselected';
    if (!hasEditor && !hasChoice) return 'vm';
    if (passed) return 'passed';
    if (!hasAnswer) return 'missing';
    return 'answer';
  }

  /** 队列构建（纯函数）：默认 = 该实验全部关卡；虚拟机实验整体剔除。 */
  function buildTeachQueue(experiments, selection, levelsByExperiment) {
    const selected = new Set((selection && selection.experiments) || []);
    const queue = [];
    const skippedJupyter = [];
    for (const experiment of experiments) {
      if (!selected.has(experiment.identifier)) continue;
      if (experiment.jupyter) {
        skippedJupyter.push(experiment.name);
        continue;
      }
      const all = (levelsByExperiment && levelsByExperiment[experiment.identifier]) || [];
      const chosen = selection && selection.levels ? selection.levels[experiment.identifier] : null;
      const wanted = Array.isArray(chosen) && chosen.length ? chosen : all.map((level) => level.position);
      if (!wanted.length) continue;
      queue.push({
        identifier: experiment.identifier,
        name: experiment.name,
        positions: [...wanted].sort((a, b) => a - b),
        levelNames: Object.fromEntries(all.map((level) => [level.position, level.name]))
      });
    }
    return {
      experiments: queue,
      skippedJupyter,
      total: queue.reduce((sum, item) => sum + item.positions.length, 0)
    };
  }

  /** 组装一条缺答案上报。 */
  function buildMissingItem({ course, experiment, levelName, position, identifier, url }) {
    return {
      course: String(course || ''),
      experiment: String(experiment || ''),
      level: String(levelName || `第${position}关`),
      level_id: position > 0 ? `L${String(position).padStart(3, '0')}` : '',
      identifier: String(identifier || ''),
      url: String(url || ''),
      reason: 'no-answer'
    };
  }

  /** 关卡 URL 的标识永远在最后一段（见 docs/ARCHITECTURE.md「关卡 URL 两种形态」）。 */
  function urlForLevelIdentifier(identifier) {
    const parts = String(location.pathname || '').split('/').filter(Boolean);
    if (parts[0] !== 'tasks' || !identifier) return '';
    parts[parts.length - 1] = identifier;
    return `${location.origin || ''}/${parts.join('/')}${location.search || ''}`;
  }

  /** handoff 新鲜度：只有「刚跳过转」才自动续跑，防止用户随手点开关卡页就乱跑。 */
  function queueFresh(queue) {
    return Boolean(queue && queue.running) && Date.now() - Number(queue.handoffAt || 0) < 90 * 1000;
  }

  /** 队首是否就是当前页面（纯函数）。 */
  function queueMatchesCurrent(queue, current) {
    if (!queue || !queue.running) return false;
    if (queue.mode === 'missing') {
      const item = queue.items[queue.index];
      if (!item) return false;
      if (item.identifier && item.identifier === current.identifier) return true;
      return Boolean(item.url) && String(item.url).split('?')[0] === String(current.href || '').split('?')[0];
    }
    const experiment = queue.experiments[queue.experimentIndex];
    if (!experiment) return false;
    if (queue.expected && queue.expected.identifier) return queue.expected.identifier === current.identifier;
    return Number(current.position) > 0 && experiment.positions.includes(Number(current.position));
  }
```

② 扩展 `observeTaskJson` 的 `state.taskContext`（在既有的 `levelId` 字段旁补四个字段）：

```js
    state.taskContext = {
      id: match[1],
      sizes,
      levelId: position > 0 ? `L${String(position).padStart(3, '0')}` : '',
      // 教学课程队列要用：序号（按序号选关/续跑）、前后关卡标识（回退链首）、是否已通过。
      levelPosition: position,
      nextGame: String(payload.next_game || ''),
      prevGame: String(payload.prev_game || ''),
      passed: levelPassedOf(payload),
      submitted: submitted || Boolean(prev && prev.submitted),
      standardAnswer: standardAnswer.length ? standardAnswer : (prev && prev.standardAnswer) || []
    };
```

③ 测试钩子导出块追加：

```js
      parsePathCourse,
      parseCurrentPathIdentifier,
      parseCourseExperiments,
      parseChallenges,
      levelPassedOf,
      teachLevelAction,
      buildTeachQueue,
      buildMissingItem,
      urlForLevelIdentifier,
      queueFresh,
      queueMatchesCurrent,
```

- [ ] **Step 4: 运行测试，确认通过**

Run: `node --test test/userscript-teach.test.js`
Expected: PASS（9 个用例）

- [ ] **Step 5: 提交**

```bash
git add userscript/educoder-autosubmit.user.js test/userscript-teach.test.js
git commit -m "feat(userscript): 教学课程发现层与队列判定（解析/构建/动作判定/URL 回退）"
```

---

## Task C3: 教学课程面板（三级选择）

**Files:**
- Modify: `userscript/educoder-autosubmit.user.js`（`KEYS` / `state` / 面板段 / `buildPanel` / CSS / 钩子）

> **探针后修订**：① `KEYS` 增加 `teachCourses: 'educoder-teach-courses'`；
> ② 后文 `refreshTeachCourses` / `loadTeachExperimentsPanel` 两个函数用**下面的版本**（课程来源 = 当前 /paths 页识别 + 本地缓存，不再有 `loadTeachCourses`）：

```js
  /** 「刷新课程」：把当前页（若是 /paths/{id}）识别出来并入缓存，然后刷新下拉。 */
  async function refreshTeachCourses() {
    const teach = teachSelection();
    teach.busy = true;
    teach.status = '读取课程…';
    renderTeach();
    try {
      const identifier = parseCurrentPathIdentifier();
      if (identifier) {
        const course = await loadPathCourse(identifier);
        cacheTeachCourse(course);
        teach.courseId = course.id;
      }
      teach.courses = readTeachCourseCache();
      teach.status = teach.courses.length
        ? `课程 ${teach.courses.length} 门（缓存）`
        : '还没缓存课程：打开一个教学课程页（/paths/…）再点「刷新课程」';
      log(`教学课程列表：${teach.courses.length} 门（来自缓存${identifier ? '，已并入当前页课程' : ''}）`, 'ok');
    } catch (error) {
      teach.status = '课程读取失败';
      log(`课程读取失败：${error.message}`, 'error');
    }
    teach.busy = false;
    renderTeach();
  }

  /** 读课程实验：用所选课程的标识（不依赖当前页面）。 */
  async function loadTeachExperimentsPanel() {
    const teach = teachSelection();
    const course = teach.courses.find((item) => Number(item.id) === Number(teach.courseId)) || teach.courses[0];
    if (!course) {
      log('先点「刷新课程」（需要先打开一个教学课程页）', 'warn');
      return;
    }
    teach.busy = true;
    teach.status = '读取课程实验…';
    renderTeach();
    try {
      teach.experiments = await loadTeachExperiments(course.identifier);
      teach.status = `课程实验 ${teach.experiments.length} 个`;
      log(`课程实验已读取：${teach.experiments.length} 个`, 'ok');
    } catch (error) {
      teach.status = '课程实验读取失败';
      log(`课程实验读取失败：${error.message}`, 'error');
    }
    teach.busy = false;
    renderTeach();
  }
```

> 后文 Task C3「渲染与处理器」里与这两段重名的示例**以本段为准**（`renderTeach` / `toggleTeachFold` / `loadTeachLevelsPanel` / `persistTeachSelection` / `restoreTeachSelection` / `teachExperimentLabel` 仍用后文版本）。

- [ ] **Step 1: 加 KEYS 与面板状态**

`KEYS` 里加两个键：

```js
    teachSel: 'educoder-teach-selection',
    teachQueue: 'educoder-teach-queue',
```

面板段（放在「面板」段之前）加：

```js
  /* -------------------------------------------------------------- 教学课程面板 */

  const TEACH_ENTRY_PATTERNS = [/查看实战/, /继续挑战/, /开始挑战/, /继续学习/, /开始学习/, /进入关卡/];

  function teachSelection() {
    if (!state.teach) {
      state.teach = {
        open: false,
        courses: [],
        courseId: 0,
        experiments: [],
        selectedExperiments: {},
        levelsByExperiment: {},
        selectedLevels: {},
        missingCount: 0,
        busy: false,
        status: '未开始'
      };
      restoreTeachSelection();
    }
    return state.teach;
  }

  function persistTeachSelection() {
    const teach = teachSelection();
    GM_setValue(KEYS.teachSel, {
      courseId: teach.courseId,
      experiments: Object.keys(teach.selectedExperiments).filter((key) => teach.selectedExperiments[key]),
      levels: teach.selectedLevels
    });
  }

  function restoreTeachSelection() {
    const saved = GM_getValue(KEYS.teachSel, null);
    const teach = teachSelection();
    if (!saved || typeof saved !== 'object') return;
    teach.courseId = Number(saved.courseId) || 0;
    teach.selectedExperiments = Object.fromEntries((saved.experiments || []).map((key) => [String(key), true]));
    teach.selectedLevels = saved.levels && typeof saved.levels === 'object' ? saved.levels : {};
  }

  function teachExperimentLabel(experiment) {
    const parts = [`${experiment.name}（${experiment.finished}/${experiment.challenges}）`];
    if (experiment.jupyter) parts.push('虚拟机');
    return parts.join(' ');
  }
```

- [ ] **Step 2: 渲染与处理器**

```js
  function renderTeach() {
    const teach = teachSelection();
    const host = document.getElementById('eas-teach-body');
    if (!host) return;
    host.hidden = !teach.open;
    const fold = document.getElementById('eas-teach-fold');
    if (fold) fold.textContent = teach.open ? '收起 ▾' : '展开 ▸';

    const courseSelect = document.getElementById('eas-teach-course');
    if (courseSelect) {
      const current = String(teach.courseId || '');
      courseSelect.replaceChildren(
        ...(teach.courses.length
          ? teach.courses.map((course) => el('option', {
            value: String(course.id),
            textContent: course.name,
            selected: String(course.id) === current
          }))
          : [el('option', { value: '', textContent: '（先点「刷新课程」）' })])
      );
      courseSelect.onchange = () => {
        teach.courseId = Number(courseSelect.value) || 0;
        teach.experiments = [];
        teach.selectedExperiments = {};
        teach.levelsByExperiment = {};
        persistTeachSelection();
        renderTeach();
      };
    }

    const experimentsBox = document.getElementById('eas-teach-experiments');
    if (experimentsBox) {
      experimentsBox.replaceChildren(
        ...(teach.experiments.length
          ? teach.experiments.map((experiment) => {
            const input = el('input', { type: 'checkbox', checked: Boolean(teach.selectedExperiments[experiment.identifier]) });
            input.addEventListener('change', () => {
              teach.selectedExperiments[experiment.identifier] = input.checked;
              persistTeachSelection();
              renderTeach();
            });
            return el('label', { className: 'eas-teach-item' }, [
              input,
              el('span', { textContent: teachExperimentLabel(experiment) })
            ]);
          })
          : [el('div', { className: 'eas-row', textContent: teach.busy ? '读取中…' : '（选课程 → 点「读取课程实验」）' })])
      );
    }

    const levelsBox = document.getElementById('eas-teach-levels');
    if (levelsBox) {
      const rows = [];
      for (const experiment of teach.experiments) {
        if (!teach.selectedExperiments[experiment.identifier] || experiment.jupyter) continue;
        const levels = teach.levelsByExperiment[experiment.identifier] || [];
        if (!levels.length) continue;
        rows.push(el('div', { className: 'eas-teach-group', textContent: experiment.name }));
        const chosen = teach.selectedLevels[experiment.identifier];
        for (const level of levels) {
          const checked = !Array.isArray(chosen) || !chosen.length || chosen.includes(level.position);
          const input = el('input', { type: 'checkbox', checked });
          input.addEventListener('change', () => {
            const current = new Set(teach.selectedLevels[experiment.identifier] || levels.map((item) => item.position));
            if (input.checked) current.add(level.position);
            else current.delete(level.position);
            teach.selectedLevels[experiment.identifier] = [...current].sort((a, b) => a - b);
            persistTeachSelection();
          });
          rows.push(el('label', { className: 'eas-teach-item' }, [
            input,
            el('span', { textContent: `第${level.position}关 ${level.name}` })
          ]));
        }
      }
      levelsBox.replaceChildren(...(rows.length ? rows : [el('div', { className: 'eas-row', textContent: '（勾选实验后点「读取关卡」）' })]));
    }

    const status = document.getElementById('eas-teach-status');
    if (status) {
      status.textContent = `${teach.status}${teach.missingCount ? `｜缺答案 ${teach.missingCount}` : ''}`;
    }
  }

  function toggleTeachFold() {
    const teach = teachSelection();
    teach.open = !teach.open;
    renderTeach();
  }

  async function refreshTeachCourses() {
    const teach = teachSelection();
    teach.busy = true;
    teach.status = '读取课程列表…';
    renderTeach();
    try {
      const result = await loadTeachCourses();
      teach.courses = result.courses;
      if (!teach.courseId && result.courses.length) teach.courseId = result.courses[0].id;
      teach.status = `课程 ${result.courses.length} 门`;
      log(`教学课程列表已读取：${result.courses.length} 门`, 'ok');
    } catch (error) {
      teach.status = '课程列表读取失败';
      log(`课程列表读取失败：${error.message}`, 'error');
    }
    teach.busy = false;
    renderTeach();
  }

  async function loadTeachExperimentsPanel() {
    const teach = teachSelection();
    if (!teach.courseId) {
      log('先选课程（或点「刷新课程」）', 'warn');
      return;
    }
    teach.busy = true;
    teach.status = '读取课程实验…';
    renderTeach();
    try {
      teach.experiments = await loadTeachExperiments(teach.courseId);
      teach.status = `课程实验 ${teach.experiments.length} 个`;
      log(`课程实验已读取：${teach.experiments.length} 个`, 'ok');
    } catch (error) {
      teach.status = '课程实验读取失败';
      log(`课程实验读取失败：${error.message}`, 'error');
    }
    teach.busy = false;
    renderTeach();
  }

  async function loadTeachLevelsPanel() {
    const teach = teachSelection();
    teach.busy = true;
    teach.status = '读取关卡…';
    renderTeach();
    const next = { ...(teach.levelsByExperiment || {}) };
    let failed = 0;
    for (const experiment of teach.experiments) {
      if (!teach.selectedExperiments[experiment.identifier] || experiment.jupyter) continue;
      try {
        next[experiment.identifier] = await loadTeachChallenges(experiment.identifier);
      } catch (error) {
        failed += 1;
        log(`关卡列表读取失败（${experiment.name}）：${error.message}`, 'error');
      }
    }
    teach.levelsByExperiment = next;
    teach.busy = false;
    teach.status = failed ? `关卡读取有 ${failed} 个失败` : '待选关卡';
    renderTeach();
  }
```

- [ ] **Step 3: 面板 DOM 与接线**

`buildPanel()` 的按钮组之后（`el('div', { className: 'eas-opts' }, ...)` 之前）插入：

```js
        el('div', { className: 'eas-teach' }, [
          el('div', { className: 'eas-teach-hd' }, [
            el('b', { textContent: '教学课程' }),
            el('span', { style: 'flex:1' }),
            el('button', { id: 'eas-teach-fold', textContent: '展开 ▸', onclick: toggleTeachFold })
          ]),
          el('div', { id: 'eas-teach-body', hidden: true }, [
            el('div', { className: 'eas-teach-row' }, [
              el('select', { id: 'eas-teach-course' }),
              el('button', { textContent: '刷新课程', onclick: refreshTeachCourses }),
              el('button', { textContent: '读取课程实验', onclick: loadTeachExperimentsPanel })
            ]),
            el('div', { id: 'eas-teach-experiments', className: 'eas-teach-list' }),
            el('div', { className: 'eas-teach-row' }, [
              el('button', { textContent: '读取关卡', onclick: loadTeachLevelsPanel })
            ]),
            el('div', { id: 'eas-teach-levels', className: 'eas-teach-list' }),
            el('div', { className: 'eas-teach-row' }, [
              el('button', { id: 'eas-teach-run', className: 'eas-primary', textContent: '一键答题（选中）', onclick: startTeachRun }),
              el('button', { id: 'eas-teach-missing', textContent: '按清单再答', onclick: startMissingRun })
            ]),
            el('div', { id: 'eas-teach-status', className: 'eas-row', textContent: '未开始' })
          ])
        ])
```

`CSS` 末尾（`#eas-panel .eas-opt input { margin: 0; }` 之后）追加：

```css
    #eas-panel .eas-teach { border-top: 1px solid #222833; padding-top: 6px; }
    #eas-panel .eas-teach-hd { display: flex; align-items: center; gap: 6px; color: #dde4ee; }
    #eas-panel .eas-teach-row { display: flex; gap: 6px; align-items: center; margin: 4px 0; }
    #eas-panel .eas-teach-row select { flex: 1; min-width: 0; background: #11151c; color: #dde4ee; border: 1px solid #2d3644; border-radius: 4px; padding: 2px 4px; }
    #eas-panel .eas-teach-list { max-height: 150px; overflow: auto; border: 1px solid #222833; border-radius: 4px; padding: 4px; }
    #eas-panel .eas-teach-item { display: flex; align-items: center; gap: 6px; color: #a4b0c1; }
    #eas-panel .eas-teach-group { color: #6d7a8c; margin-top: 4px; }
```

`boot()` 里（`dumpStatus()` 之后）加：

```js
    restoreTeachSelection();
    if (/^\/paths\//.test(location.pathname)) teachSelection().open = true;
    renderTeach();
    refreshTeachMissingCount();
```

- [ ] **Step 4: 静态检查 + 提交**

Run: `node --test test/userscript-sign.test.js test/userscript-teach.test.js`
Expected: PASS（面板接线不破坏既有纯函数）

Run: `node --check userscript/educoder-autosubmit.user.js`
Expected: 无输出（语法合法）

```bash
git add userscript/educoder-autosubmit.user.js
git commit -m "feat(userscript): 教学课程面板（课程→课程实验→关卡三级选择）"
```

---

## Task C4: 队列状态机与续跑

**Files:**
- Modify: `userscript/educoder-autosubmit.user.js`（队列段 + `boot()`）

- [ ] **Step 1: 队列读写与匹配（纯函数，测试已覆盖匹配部分）**

```js
  /* ---------------------------------------------------------------- 教学课程队列 */

  function readTeachQueue() {
    return GM_getValue(KEYS.teachQueue, null);
  }

  function writeTeachQueue(queue) {
    GM_setValue(KEYS.teachQueue, queue);
  }

  // queueFresh / queueMatchesCurrent 已在 Task C2 定义（纯函数 + 单测），这里直接复用。

  function finishTeachQueue() {
    const queue = readTeachQueue();
    if (queue) {
      queue.running = false;
      queue.finishedAt = new Date().toISOString();
      writeTeachQueue(queue);
    }
    const teach = teachSelection();
    teach.status = '队列完成';
    renderTeach();
    log('教学课程队列完成', 'ok');
  }

  function stopTeachQueue(reason, tone = 'warn') {
    const queue = readTeachQueue();
    if (queue && queue.running) {
      queue.running = false;
      queue.stoppedReason = reason || '';
      writeTeachQueue(queue);
    }
    const teach = teachSelection();
    teach.status = reason ? `已停：${reason}` : '已停';
    renderTeach();
    log(`教学课程队列已停：${reason || '（未说明原因）'}`, tone);
  }

  function requestTeachStop() {
    stopTeachQueue('用户点了停止');
  }
```

- [ ] **Step 2: boot 续跑入口**

```js
  /** 每次页面加载后调用：队首在此就续跑，不在就暂停（不盲目乱跑）。 */
  async function maybeContinueTeachQueue() {
    const queue = readTeachQueue();
    if (!queue || !queue.running) return;
    if (!queueFresh(queue)) {
      stopTeachQueue('超过 90 秒没有新的跳转，已暂停（防止乱跑）');
      return;
    }
    if (/^\/tasks\//.test(location.pathname)) {
      await sleep(1500); // 等页面自己的关卡数据（taskContext）到位
      const current = {
        identifier: currentLevelId(),
        position: Number(state.taskContext && state.taskContext.levelPosition) || 0,
        href: location.href
      };
      if (!queueMatchesCurrent(queue, current)) {
        stopTeachQueue('当前页面不是队首，已暂停');
        return;
      }
      await processTeachLevel(queue);
      return;
    }
    if (/^\/shixuns\//.test(location.pathname) && queue.mode === 'course') {
      const experiment = queue.experiments[queue.experimentIndex];
      if (!experiment || !location.pathname.includes(`/shixuns/${experiment.identifier}/`)) return;
      await sleep(1500);
      const button = findButton(TEACH_ENTRY_PATTERNS);
      if (!button) {
        stopTeachQueue(`实验页上没找到入口按钮（${location.pathname}）`);
        return;
      }
      log(`进入课程实验「${experiment.name}」：点击「${buttonLabel(button)}」`);
      queue.handoffAt = Date.now();
      writeTeachQueue(queue);
      button.click();
    }
  }
```

`boot()` 末尾（菜单注册之后）加：

```js
    maybeContinueTeachQueue().catch((error) => log(`队列续跑失败：${error.message}`, 'error'));
```

> 注：`processTeachLevel` / `advanceTeachQueue` / `enterTeachExperiment` 在 Task C5 才定义
> （同为函数声明，作用域内可提升）。C4 提交前**不要**真跑队列，跑会在续跑时引用未定义的名字——
> C5 提交后再做真机验证。

- [ ] **Step 3: 检查 + 提交**

Run: `node --check userscript/educoder-autosubmit.user.js`
Expected: 无输出

```bash
git add userscript/educoder-autosubmit.user.js
git commit -m "feat(userscript): 教学课程队列状态机（持久化 + 90 秒内自动续跑 + 不盲目乱跑）"
```

---

## Task C5: 执行（进入实验 / 逐关 / 跳过规则 / 上报）

**Files:**
- Modify: `userscript/educoder-autosubmit.user.js`（执行段）

- [ ] **Step 1: 控制台小工具（缺答案清单的读写）**

```js
  /* -------------------------------------------------------- 缺答案清单（控制台） */

  function consoleRequest(method, pathname, body) {
    return new Promise((resolve, reject) => {
      const url = consoleEndpoint(pathname);
      if (!url) {
        reject(new Error('还没设置本地控制台地址（点「设置本地地址」）'));
        return;
      }
      GM_xmlhttpRequest({
        method,
        url,
        headers: body ? { 'Content-Type': 'application/json' } : undefined,
        data: body ? JSON.stringify(body) : undefined,
        onload: (response) => {
          let payload = null;
          try {
            payload = JSON.parse(response.responseText || '{}');
          } catch (error) {
            payload = null;
          }
          if (response.status === 403) {
            reject(new Error('控制台拒绝：地址里的 token 不对'));
            return;
          }
          if (!payload || !payload.ok) {
            reject(new Error((payload && payload.error) || `HTTP ${response.status}`));
            return;
          }
          resolve(payload);
        },
        onerror: () => reject(new Error('连不上本地控制台'))
      });
    });
  }

  async function fetchTeachMissing() {
    const payload = await consoleRequest('GET', '/api/missing');
    return Array.isArray(payload.items) ? payload.items : [];
  }

  async function postTeachMissing(body, okMessage) {
    try {
      const payload = await consoleRequest('POST', '/api/missing', body);
      if (okMessage) log(okMessage, 'ok');
      return payload;
    } catch (error) {
      log(`缺答案清单操作失败（不影响本次作答）：${error.message}`, 'warn');
      return null;
    }
  }

  /** 面板上的「缺答案 N」：拉取失败只影响显示，不影响答题。 */
  async function refreshTeachMissingCount() {
    try {
      const items = await fetchTeachMissing();
      teachSelection().missingCount = items.length;
      renderTeach();
    } catch (error) {
      /* 控制台没开就不显示，不打扰 */
    }
  }

  /** 上报一条缺答案。 */
  function reportTeachMissing(queue, levelName) {
    const position = Number(state.taskContext && state.taskContext.levelPosition) || 0;
    const experiment = queue.mode === 'course' ? queue.experiments[queue.experimentIndex] : null;
    const item = buildMissingItem({
      course: queue.courseName || '',
      experiment: experiment ? experiment.name : '',
      levelName,
      position,
      identifier: currentLevelId(),
      url: location.href
    });
    postTeachMissing({ action: 'add', items: [item] }, `已上报缺答案：${item.level}`);
  }

  /** 答完后把这一关从清单里移除（按标识/URL 找 id，再按 id 删）。 */
  async function resolveTeachMissing() {
    const identifier = currentLevelId();
    const href = location.href;
    try {
      const items = await fetchTeachMissing();
      const ids = items
        .filter((item) => item.identifier === identifier || (item.url && String(item.url).split('?')[0] === href.split('?')[0]))
        .map((item) => item.id);
      if (ids.length) await postTeachMissing({ action: 'remove', ids }, `已从缺答案清单移除 ${ids.length} 条`);
      teachSelection().missingCount = Math.max(0, items.length - ids.length);
      renderTeach();
    } catch (error) {
      /* 移除失败不影响作答 */
    }
  }
```

- [ ] **Step 2: 执行段**

```js
  /* -------------------------------------------------------------- 教学课程执行 */

  /** 进入指定课程实验：把 handoff 写好再跳转，落地由 maybeContinueTeachQueue 接手。 */
  function enterTeachExperiment(queue) {
    const experiment = queue.experiments[queue.experimentIndex];
    if (!experiment) {
      finishTeachQueue();
      return;
    }
    const teach = teachSelection();
    teach.status = `进入实验：${experiment.name}`;
    renderTeach();
    queue.expected = { identifier: '', position: 0 };
    queue.handoffAt = Date.now();
    writeTeachQueue(queue);
    location.href = `https://www.educoder.net/shixuns/${experiment.identifier}/challenges`;
  }

  /**
   * 若落点不在链首，沿 prev_game 回退（标识永远在 URL 最后一段，直接换掉最后一段即可）。
   * @returns {Promise<boolean>} true = 已在链首，可以开始处理
   */
  async function goToChainStart(queue) {
    let guard = 0;
    while (state.taskContext && state.taskContext.prevGame && guard < 30) {
      guard += 1;
      const previous = state.taskContext.prevGame;
      log(`当前不在第 1 关，沿 prev_game 回退：${previous}`);
      queue.expected = { identifier: previous, position: 0 };
      queue.handoffAt = Date.now();
      writeTeachQueue(queue);
      location.href = urlForLevelIdentifier(previous);
      return false; // 已发起跳转，等下一轮 boot 接力
    }
    return true;
  }

  /** 处理队首这一关：判定 → 作答 → 推进；任何一步失败都停下并指明是哪一关。 */
  async function processTeachLevel(queue) {
    const experiment = queue.mode === 'course' ? queue.experiments[queue.experimentIndex] : null;
    const position = Number(state.taskContext && state.taskContext.levelPosition) || 0;
    const levelName = experiment && experiment.levelNames
      ? experiment.levelNames[position] || `第${position}关`
      : (queue.mode === 'missing' && queue.items[queue.index] ? queue.items[queue.index].level : `第${position}关`);
    const label = `${experiment ? `${experiment.name} ` : ''}${levelName}`;

    if (queue.mode === 'course' && !(await goToChainStart(queue))) return;

    const selected = experiment ? experiment.positions.includes(position) : true;
    const editor = findCodeEditor();
    const hasChoice = Boolean(document.querySelector('input[type="radio"], input[type="checkbox"]'));
    const parsedAnswer = readAnswer(state.answers[answerKeyForCurrentLevel()]);
    const action = teachLevelAction({
      selected,
      hasEditor: Boolean(editor),
      hasChoice,
      passed: Boolean(state.taskContext && state.taskContext.passed),
      hasAnswer: Boolean(parsedAnswer)
    });

    queue.expected = { identifier: '', position: 0 };
    writeTeachQueue(queue);

    if (action === 'unselected') {
      log(`${label}：未选中，路过`);
      return advanceTeachQueue(queue);
    }
    if (action === 'vm') {
      log(`${label}：没有编辑器也没有选项（虚拟机题？），跳过`, 'warn');
      return advanceTeachQueue(queue);
    }
    if (action === 'passed') {
      log(`${label}：已通过，跳过`, 'ok');
      return advanceTeachQueue(queue);
    }
    if (action === 'missing') {
      log(`${label}：没有本地答案，记入缺答案清单`, 'warn');
      reportTeachMissing(queue, levelName);
      return advanceTeachQueue(queue);
    }

    // 作答（与既有 processCurrentLevel 同源，但**不做**「编辑器内容兜底」——没答案就上报）
    let answerText = '';
    if (parsedAnswer.kind === 'choice') {
      const picked = await answerChoiceLevel(parsedAnswer.map);
      if (!picked.length) {
        stopTeachQueue(`${label}：选择题勾选失败`, 'error');
        return;
      }
      log(`已勾选：${picked.join('、')}`, 'ok');
    } else {
      answerText = String(parsedAnswer.code || '');
      if (!editor) {
        stopTeachQueue(`${label}：找不到代码编辑器`, 'error');
        return;
      }
      editor.write(answerText);
      log(`已把答案写入 ${editor.kind} 编辑器（${answerText.length} 字符）`, 'ok');
    }

    const submitButton = findButton(SUBMIT_PATTERNS);
    if (!submitButton) {
      stopTeachQueue(`${label}：页面上找不到提交按钮`, 'error');
      return;
    }
    log(`点击页面按钮「${buttonLabel(submitButton)}」`);
    submitButton.click();
    const outcome = await waitForSubmitResult();
    if (outcome.status !== 'passed') {
      stopTeachQueue(
        `${label}：${outcome.status === 'failed' ? '提交未通过' : '没拿到明确结论'}——${snippet(outcome.text, 120)}。修好答案后可重跑（已通过的会自动跳过）`,
        'error'
      );
      return;
    }
    log(`${label}：提交成功`, 'ok');
    const payload = passedAnswerPayload({
      choiceMap: parsedAnswer.kind === 'choice' ? parsedAnswer.map : null,
      codeText: answerText
    });
    if (payload) pushAnswerToConsole(payload);
    await resolveTeachMissing();
    return advanceTeachQueue(queue);
  }

  /** 推进：能点「下一关」就点；本实验选中的都做完了或链到头就换下一个实验。 */
  function advanceTeachQueue(queue) {
    const experiment = queue.mode === 'course' ? queue.experiments[queue.experimentIndex] : null;
    const position = Number(state.taskContext && state.taskContext.levelPosition) || 0;

    if (experiment) {
      queue.visited = Array.isArray(queue.visited) ? queue.visited : [];
      if (position && !queue.visited.includes(position)) queue.visited.push(position);
      const done = experiment.positions.every((item) => queue.visited.includes(item));
      if (done || !state.taskContext || !state.taskContext.nextGame) {
        queue.experimentIndex += 1;
        queue.visited = [];
        return enterTeachExperiment(queue);
      }
    } else if (queue.mode === 'missing') {
      queue.index += 1;
      if (queue.index >= queue.items.length) {
        finishTeachQueue();
        return;
      }
      queue.handoffAt = Date.now();
      writeTeachQueue(queue);
      log(`按清单再答：前往下一关（${queue.index + 1}/${queue.items.length}）`);
      location.href = queue.items[queue.index].url;
      return;
    }

    const nextButton = findButton([/下一关/, /下一题/, /继续下一/]);
    if (!nextButton) {
      queue.experimentIndex += 1;
      queue.visited = [];
      return enterTeachExperiment(queue);
    }
    queue.handoffAt = Date.now();
    writeTeachQueue(queue);
    log('进入下一关');
    nextButton.click();
  }
```

- [ ] **Step 3: 检查 + 提交**

Run: `node --check userscript/educoder-autosubmit.user.js`
Expected: 无输出

Run: `node --test test/userscript-teach.test.js`
Expected: PASS（判定函数未被改坏）

```bash
git add userscript/educoder-autosubmit.user.js
git commit -m "feat(userscript): 教学课程队列执行（进入实验/逐关推进/虚拟机与已通过跳过/缺答案上报）"
```

---

## Task C6: 一键答题入口与「按清单再答」

**Files:**
- Modify: `userscript/educoder-autosubmit.user.js`（两个入口函数）

- [ ] **Step 1: 两个入口**

```js
  /* ------------------------------------------------------------ 两个队列入口 */

  /** 「一键答题（选中）」：把面板选择落成队列，然后从第一个实验开始跑。 */
  async function startTeachRun() {
    const teach = teachSelection();
    if (state.busy) {
      log('已有任务在执行（等它结束或点停止）', 'warn');
      return;
    }
    if (!teach.courseId) {
      log('先选课程（或点「刷新课程」）', 'warn');
      return;
    }
    if (!teach.experiments.length) {
      log('先点「读取课程实验」', 'warn');
      return;
    }
    const selectedIds = Object.keys(teach.selectedExperiments).filter((key) => teach.selectedExperiments[key]);
    if (!selectedIds.length) {
      log('至少勾选一个课程实验', 'warn');
      return;
    }

    // 关卡列表以「现读」为准（面板里的可能过期），顺便拿到上报要用的关卡名。
    teach.busy = true;
    teach.status = '读取关卡…';
    renderTeach();
    const levelsByExperiment = {};
    for (const experiment of teach.experiments) {
      if (!teach.selectedExperiments[experiment.identifier] || experiment.jupyter) continue;
      try {
        levelsByExperiment[experiment.identifier] = await loadTeachChallenges(experiment.identifier);
      } catch (error) {
        teach.busy = false;
        teach.status = '关卡列表读取失败';
        renderTeach();
        log(`关卡列表读取失败（${experiment.name}）：${error.message}`, 'error');
        return;
      }
    }
    teach.levelsByExperiment = levelsByExperiment;
    teach.busy = false;

    const built = buildTeachQueue(teach.experiments, { experiments: selectedIds, levels: teach.selectedLevels }, levelsByExperiment);
    if (built.skippedJupyter.length) log(`虚拟机实验已跳过：${built.skippedJupyter.join('、')}`, 'warn');
    if (!built.total) {
      teach.status = '没有可作答的关卡';
      renderTeach();
      log('没有可作答的关卡（是不是全被跳过/全没勾？）', 'warn');
      return;
    }

    const course = teach.courses.find((item) => item.id === teach.courseId);
    const queue = {
      mode: 'course',
      courseName: course ? course.name : '',
      experiments: built.experiments,
      experimentIndex: 0,
      visited: [],
      expected: { identifier: '', position: 0 },
      running: true,
      handoffAt: Date.now(),
      startedAt: new Date().toISOString()
    };
    writeTeachQueue(queue);
    teach.status = `一键答题：0/${built.total}`;
    renderTeach();
    log(`一键答题开始：${built.experiments.length} 个实验 / ${built.total} 关`, 'ok');
    enterTeachExperiment(queue);
  }

  /** 「按清单再答」：直接照缺答案清单里的 URL 逐条跑。 */
  async function startMissingRun() {
    if (state.busy) {
      log('已有任务在执行（等它结束或点停止）', 'warn');
      return;
    }
    let items = [];
    try {
      items = await fetchTeachMissing();
    } catch (error) {
      log(`读取缺答案清单失败：${error.message}`, 'error');
      return;
    }
    const usable = items.filter((item) => item.url);
    if (!usable.length) {
      log(items.length ? '清单里的条目没有关卡 URL（旧数据？），无法直达' : '缺答案清单是空的', 'warn');
      return;
    }
    const queue = {
      mode: 'missing',
      items: usable.map((item) => ({
        id: item.id,
        course: item.course,
        experiment: item.experiment,
        level: item.level,
        level_id: item.level_id,
        identifier: item.identifier,
        url: item.url
      })),
      index: 0,
      running: true,
      handoffAt: Date.now(),
      startedAt: new Date().toISOString()
    };
    writeTeachQueue(queue);
    const teach = teachSelection();
    teach.status = `按清单再答：0/${queue.items.length}`;
    renderTeach();
    log(`按清单再答：共 ${queue.items.length} 关`, 'ok');
    location.href = queue.items[0].url;
  }
```

- [ ] **Step 2: 检查 + 提交**

Run: `node --check userscript/educoder-autosubmit.user.js`
Expected: 无输出

```bash
git add userscript/educoder-autosubmit.user.js
git commit -m "feat(userscript): 一键答题（选中）与按清单再答两个入口"
```

---

## Task C7: 全量测试 + 真机端到端 + 收尾

**Files:**
- Modify: `HANDOVER.md`（§7 当前状态 + §6 坑位）

- [ ] **Step 1: 全量测试**

Run: `node --test --test-concurrency=1 "test/**/*.test.js"`
Expected: 全绿（除既知的沙箱 flaky：整个文件级失败 `exitCode 3221225477` 时单独重跑该文件即可）

- [ ] **Step 2: 真机端到端（三条链路）**

1. **只提取答案（课堂）**：`npm run ui` → 选一门课 → 勾「只提取答案」→ 开始提取。
   验：日志只有「已导出答案 / 答案已是最新 / 只提取答案：未通过测评」；`exports/` 下**没有**新的实验目录；`exports/answers/` 有答案文件。
2. **教学课程一键答题**：装好脚本 → 打开 `https://www.educoder.net/paths/{id}` → 面板「教学课程」→ 刷新课程 → 读课程实验 → 勾一个实验 → 读关卡 → 「一键答题（选中）」。
   验：逐关自动作答；已通过/虚拟机跳过有日志；没答案的关卡在控制台「缺答案清单」里出现；跨实验能自动接着跑；中途「停止」立即生效。
3. **按清单再答**：给清单里某一关补上答案（AI 作答或手动导出到 `exports/answers/`）→ 篡改猴点「从本地拉答案」→「按清单再答」。
   验：只跑清单里的关卡；答完后清单条目自动移除、面板计数减少。

- [ ] **Step 3: 记录与收尾**

在 `HANDOVER.md` §6（已踩过的坑）追加真机发现（入口按钮的真实文案、关卡链回退的实际做法、签名常量双份维护的提醒）；
§7（当前状态）补上本批能力的入口与用法；若发现 bug，按既有格式记进 `BUG.md`。

- [ ] **Step 4: 提交**

```bash
git add HANDOVER.md
git commit -m "docs(handover): 只提取答案 / 缺答案清单 / 教学课程一键答题的入口与真机结论"
```

---

# 完成标准（Definition of Done）

- [ ] `node --test --test-concurrency=1 "test/**/*.test.js"` 全绿（除既知 flaky）。
- [ ] 控制台：勾「只提取答案」跑课堂，只出 answers、不产生题目存档；「缺答案清单」面板可看/刷新/清空。
- [ ] 篡改猴：教学课程页能选「课程 → 课程实验 → 关卡」；「一键答题（选中）」逐步跑完并跳过已通过/虚拟机；缺答案上报；「按清单再答」自动移除。
- [ ] 4 个新测试文件（`missing-store` / `missing-api` / `userscript-sign` / `userscript-teach`）+ 两个既有测试文件的追加用例全过。
- [ ] 真机三条链路验证通过（Task C7 Step 2）。