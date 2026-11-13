# 2026-11-13 上游同步（opb/sync-2026-11-13）

状态：已在 sts-mbp 的整合分支 `opb/sync-2026-11-13` 合入并做构建验证；**未部署**，未合并进 `main`、`opb/branding` 或 `opb/docs`。DeviceManager 任务 `TASK-20261113-task-dbf64b0a`。

## 背景

fork 的 `main` 停在上游 `2d74ba961`（v0.1.154）。上游 `milind-soni/OpenMausBot` 按 merge train 成批合入，到 2026-11-13 已超前 425 个提交（约 125 个 PR，含 TypeScript 7 工具链 #5500/#5644、导入与导出设置页 #5613、USB 手机接管、Fish Audio 通话、OMP 引擎、共享 VM 手机接管、1Password 单条目授权、云电脑自动恢复 #5640 等）。只读评估（box，2026-11-13 13:20–14:30）结论：只有一个机械冲突，兼容性合同只有新增。用户原话授权：「同意在 sts-mbp 建整合分支一次合入上游（含 TS7），加 OPB 修正提交，先在 sts-mbp 构建验证，不部署」。

## 做法

1. 从 `opb/docs`（= `opb/branding` + 文档）切出 `opb/sync-2026-11-13`。
2. `git merge --no-ff upstream/main`，上游尖端 `ee89b889c97f208231e22957ca36e5896f1329a1`（#5640，评估时为 `41dc73051`，之后又进了 #5674、#5640）。不 rebase，保留上游历史，便于以后继续合并。
3. 唯一冲突 `electron/lending-indicator.mjs`：上游 #5623 改写托盘 tooltip（加入正在操作的 bot 名 `acting`），我们把同一行前缀改成了 `OpenPigeonBot:`。解法：取上游新代码，前缀保留 `OpenPigeonBot:`。
4. 合并之后单独加 OPB 修正提交（不混进 merge）：
   - 服务端读名：`server/brand.ts` 新增的 `PRODUCT_NAME`（上游 `"MausBot"`，用于 Fish agent 命名）改为 `"PigeonBot"`；`DEFAULT_BRAND`（`"OpenMausBot"`）不动。CLI 配对提示、phone-secret、adb 报错里指本机桌面 App 的 `OpenMausBot` 改为 `OpenPigeonBot`，并同步测试。
   - 桌面显示名：`electron/system-tray.mjs` tooltip、`electron/startup-screen.mjs` 窗口标题与标题行改为 `OpenPigeonBot`（同步 `startup-tray.node-test.mjs` 与 `scripts/verify-startup-tray.mjs`）。托盘菜单文字来自 native locale，仍待 locale 决定，不在本次改。
   - 测试期望：`lend-automatically.node-test.mjs`（上游 #5623 新断言）、`src/pair/PairPage.test.ts`、`scripts/testing/web-app-assets.test.ts` 改为 OPB 名字。
5. 不改：`FISH_UPDATE_PREFIX="[OpenMausBot]"`（协议标记）、`PHONE_SECRET_INFO`（加密 info 串）、`OpenMausBot Speech.app` 包名、MausBot Cloud / MausBot Relay / 官方 MausBot 手机 App / MausBot team 等指上游服务或团队的字样、端口、`OMB_*`、配置键、schema、链接、Bonjour。

## 接口影响

只有新增，无删除或改形：

- 新 API：`/api/usb-phones`（含 WS stream）、`/api/live/relay`、`PUT/PATCH /api/logins/1password/items`、`/api/internal/cloud-computer/reset`、`/api/internal/post-to-thread`、`PUT /api/groups/order`、线程附件 `?media=` GET；#5640 另加云电脑恢复路由（`server/routes/cloud-computer-recovery.ts`）。
- WS / wire：可选字段 `threadPost`、事件 `groups.order`、Fish relay 事件。
- 新环境变量：`OMB_FISH_LIVE_AGENT_ID`、`OMB_FISH_LIVE_URL`、`OMB_CLOUD_COMPUTER_AUTO_REPLACE`（另有测试用 `OMB_EMAIL_REAL_TEST`）。
- 新可选配置键：`features.usbPhoneTakeover`（默认关）、`features.cloudComputerAutoReplace`（默认关）、`live.provider:"fish"`、`live.fishAgentId` / `fishAgentFor`。
- `openmausbot://thread/...` 新路径（#5505）。
- 不变：端口 8799/8800/8810/8811、`_openmausbot._tcp`、配对 / QR 格式、`/api/health` 的 `app:"openmausbot"`、`/api/brand` 默认值、`~/.openmausbot`、本地 DB schema。
- 行为变化（需知晓）：手机可接管共享本机 VM，人工接管时该桌面上所有 bot 暂停；Move from Grok Bot 入口移到「设置 → 导入与导出」（#5613）。

## 取舍

- 一次合入而不是挑拣：上游 train 之间互相合过 main，cherry-pick 低风险部分要拆几十个提交且更易冲突；整体合入只有 1 个冲突。代价是一并带进 TS7 和新功能，因此先请用户批准（已批）。
- 品牌修正与 merge 分开提交：以后对照上游时，`git log --first-parent` 能清楚区分上游代码与 OPB 改动。
- 只改明确的显示名；locale 批量字样、`brand().name` 展示点、图标仍按 HANDOFF 待用户决定。

## 验证

2026-11-13 在 sts-mbp（macOS 26.6.2，arm64，Node 25.2.1，pnpm 10.33.0）上，对 `c1bfe1b35` 的树执行（该树与 box 上先行验证的树 hash 一致：merge 树 `38a2c2b2b`，修正后树 `29b7e7c85`）：

- `pnpm install --frozen-lockfile --registry=https://registry.npmmirror.com`：成功（直连 npmjs 经代理下载 TS7 原生包时 ECONNRESET，改用镜像；锁文件 integrity 校验不变）。
- `pnpm typecheck`（tsc 7.0.2）：通过。`pnpm lint`：通过。`pnpm i18n:check`：通过（16 种语言，10652 条英文串）。
- `pnpm test:electron`：2063 项，2050 通过，0 失败，13 跳过。
- 重点 vitest（brand、phone-secret、PairPage、web-app-assets、live-session-wiring、UpdateIndicator、ImportSettings、SettingsWindow 等 9 个文件）：96 项全部通过。
- 全量 vitest：本轮未跑（box 上 2026-11-13 的全量结果为 36787 项中 167 失败，多为环境相关 e2e，见评估报告）。
- 构建：`pnpm build`、`pnpm build:server`、`pnpm build:companion` 通过；`CSC_IDENTITY_AUTO_DISCOVERY=false pnpm exec electron-builder --mac dir --arm64 -c.mac.identity="-" -c.electronDist=node_modules/electron/dist -c.directories.output=~/.local/opb-sync/release --publish never` 成功（不加 `electronDist` 时 electron-builder 会从 GitHub 重新下载 Electron，在本机网络下失败）。
- 产物 `~/.local/opb-sync/release/mac-arm64/OpenPigeonBot.app`（795M，0.1.154）：`codesign -dv` 主程序 `Identifier=com.stephenfeng.openpigeonbot`，`Signature=adhoc`，`flags=adhoc,runtime`，`TeamIdentifier=not set`；四个 Helper 为 `com.stephenfeng.openpigeonbot.helper[.GPU/.Plugin/.Renderer]`；`codesign --verify --deep --strict` 通过；entitlements 含 `disable-library-validation`、`allow-jit`。`spctl --assess` 为 rejected（未公证，预期内）。**签名修正方案（`-c.mac.identity="-"`）在构建侧验收通过，无需 `codesign --deep`。**
- 冒烟：用包内 Electron 以 `ELECTRON_RUN_AS_NODE=1`、临时 `OMB_DATA_DIR` 启动包内 `server/index.js` 数秒：只监听 `127.0.0.1:8799` / `127.0.0.1:8800`；`/api/health` 返回 `app:"openmausbot"`；`/api/brand` 返回 `{"brand":{"name":"OpenMausBot"},"source":"default",...}`。随后停止，端口释放。完整 GUI 启动、窗口与引擎对话未验证。

## 回滚

- 整合分支未合并进任何长期分支；放弃时不合并即可（删除远端分支须用户同意）。
- `opb/docs`、`opb/branding`、`main` 均未改动。
- sts-mac 上的 `/Applications/OpenPigeonBot.app` 未替换；部署须用户在聊天里明确同意。
