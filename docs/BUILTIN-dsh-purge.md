# 出厂内置 dsh-purge（破甲插件）

> 状态：0.15.x 起生效。目标是一句话：**APK 装完，插件已经在里面并且是启用状态。**

本文记录「怎么做的、为什么这么做、怎么验、怎么撤」，供后续维护。用户可见的入口是
README 的「它能做什么」列表与设置页里的 dsh-purge 面板。

## 1. 结论先行

| 关注点 | 落点 |
|---|---|
| 插件字节 | `vendor/dsh-purge/`（vendored，逐文件 sha256 绑定） |
| 注入集 | `scripts/plugin-dirs.json` → `externals` |
| 注入器 | `scripts/inject-all.py`（`--external vendor/dsh-purge`，由 `scripts/build-apk.mjs` 传入） |
| 装配面 | `scripts/profile-web.cordis.patch.yml` 末尾四段 insert + 一段 `system-prompt` 配置覆盖 |
| 版本/兼容登记 | `scripts/contract.json`（`inserted` + `runtimeCompat.noPeerRows`）、`scripts/contract-pin-gaps.json` |
| 路由鉴权登记 | `scripts/api-route-auth-policy.json`（`routeRegistrations` + `localRoutes`） |
| 与上游的差异 | `vendor/dsh-purge/PATCHES.md` |

## 2. 为什么走「快照注入 + 出厂装配」而不是「首次启动安装」

仓库里两条插件管理通道都不适合「安装后自带」：

- `PluginPatchOffline`（`app/src/main/java/.../PluginPatchOffline.kt`）只做 **disabled 覆盖**，
  不能新增 insert 条目 —— 它假设条目已在装配清单里；
- `PluginManagementRepository` 走引擎 `pluginManager` RPC，前提同样是「条目已存在」。

也就是说「自带」必须落在**装配清单与快照**这两处，而不是运行期安装动作。这也与本仓既有
vendor 插件（`dshmarketplace-plugin`、`dsh-plugin-subscriptions`）同一条链，收益是：首次安装
不需要网络、不受市场可用性影响、插件字节受 `check-vendor-provenance` 绑定。

## 3. 装配面：为什么是四个 insert 行

上游 `dsh-purge` 是一个包、四个装配面（`package.json` 的 `exports`）：

| 条目 id | `name:` | 作用 |
|---|---|---|
| `dsh-purge` | `dsh-purge` | 主包：system-prompt 装配改写、设置页数据面、工具面 |
| `dsh-purge-redteam` | `dsh-purge/redteam` | 演练台：预设 + 技能落地（读包内 `presets/`、`skills/`） |
| `dsh-purge-redteam-store` | `dsh-purge/redteam/store` | 演练台事实库（`node:sqlite`，数据在 `$DSH_HOME/redteam`） |
| `dsh-purge-redteam-ui` | `dsh-purge/redteam/ui` | 演练台 HTTP 桥（`POST /redteam/api`） |

只挂主包时设置页能开、演练台是空的（服务缺席 → 工具行 pending）。四行同进同出才是上游
「安装后」的形态。`name:` 里的 `/` 是 **exports 子路径**而不是「另一个包」——
`check-patch-mounts.mjs` 与 `check-contract.mjs` 都按包名归一后判定（两处判据同源）。

## 4. 注入面：为什么 `presets/` 要单独加宽

根级外部包的注入面由 `scripts/inject-all.py` 的 `build_ext_replacements` 决定，历史上是
`package.json + lib/** + skills/** + 清单文件`。`dsh-purge` 的演练台预设放在
`presets/redteam/{agent.cordis.yml,preset.yml,anchor-gate.mjs}`，运行期由
`lib/redteam/index.js` 读包内相对路径落到 `$DSH_HOME/.agent-presets/redteam/`。

不把它纳入注入面，症状是**设备上预设为空、而仓内成员/导入两条判据全绿**（经典的
「注入面漏一个目录」）。故三处同源加宽：

- `scripts/inject-all.py`（构建链实际调用的单 pass 注入器）
- `scripts/inject-external-plugins.py`（独立注入器，保持同一口径）
- `scripts/check-inject-completeness.mjs`（产物级对账的注入面定义）

## 5. 客户端 bundle 为什么放在 `lib/client.js`

上游把客户端产物放包根 `client.js`，`exports["./client"]` 指过去 —— 宿主解析这一点两边一致
（`dsh-client-modules` 的 `resolveClientRel` 认 `exports["./client"]`）。但本仓构建链的三条
判据按 `lib/client.js` 命名：浏览器语法下限降级（`build-apk.mjs`）、ES2021 可解析门禁、
combo 面。产物留在根目录会整体跳出这三条判据，而老内核（WebView 87）解析失败是整模块不执行
（静默白屏）。故 vendored 副本把它落到 `lib/client.js` 并改 `exports` 一行，细节见
`vendor/dsh-purge/PATCHES.md` 差异 1。

## 6. 路由鉴权：为什么单开一张 `localRoutes` 表

`dsh-purge` 的 16 条宿主路由（`/dsh-purge/*`）与演练台桥（`/redteam/api`）都不在 `/api`
前缀下 —— 而 `check-api-route-auth.mjs` 的 `routes` 表要求 `path.startsWith('/api/')`（那是
client-connection 鉴权门的载体）。既不登记会从「非字面量 path 表达式」判据逃逸，塞进 `routes`
又会判红。

故新增独立登记面 `localRoutes`，判据四条：

1. `paths` 每条都要在源文件里逐字出现（改路径不登记 ⇒ 红）；
2. 该 source 扫到的对应注册站点必须存在（删路由不删登记 ⇒ 红）；
3. `authCallMarker` 必须在该注册块内（剥注释后）—— 与 `routes` 的 protected 判据同口径；
4. 不得占用 `/api` 前缀（两表不许互相顶替）。

`dsh-purge` 的 guard 是 `allowPluginRequest(request, response)`：Host 头必须存在、Origin 的
host 必须与 Host 同源，否则 `denyCrossOrigin` 直接 403；两条注册包装
（`installWebServer` 的 `registerPluginRoute`、模块级 `registerRoute`）都把它套在 handler 最外层。

## 7. 怎么验

```bash
# 1) 静态面（不需要构建产物）
node scripts/check-patch-mounts.mjs scripts/profile-web.cordis.patch.yml \
  dsh-shell-termux dsh-client-ui-responsive dsh-host-web-compat \
  plugins/dsh-android-bridge plugins/dsh-android-manage plugins/dsh-android-linux-env \
  plugins/dsh-android-file-open plugins/dsh-android-browser plugins/dsh-android-vdisplay \
  plugins/dsh-session-delete plugins/dsh-linux-bootstrap \
  vendor/dshmarketplace-plugin vendor/dsh-plugin-subscriptions vendor/dsh-purge
node scripts/check-api-route-auth.mjs
node scripts/check-vendor-provenance.mjs

# 2) 注入链（合成快照即可，验成员齐不齐）
python scripts/make-synthetic-snapshot.py .deploy-tmp/synth/synth.tar.xz x86_64 \
  scripts/profile-web.cordis.patch.yml
python scripts/inject-all.py .deploy-tmp/synth/synth.tar.xz .deploy-tmp/synth/out.tar.xz \
  scripts/profile-web.cordis.patch.yml --dsh-android \
  --external vendor/dshmarketplace-plugin vendor/dsh-plugin-subscriptions vendor/dsh-purge
node scripts/check-inject-completeness.mjs .deploy-tmp/synth/out.tar.xz
node scripts/check-browser-syntax-floor.mjs --scan .deploy-tmp/synth/out.tar.xz

# 3) 设备面（真机构建后）
#    设置页出现「破甲 / 演练台」面板；插件管理里四条 dsh-purge 行均为启用态；
#    /dsh-purge/status 返回 ok:true；/redteam/api 对跨源请求返回 403。
```

## 8. 怎么撤

1. `scripts/profile-web.cordis.patch.yml`：删掉末尾 `system-prompt` 覆盖与四行 insert；
2. `scripts/plugin-dirs.json`：`externals` 删 `vendor/dsh-purge`；
3. `scripts/contract.json`：删 `inserted` 的 `dsh-purge` 行与 `runtimeCompat.noPeerRows` 的四条；
4. `scripts/contract-pin-gaps.json`：删 `vendor/dsh-purge` 条目；
5. `scripts/api-route-auth-policy.json`：删两条 `routeRegistrations` 与两条 `localRoutes`；
6. `scripts/vendor-provenance.json`：`node scripts/check-vendor-provenance.mjs --write`；
7. 删目录 `vendor/dsh-purge/`。

## 9. 已知边界（诚实登记）

- `skills/redteam/unauth-exploit.md` 未入库：Windows Defender 按 `Backdoor:PHP/Chopper.*`
  判据拦截该 markdown（连读取都拒绝），故本机无法落地。演练台仍有 22 个技能文件；
  追版时在无 AV 干扰的构建机上补齐并重跑 `--write`。详见 `vendor/dsh-purge/PATCHES.md` 差异 4。
- `lib/hide-console.js` 有一条动态模板导入（`./${IMPORT_MARK}.js`）——
  上游自身也不含该目标，属源码侧动态形态，`check-inject-completeness` 记 WARN 不判红。
- 本机未跑完整 `build-apk`（需要插件构建 + 引擎快照拉取 + gradle），故设备冒烟待构建机执行；
  静态面与注入链已在上面第 7 节逐条跑过。
