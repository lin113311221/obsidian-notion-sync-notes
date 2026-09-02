# Obsidian 插件怎么测：`require('obsidian')` 进不了你的测试套件

Obsidian 插件有个结构性难题：入口文件顶部要 `require('obsidian')`，
而这个模块**只在 Obsidian 运行时里存在**。你的 `node tests/run.js` 一跑就挂。

最常见的应对是「那就只测纯函数吧」——然后你的插件层就永远没有测试覆盖。
下面是我们踩出来的路。

---

## 分层：core 不许 require('obsidian')

```
core/          纯 JS，零 Obsidian 依赖，Node 下可测
  ├─ providers/   平台差异
  ├─ targets/     写入差异（obsidian.js / notion.js）
  └─ engine.js    唯一流水线
plugin/        只做 DOM、设置页、生命周期
  └─ main.js      require('obsidian')，不进测试
```

铁律：**`core/` 下任何文件都不允许 `require('obsidian')`**。

需要 Obsidian 的能力（比如路径归一化）怎么办？在 `core/` 里写一份纯函数实现：

```js
// core/model.js —— 不依赖 Obsidian 的等价实现
function normalizePath(path) { /* 连续斜杠 / 首尾斜杠 / Unicode NFC */ }
```

---

## 真实 bug 1：防御性兜底把故障藏了起来

我们有个 bug 藏了很久：

```js
// core/targets/obsidian.js
const { normalizePath } = require('../model');
// ...
const p = normalizePath ? normalizePath(raw) : raw.replace(/\\/g, '/');  // ← 问题在这里
```

`core/model.js` **从来没有导出** `normalizePath`。但代码不会报错 ——
三元表达式判断它为 undefined，静默走兜底分支。

结果：Obsidian 官方的路径归一化（连续斜杠 / 首尾斜杠 / Unicode NFC）
**实际一直没生效**，而测试全绿。

> **判断标准：依赖缺失就该当场炸掉，不要写兜底三元把故障藏起来。**
>
> 兜底三元只在一种情况下合理：你已经想清楚了「缺失时应该怎么降级」，
> 并且这个降级行为是被测试覆盖的。否则它就是一个静默的 bug 发生器。

---

## 真实 bug 2：接线写在不可测的层里，永远测不到

这个更隐蔽，是我们用变异测试抓出来的。

场景：对话功能的检索要做两步 —— 先从用户问题里提取过滤条件，再把条件传给检索：

```js
// plugin/main.js（进不了测试套件）
const filter = core.extractFilter(q);
const hits = core.search(idx, q, { topK: 8, filter });   // ← 这行写漏 filter 怎么办？
```

`extractFilter` 有测试，`search` 有测试，**两个都对**。
但把它们接起来的这一行，**没有任何测试覆盖**。写漏 `filter` 参数，功能整个失效，测试依然全绿。

我们真的写漏过一次，是变异测试抓出来的。

### 解法：把接线收进可测的层

```js
// core/search.js —— 新增一个函数，把两步合一
function retrieve(index, question, opts) {
  const o = opts || {};
  const filter = o.filter != null ? o.filter : extractFilter(question, o.now);
  const hits = search(index, question, { filter, topK: o.topK, minScore: o.minScore });
  return { hits, filter };   // filter 一并返回，便于 UI 显示"我理解成什么了"
}
```

插件层只调这一个：

```js
const { hits, filter } = core.retrieve(idx, q, { topK: 8 });
```

> **判断标准：接线（wiring）写在不可测的层里 = 永远测不到。**
>
> 与其给不可测的层补测试（成本高、收益低），不如**把接线收进可测的层**。
> 这不是为了好看，是为了让它能被测试覆盖到。

---

## 变异测试：证明「测试全绿」不等于「没坏」

测试全绿是一个很弱的信号。要验证你的测试**真的能抓到 bug**，做法是主动注入 bug。

写一个脚本，往源码里注入真实世界的回归，跑测试，看它是否失败：

```js
const MUTANTS = [
  {
    name: '平台过滤失效',
    file: 'core/search.js',
    from: 'if (want.length && !want.map(String).includes(String(e.platform))) return false;',
    to: 'if (false) return false;',
  },
  {
    name: '日期比较不截前 10 位',
    file: 'core/search.js',
    from: "const day = (v) => String(v || '').slice(0, 10);",
    to: "const day = (v) => String(v || '');",
  },
  // ...
];

for (const m of MUTANTS) {
  writeFileSync(p, original.replace(m.from, m.to));   // 注入
  const failed = runTests() !== 0;                   // 跑测试
  writeFileSync(p, original);                        // 立刻还原
  if (!failed) console.log(`✗ 漏网：${m.name} —— 这里有盲区`);
}
```

**测试结果仍全绿 = 这个变异体"存活" = 你的测试有盲区。**

几个要点：

- 每个变异体都要对应**一类真实事故**，不要为了凑数造假变异
- 跑完**立刻还原**，不留脏文件
- 变异体存活时，**补的是测试断言，不是删掉变异体**
- 我们目前 13 个变异体全部被抓到（13/13）

变异测试最大的价值不是覆盖率数字，而是**能指出你以为测到了、其实没测到的地方**。

---

## 附：断言要写死期望形状

不要写这种断言：

```js
ok(hits.length > 0, '检索有结果');        // 太弱，过滤失效了照样过
ok(filter.platform, '提取到平台');        // 同上
```

写这种：

```js
eq(r.hits.length, 1, '限定平台和月份后只剩 1 条');
eq(r.hits[0].entry.title, '8 月的 B站 Rust', '命中的正是那条');
// 反证：如果接线断了（filter 没传进去），这里会是 3
ok(r.hits.length < 3, '必须比不加过滤的结果少');
```

**加一条反证断言**，能抓住"功能整个失效但返回值仍非空"这类最阴的 bug。

时间相关的断言一定要注入固定时间，否则明年自己失效：

```js
const T = new Date('2026-09-02T10:00:00');   // 不要 new Date()
const f = extractFilter('上个月收藏的', T);
eq(f.savedAfter, '2026-08-01');
eq(f.savedBefore, '2026-08-31');             // 注意是 8/31 不是 9/1
```

---

**上一篇**：[05-notion.md](05-notion.md)　**下一篇**：[07-rag.md](07-rag.md)
