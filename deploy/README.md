# 学习目标管理台 · 静态部署版

一个零依赖的单文件学习打卡工作台：目标管理、每日打卡、补记、周复盘、热力图、三态主题（深/浅/跟随系统）。
部署到任意静态托管后即可使用，**无需后端、无需登录**。

- 入口：`index.html`（发布只需要这一个文件）
- 数据：浏览器 `localStorage`，按设备隔离
- 规模：约 168 KB 单文件，无外部 JS / CSS / 字体 / 图片引用，可完全离线运行

---

## 原理：换掉数据层，不动业务代码

原版（WorkBuddy 资料库版）的所有数据读写都经过一个 Promise 化的 SDK：

```js
var db = window.__SMART_PAGE__.database;   // getSchema / query / addRecord / updateRecord / deleteRecord
```

静态托管环境里没有这个 SDK。本包在 `<head>` 早期注入了一个**接口完全一致**的 localStorage
适配器，把它挂到同一个位置上：

```js
window.__SMART_PAGE__ = { database: LocalDB };
```

因此页面主脚本（IIFE + strict，约 2200 行）**一行未改**即可运行。适配器的关键约定：

| SDK 语义 | 适配器实现 |
|---|---|
| `properties` 是 `{字段名:{类型:值}}` 包装形态 | 存库时拆包为扁平的 `{字段名:值}`，按「对象里唯一的那个键」取值，不硬编码类型名 |
| 所有方法返回 Promise | 统一经 `setTimeout(…,60)` 转异步，保持「`then` 晚于当前 tick」的时序 |
| `query` 返回 `{results:[…]}` | 直接返回数组副本 |
| `getSchema` 返回 `{properties:[{name,type,config}]}` | 内联三张表的 schema，供「标签 / 状态」select 选项使用 |

三张表（沿用原库 ID 作为存储键后缀）：

| 表 | ID | 键 |
|---|---|---|
| 学习目标 | `OereifjY46k3XWmQX6KVfK` | `sgt_local_v1_OereifjY46k3XWmQX6KVfK` |
| 打卡记录 | `DRuD6oQI5lrK5IhZQS1fzn` | `sgt_local_v1_DRuD6oQI5lrK5IhZQS1fzn` |
| 周复盘 | `kVj8GOSzCqSOUhes5vYBhg` | `sgt_local_v1_kVj8GOSzCqSOUhes5vYBhg` |

### 首次访问的种子数据

三张表**全空**时写入 3 个示例目标 + 13 条示例打卡（日期按打开当天相对生成，不会过期），
并置标记 `sgt_local_seeded_v1`。示例目标带 `标签 = 示例`，
App 内 `我的 → 数据管理 → 清空示例数据` 可一键清除。

### 备份与迁移

主脚本包在 IIFE 里，外部拿不到内部状态，因此备份卡片以独立脚本注入：
用 `MutationObserver` 监听 `#tab-mine`（该面板每次进入都整块重渲染），
重建一次补一次卡片；事件用 `document` 级委托绑定，避免随 DOM 重建丢失。

导出结构：

```json
{ "format": "sgt-local-backup", "version": 1, "exportedAt": "...", "tables": { "<库ID>": [ … ] } }
```

### 多标签页同步

写入后置一个 `sgt_local_touch` 时间戳，其他标签页通过 `storage` 事件收到通知并调用页面
导出的 `window.loadAll()` 重新拉取。

---

## 部署

**Netlify**：拖文件夹到 https://app.netlify.com/drop （`netlify.toml` 已配好 `publish = "."`
与 `Cache-Control: no-cache`，保证更新即时生效）。

**GitHub Pages**：仓库根目录放本包文件，`Settings → Pages → main / root`。

**其他**：任何静态托管（Vercel / Cloudflare Pages / 对象存储 / 本地 `file://`）均可，
因为没有构建步骤，也没有路径相关的资源引用，放在子路径下同样正常。

---

## 与线上版的差异

| | 线上版 | 本版 |
|---|---|---|
| 存储 | 云端资料库，跨设备同步 | 本机 localStorage |
| 登录 | 未登录只读 | 不需要 |
| 备份 | 平台侧负责 | 用户手动导出 JSON |
| 功能 | — | 完全一致 |

---

## 已知限制

1. **不跨设备同步** —— 数据绑在「设备 + 浏览器」上，换环境需导入备份。
2. **清理浏览器数据会丢** —— 无痕/隐私窗口下 localStorage 可能被禁用或随窗口销毁。
3. **无服务端校验** —— 全部校验在前端，多端并发写入不存在，但手动改 localStorage 不会被拦。
4. **容量** —— localStorage 通常 5–10 MB；按每天 3 条打卡估算可用数十年。

---

## 维护

本包由根目录 `build-deploy.py` 从 `study-goal-tracker.html` 生成（注入适配器 + 本地化文案 +
备份卡片）。改了源文件后重新执行即可：

```bash
python build-deploy.py     # 源文件 -> deploy/index.html
python pack-deploy.py      # deploy/ -> 日期+学习目标管理台-部署包.zip
node verify-deploy.js      # 第一组：种子 / 读取 / 打卡 / 备份卡片 / 徽标（33 项）
node verify-deploy2.js     # 第二组：删除 / 新建 / 清空 / 清空后不复活（22 项）
```

> jsdom 需先安装，运行时要带 `NODE_PATH`：
> `NODE_PATH=C:/Users/27361/.workbuddy/binaries/node/workspace/node_modules node verify-deploy.js`

两组共 55 项断言，覆盖了「破坏性按钮是否真生效」
（项目里 `clearAll` / `confirmDeleteGoal` 一旦静默无操作，最难被察觉）。

## 许可

个人项目，可自由使用与修改。
