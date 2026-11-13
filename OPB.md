# OpenPigeonBot (OPB)

OpenPigeonBot（以下简称 OPB）是 StephenFeng 基于
[OpenMausBot](https://github.com/milind-soni/OpenMausBot)（Apache-2.0）的下游构建。
它与 OpenMausBot 维护者、MausBot Cloud 或 xAI 没有隶属关系。

本文是定制版总览：说明与上游的差异、必须遵守的兼容性合同、构建与部署方式，以及开发流程。进度与未完成项见 `docs/opb/HANDOFF.md`，按日期的变更记录见 `CHANGELOG-OPB.md`，单功能设计见 `docs/opb/design/`。

## 分支与远程

| 名称 | 含义 |
| --- | --- |
| `origin` | `https://github.com/5tephenFeng/OpenPigeonBot`（本 fork，公开） |
| `upstream` | `https://github.com/milind-soni/OpenMausBot` |
| `main` | 镜像 `upstream/main`；不要在这里提交 OPB 改动。当前对齐的上游提交是 `2d74ba9616270b4a1df1ebfa4a912b55bd5cb4ea`（2026-11-12） |
| `opb/branding` | OPB 品牌化及 fork 专用改动，相对 `main` 变基或合并 |
| `opb/docs` | 本次文档补齐分支，从 `opb/branding` 切出，首个提交 `311b3513a`。不要合并进 `main` |
| `opb/sync-2026-11-13` | 2026-11-13 上游整合分支：从 `opb/docs` 切出，`--no-ff` 合入上游 `ee89b889c`，再加 OPB 修正提交。已在 sts-mbp 构建验证，未部署、未合并进其他分支。见 `docs/opb/design/2026-11-13-upstream-sync.md` |

同步：`git fetch upstream && git checkout main && git merge --ff-only upstream/main && git push origin main`，
然后把 `opb/*` 分支跟上 `main`，并重跑下文的检查。

上游跟踪节奏、何种更新可自行合并，见「角色与流程」。

## 兼容性合同

OPB 必须能与官方 iOS、Android 和桌面客户端一起工作。下列内容不要改：

- HTTP / WebSocket 路由与载荷
- 伴侣配对协议，以及配对链接 / 二维码格式
- `openmausbot://` URL scheme 与 app-link 域名
- Bonjour 类型 `_openmausbot._tcp`
- 默认端口（8799 / 8800 / 8810 / 8811…）
- `OMB_*` 环境变量
- `config.json` 的键
- MCP / ACP 协议
- 其他脚本所依赖的 npm 包名
- 数据库 schema
- 数据目录 `~/.openmausbot`
- `GET /api/brand` 的默认值：在没有可用的白标 `brand.json`、或没有 `whitelabel` 授权时，服务端（`server/brand.ts` 的 `loadBrand()` / `DEFAULT_BRAND`）返回 `source: "default"`，`brand` 为 `{ "name": "OpenMausBot" }`，并带上所咨询的 `file` 路径（以及可能的 `notice`）。不要改这条路由，也不要改默认 `brand` 对象或载荷形状。客户端 `src/lib/brand.ts` 的 `productName()` 在 `source !== "file"` 时固定返回 `PRODUCT_NAME`（PigeonBot），`document.title` 也走这条路径。`applyBrand()` 仍会把 `brand().name` 换成服务端的 `"OpenMausBot"`。直接读取 `brand().name` 做展示的调用点尚未穷尽排查。本地 `DEFAULT_BRAND`（`OpenPigeonBot`）只是请求失败时的回退，不是接口默认值。
- `GET /api/health` 必须继续返回 `app: "openmausbot"`（`server/routes/status.ts`）。不要改这条路由或该字段。响应里还有 `pid`、`static`、`capabilities` 等字段，同样不要擅自改形状。

品牌化只应落在：`src/lib/brand.ts`（`PRODUCT_NAME`、客户端 `DEFAULT_BRAND`）、`src/lib/app-links.ts`、`index.html`、`rail.html`、`public/manifest.webmanifest`、`electron-builder.yml`（appId `com.stephenfeng.openpigeonbot`、productName、产物名、macOS 用途说明、publish 指向本 fork）、`package.json`（`productName`、链接），以及桌面窗口 / 托盘标题。

## 与上游差异（已完成）

相对 fork 的 `main`（`2d74ba961`），`opb/branding`（尖端 `0ec29a9fb`）超前 6 个提交、落后 0 个，共改 66 个文件。这 6 个提交都在 2026-11-12（Asia/Shanghai）：

1. `3fd3f4f4b570` — OPB: name the app PigeonBot / OpenPigeonBot in the UI
2. `470eb4bed932` — OPB: package the desktop app as OpenPigeonBot
3. `8c7b654bf7f9` — OPB: fork notes, README banner and agent pointer
4. `40745da2b9fa` — OPB: tray tooltip tests follow the OpenPigeonBot name
5. `a5d117ba50a6` — OPB: component test expectations follow the PigeonBot / OpenPigeonBot names
6. `0ec29a9fb043` — OPB: last two test expectations for the fork docs link and the PigeonBot updates label

已做的品牌化：人对人显示的产品名是 PigeonBot，项目 / 安装包 / `productName` 是 OpenPigeonBot，macOS Bundle ID 是 `com.stephenfeng.openpigeonbot`；文档链接指向本 fork。详细落点、locale 统计、图标清单和漏改见 `docs/opb/design/2026-11-12-branding.md`。

**接口未动。** 上述 66 个文件里没有服务端路由、配对协议、数据库 schema 或数据目录的改动；`server/brand.ts` 与 `server/routes/status.ts` 均未修改，因此 `/api/brand` 的默认 `brand.name` 仍是 `"OpenMausBot"`，`/api/health` 仍返回 `app: "openmausbot"`。

**Move from Grok Bot 已在当前树中**（不是必须先合并上游积压才能用）：实现位于 `shared/grok-import.ts`、`server/grok-local.ts`、`src/components/GrokMoveDialog.tsx`、`GrokMovePopup.tsx`、`src/lib/grok-move.ts`、`src/components/ImportSettings.tsx`，说明见 `docs/team-sharing.md`。手动入口是 Settings → Import → Move from Grok Bot，没有用品牌常量隐藏。合入上游 #5613 之后（`opb/sync-2026-11-13` 起），入口改为「设置 → 导入与导出 → Move from Grok Bot」（英文 Settings → Import & export），简单与高级模式都可见。首次自动提示（`GROK_MOVE_NOTICE`）会避开 Team Cloud、托管成员、`brandStatus().source === "file"` 的白标构建，以及组织桌面；当前 OPB 默认仍是 `source: "default"`，按代码不会仅仅因为改了显示名就关掉该提示。导入是本地、离线解析，不登录 Grok 云。试迁移顺序见 `docs/opb/HANDOFF.md`。

尚未改、需用户决定的事项（locale 字样、图标、暂存清理等）列在 `docs/opb/HANDOFF.md`，不在本次已完成范围内。

## 构建、签名与部署

本仓库使用 `electron-builder` 26.15.3。该版本里 `mac.identity` 为 `null` 会完全跳过签名；`mac.identity` 为 `"-"` 才走内置 ad-hoc 签名。对应的 CLI 是 `-c.mac.identity=...`（不要写成较新文档里的 `-c.mac.sign.identity`，那不是 26.15.3 的字段路径）。`electron-builder.yml` 和 `package.json` 的 `package:mac` 脚本本身没有写 `identity`。详细方案见 `docs/opb/design/2026-11-13-signing-fix.md`（未实施）。

### 0.1.154 实际用过的命令（构建会留下 Identifier=`Electron`，不要再作为标准流程）

```sh
pnpm install            # Node >= 24, pnpm 10.33 via packageManager
pnpm typecheck
pnpm test               # or a subset: pnpm exec vitest run src/lib
pnpm build && pnpm build:server && pnpm build:companion
CSC_IDENTITY_AUTO_DISCOVERY=false pnpm exec electron-builder --mac dir --arm64 \
  -c.mac.identity=null --publish never
```

构建产物在进入 sts-mac 暂存目录时，Identifier 仍为 `Electron`（`flags=adhoc,linker-signed`）。随后在暂存副本上执行了 `codesign --deep --force --sign -`，并用 `codesign --verify --deep --strict` 核验通过。此后 Identifier 已改为 `com.stephenfeng.openpigeonbot`（`Signature=adhoc`，`TeamIdentifier=not set`）。这是已经发生的补救，后续应改走下一节的计划，而不是重复 `--deep`。

去隔离当时写了「清除隔离属性」，但证据目录和操作记录里没有留下完整的 `xattr` 命令行，具体参数仍待确认。今后去隔离用 `xattr`（常见写法是对应用包执行 `xattr -cr`）。必须先完成签名修正并验收，再授予完全磁盘访问（FDA）和辅助功能；不要在构建流水线仍产出 Identifier=`Electron` 的包、或尚未用新流程验收时先授权。

禁止用上游身份对 OPB 产物做签名、公证，或发布到上游的 release 仓库。构建产物（`dist*`、`release/`、`node_modules/`）不要进入 Syncthing；见所同步 AI 文件夹的 `.stignore`。

### 计划中的构建命令（未实施，待在 sts-mbp 上实测）

```sh
CSC_IDENTITY_AUTO_DISCOVERY=false pnpm exec electron-builder --mac dir --arm64 \
  -c.mac.identity="-" --publish never
```

`build/entitlements.mac.plist` 已包含 `com.apple.security.cs.disable-library-validation` 和 `com.apple.security.cs.allow-jit`；`electron-builder.yml` 的 `entitlements` 与 `entitlementsInherit` 都指向该文件。改用 ad-hoc 时必须保留这两项，否则在 `hardenedRuntime: true` 下可能无法启动。

计划中的验收（待在 sts-mbp 上实测，不要把尚未跑过的结果写成已通过）：对主程序执行 `codesign -dv --verbose=4`，确认 Identifier 在构建完成后、未再手工 `--deep` 的情况下就不是 `Electron`。主程序的 bundle id 按配置应为 `com.stephenfeng.openpigeonbot`；各 Helper 的 Identifier 不一定等于同一个 appId，以实测为准。

### 当前部署记录（现场核对：2026-11-13 11:44 Asia/Shanghai）

- 在 **sts-mbp** 上用上面的 `identity=null` 流程，构建了未公证的 arm64 `dir` 包，版本 0.1.154。
- 产物经 rsync 到 sts-mac 的 `~/.local/opb-stage/OpenPigeonBot.app`（该目录确是部署暂存；2026-11-13 现场 `du -sh` 为 773M），再按上一节所述在暂存副本上 `codesign --deep --force --sign -`，然后用 `ditto` 安装到 `/Applications/OpenPigeonBot.app`。
- 当前安装：路径 `/Applications/OpenPigeonBot.app`；`CFBundleShortVersionString` 与 `CFBundleVersion` 均为 `0.1.154`；`CFBundleIdentifier` 为 `com.stephenfeng.openpigeonbot`。`codesign -dv --verbose=2`：`Identifier=com.stephenfeng.openpigeonbot`，`Signature=adhoc`，`TeamIdentifier=not set`。注意：这是 `--deep` 补签之后的状态，不是构建刚结束时的 `Identifier=Electron`。
- `xattr -l`：没有 `com.apple.quarantine`，仅见 `com.apple.provenance`。`spctl` 在部署当时对暂存副本仍为 rejected，去掉隔离后用 `open` 启动成功。
- 数据目录保持 `~/.openmausbot`。部署记录还提到 Electron 用户目录在 `~/Library/Application Support/OpenPigeonBot`。
- 进程监听（现场 `lsof`，进程名 OpenPigeonBot，pid 41342）：`127.0.0.1:8799` 与 `127.0.0.1:8800`。`8810` 与 `8811` 当前未监听；部署当时（pid 20978 / 20933）同样未监听这两端口。
- 现场 `GET http://127.0.0.1:8799/api/health` 正文为：`{"app":"openmausbot","pid":41342,"static":true,"capabilities":{"guardedMessages":1,"guardedRequests":1,"guardedFullAccess":1,"guardedOnBehalfOf":1},"box":"off"}`（无令牌等敏感字段）。`GET /api/brand` 正文为：`{"brand":{"name":"OpenMausBot"},"source":"default","file":"/Users/stephen-feng/.openmausbot/brand.json"}`。页面标题在部署检查里为 PigeonBot，与 `productName()` 一致。
- 远程访问只走应用内的 Tailscale，不要把服务端口暴露到公网或局域网。该通道的开通状态见 `docs/opb/HANDOFF.md`（仍在跟进）。
- 替换已安装版本需要用户同意，不要在未同意时覆盖 `/Applications` 里的安装。
- 用户与系统的 LaunchAgents（以及 LaunchDaemons 文件名）中没有与 OPB 有关的自启项。开机自启、主机控制、完全磁盘访问（FDA）尚未配置。授 FDA / 辅助功能必须排在签名修正之后。

## 角色与流程

- **分工：** 在 sts-mbp 上开发；sts-mac 只做部署目标，不在上面做日常开发。
- **上游跟踪：** 每周一、周四 09:00（北京时间 / Asia/Shanghai）查看 `upstream/main` 是否比 fork 的 `main` 有新提交。只同步与评估，不要在未获授权时合并到 `main` 以外的约定之外去改合同。
- **自行处理 vs 交用户评估：** 低风险、且不影响定制设计和兼容性合同的上游更新，可以自行合并进 `opb/*` 并部署。会触及品牌化设计、兼容性合同、数据目录、签名方式、对外访问或用户可见行为的更新，交给用户评估后再做。
- **DeviceManager：** 所有开发与部署都走 DeviceManager 登记、执行和收工更新，不要绕过。
- **文档：** 每次收工按下一节更新相应文件。分阶段路线图见 `docs/opb/HANDOFF.md`。

## 文档维护约定

| 文件 | 约定 |
| --- | --- |
| `OPB.md` | 定制版总览、兼容性合同、与上游差异、构建 / 部署与流程的入口。合同、流程或已完成差异发生变化时更新。不要把进行中的进度写在这里。 |
| `CHANGELOG-OPB.md` | 只记录 OPB 自身变更，按日期倒序。每条注明相关提交（短 SHA）和 DeviceManager 任务号。不要改上游自带的 changelog（例如 `apps/docs/content/docs/changelog/`）。 |
| `docs/opb/HANDOFF.md` | 活文档。每次收工更新：当前进度、进行中的分支、未完成项、待用户决定项、已知问题、下一步和分阶段路线图。供中枢和 OPB 内的维护 Bot 接手。主机用主机名（sts-mbp、sts-mac），不必写 machineId。 |
| `docs/opb/design/YYYY-MM-DD-<主题>.md` | 每个功能一份，文件名带日期。必须包含：背景、方案、接口影响、取舍、验证、回滚。不要改写已发布的设计历史；后续变化另开文档或在文末追加。未实施的方案须标明状态。 |
