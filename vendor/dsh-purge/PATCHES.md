# vendor/dsh-purge 与上游的差异

本目录是 **dsh-purge**（DeepSeek Harness 破甲插件）的 vendored 副本，随 APK 出厂注入
（`scripts/plugin-dirs.json` 的 `externals`），并在 `scripts/profile-web.cordis.patch.yml`
里挂载为出厂插件。本仓立论与 `vendor/dsh-plugin-subscriptions/PROVENANCE.md` 同一条：
**除下表所列之外，其余文件与上游逐字节一致**；落地字节由
`scripts/vendor-provenance.json` 逐文件 sha256 + 字节数绑定
（`node scripts/check-vendor-provenance.mjs`），改一个字节就要显式重跑 `--write`，那次改动因此在
diff 里可见、可审。

## 来源

| 项 | 值 |
|---|---|
| 包名 / 版本 | `dsh-purge` 1.1.71 |
| 取证来源 | `https://github.com/evachxji/dsh-purge`（`master`，2026-11-12 取） |
| 上游作者仓 | `https://github.com/YuJunZhiXue/dsh-purge`（上游即将删库，镜像为唯一可取证来源） |
| 授权 | MIT（见 `LICENSE`） |
| 上游署名要求 | 见 `README.md` 的署名章节与 `MIRROR-NOTICE.md`；二开须保留原作者署名 |

## 与上游的差异（3 项，全部为本仓装配面适配，**不改运行逻辑**）

### 1. 客户端产物 `client.js` → `lib/client.js`

- 上游：`client.js` 放在包根，`package.json` 的 `exports["./client"]` 指向 `"./client.js"`。
- 本仓：产物在 `lib/client.js`，`exports["./client"]` 指向 `"./lib/client.js"`。
- 为什么：宿主按 `exports["./client"]` 定位客户端 bundle 这一点两边一致（`dsh-client-modules`
  的 `resolveClientRel`），但本仓的**构建链**对客户端产物有三条按 `lib/client.js` 命名的判据：
  浏览器语法下限降级（`scripts/build-apk.mjs` 的 `--degrade` 只处理 `lib/client.js`）、
  ES2021 可解析门禁（`check-browser-parse-floor.mjs` 的清单 = `dist/**/*.js` + 每个 `lib/client.js`）、
  combo 面（`check-combo-cache.mjs`）。产物留在根目录会**整体跳出这三条判据**——
  老内核（WebView 87）解析失败时是整模块不执行，属于静默白屏级风险。
- 可逆：把文件移回根 + 改回 `exports` 即可，但那样必须同时给上述三条判据补上根级入口。

### 2. `package.json` 的 `files` / `repository` / `dsh.install`

- `files`：去掉 `docs`、`scripts`、`screenshots.json`（本目录不入库，见差异 3）；
  保留 `lib`、`bin`、`presets`、`skills`、`cordis.patch.yml`、README、LICENSE。
- `repository` / `homepage` / `bugs` / `dsh.install.{tarball,zip,git}`：指向上游作者仓的
  链接改为本仓镜像 `evachxji/dsh-purge`。上游仓即将删除，保留旧链接会让插件页的
  「更新/主页」入口 404；`dsh.install` 是插件自更新用的搬运地址，出厂插件的
  `autoUpdateOnStart` 为 false，运行期不触发。

### 3. 不入库的目录/文件

上游仓里以下内容**不随本目录入库**（也因此不进 APK）：

| 路径 | 为什么 |
|---|---|
| `.github/` | 上游 CI 工作流（release.yml），与本仓构建链无关 |
| `docs/` | 预览截图与公告图（约 0.9 MB），APK 里用不到 |
| `scripts/` | 上游构建脚本（`embed-default-prompt.mjs` / `build-client.mjs`），产物已入库 |
| `screenshots.json` | 上游更新器的「保留文件」白名单，本仓不做插件自更新 |
| `release-notes.md`、`.gitignore`、`.npmignore` | 上游发布/仓库杂项 |

### 4. `skills/redteam/unauth-exploit.md` 未入库（构建机 AV 隔离，**追版时必须补齐**）

- 现场：本目录在 Windows 上落盘后，Windows Defender 实时保护把该文件判为
  「包含病毒或潜在的垃圾软件」，随后**拒绝任何进程读取**（Node `fs` 报 `errno -4094 UNKNOWN`、
  Python 报 `OSError: [Errno 22] Invalid argument`、PowerShell 报同一条拦截文案）。
  内容来自上游同一 zip 包（8298 B，其余 108 个文件全部逐字节落地成功），差异只在 AV 判据。
- 处置：该文件从本目录移除，并在 `scripts/vendor-provenance.json` 的清单里如实缺席
  （判据 ②「目录下每个真实文件都必须在清单里」与 ③「清单不得含已不存在的文件」两侧一致）。
- 影响面（诚实登记）：`skills/redteam/` 仍有 22 个技能文件，缺的是「未授权访问利用」这一个；
  `lib/redteam/skill-availability.js` 按目录枚举，缺一个文件只会少一条技能，不报错、不影响装配。
- 追版/修复：在**无 AV 干扰的构建机**（Linux / WSL / CI）上从上游
  `skills/redteam/unauth-exploit.md` 补齐，然后 `node scripts/check-vendor-provenance.mjs --write`。
  补回时本节的差异 4 应当删除。

运行期读取面因此**只增不减**：`lib/**`（含 `lib/redteam/**`）、`skills/redteam/**`、
`presets/redteam/**`、`bin/`、`cordis.patch.yml`、README、LICENSE、MIRROR-NOTICE。

## 注入面（本仓构建链携带什么）

`scripts/inject-all.py` 的 `build_ext_replacements` 对根级外部包携带：
`package.json` + `lib/**`（`.map` 除外）+ `skills/**` + **`presets/**`** + 在场的清单文件
（`cordis.patch.yml` / `spec.json` / `README.md` / `README.zh-CN.md` / `LICENSE`）。

`presets/**` 是本仓为 dsh-purge 显式加宽的第三棵树（0.15.x）：上游 `lib/redteam/index.js` 在
首次装配时读包内 `presets/redteam/{agent.cordis.yml,preset.yml}` 并落到
`$DSH_HOME/.agent-presets/redteam/`，而它既不是 `lib/` 也不是 `skills/` ——
旧注入面下会被静默丢弃，设备上表现是「演练台预设为空」而仓内成员/导入两条判据全绿。
三处同源：`scripts/inject-all.py`、`scripts/inject-external-plugins.py`、
`scripts/check-inject-completeness.mjs`。

## 装配面（挂在哪个 profile）

`scripts/profile-web.cordis.patch.yml` 末尾四段：

- `- id: system-prompt` 配置覆盖（`includeHarnessIdentity: false` / `personaPrefix: ""`）
- `- insert: dsh-purge`（主包：prompt 装配改写 + 设置页）
- `- insert: dsh-purge/redteam`（预设 + 技能落地）
- `- insert: dsh-purge/redteam/store`（SQLite 事实库，`$DSH_HOME/redteam`）
- `- insert: dsh-purge/redteam/ui`（HTTP 桥 `POST /redteam/api`）

与上游 `cordis.patch.yml` 的两处有意偏差（上游顶层那条 `- id: dsh-purge` + `inject` 覆盖不再单列；
redteam store 不写 `root: !!js dshHomePath('redteam')`）在 patch 文件里有逐条理由。

## 维护

```bash
# 追版：整树覆盖后重新绑定字节
node scripts/check-vendor-provenance.mjs --write
# 门禁（应与构建链同口径）
node scripts/check-vendor-provenance.mjs
node scripts/check-patch-mounts.mjs scripts/profile-web.cordis.patch.yml <dirs...> <externals...>
node scripts/check-api-route-auth.mjs
```
