# OPB 交接

供中枢和 OPB 内的维护 Bot 接手。主机只用主机名（sts-mbp、sts-mac），不必写 machineId。不要在本文写入密钥或其它秘密。

最后更新：2026-11-13（Asia/Shanghai）。文档位于 `opb/docs` 分支，首个提交 `311b3513a`（完整 SHA：`311b3513a3b7a45904183ba853e4f43869da3acf`）。

## 当前进度

- 公开 fork `5tephenFeng/OpenPigeonBot` 已建立。`main` 对齐上游 `2d74ba961`（完整 SHA：`2d74ba9616270b4a1df1ebfa4a912b55bd5cb4ea`）。
- 品牌化在 `opb/branding`（尖端 `0ec29a9fb0436cb1a8e76e43c1bff6834a1831e1`），相对 `main` 超前 6、落后 0。显示名、Bundle ID、安装包名已改；兼容性合同范围内的接口未改。任务 `TASK-20261112-task-6f02e47b`。2026-11-13 在 sts-mbp 只读核对：仓库位于 `/Users/stephen-feng/SyncthingSplit/AI/OpenPigeonBot`，当前分支 `opb/branding` 跟踪 `origin/opb/branding`，工作区干净；远程有 `origin`（本 fork）和 `upstream`（`milind-soni/OpenMausBot`）。
- 已在 sts-mac 部署 0.1.154。任务 `TASK-20261113-task-f1028a6e`，DeviceManager 发布 `1d4a6a9`，操作记录 `OP-20261113-result-4fadc16bac81`。应用路径为 `/Applications/OpenPigeonBot.app`（`CFBundleIdentifier`=`com.stephenfeng.openpigeonbot`，版本 0.1.154）。数据目录 `~/.openmausbot`。当前监听 `127.0.0.1:8799` / `127.0.0.1:8800`（进程 OpenPigeonBot，pid 41342，核对于 2026-11-13 11:44 Asia/Shanghai）；8810 / 8811 未监听。`/api/health` 正文见部署文档，`app` 为 `openmausbot`。`/api/brand` 为 `source: "default"`，`brand.name` 为 `OpenMausBot`。构建使用 `-c.mac.identity=null`；暂存副本上 `codesign --deep --force --sign -` 之后，已安装包的 Identifier 已是 `com.stephenfeng.openpigeonbot`（`Signature=adhoc`，`TeamIdentifier=not set`），不再是 `Electron`。当前应用包没有 `com.apple.quarantine`，仅有 `com.apple.provenance`。开机自启、主机控制、FDA 未做（用户与系统 LaunchAgents 中无 OPB 相关项）。引擎登录和实际对话未验证。签名修正方案已写成设计稿，尚未实施。
- Move from Grok Bot **已在当前 `opb/branding` 树中**，不必先合并上游 338 个积压提交才能试用。
- 已在 `opb/docs` 写入文档（首个提交 `311b3513a`），含源码核实和 2026-11-13 现场只读核对。

## 进行中的分支

- `opb/branding`：已推送的品牌化分支，尖端 `0ec29a9fb`。
- `opb/docs`：本次文档补齐分支，从 `opb/branding`（`0ec29a9fb`）切出，首个提交 `311b3513a`。不要合并进 `main`。

不要在 `main` 上直接提交 OPB 改动。

## 未完成项

- 按 `docs/opb/design/2026-11-13-signing-fix.md` 修正构建签名（改用 `-c.mac.identity="-"`，不再 `codesign --deep`）。验收待在 sts-mbp 上实测。授 FDA / 辅助功能必须排在签名修正之后。当前已安装包虽然 Identifier 已是 bundle id，但那是 `--deep` 补签的结果，不能当作新流水线已验收。
- 跑完全量 vitest。
- 处理或明确记录前端测试中的 7 个失败（2026-11-12 的记录：上游在 macOS 上同样失败；本次未复跑）。
- 验证引擎登录和实际对话。
- Tailscale 访问由中枢跟进。策略是远程只走应用内的 Tailscale；开通是否完成待确认。
- 上游积压：2026-11-13 11:44（Asia/Shanghai）再次核对，`milind-soni/OpenMausBot` 的 `main` 仍比 fork 的 `main` 超前 338 个提交，尖端仍为 `f509c24bb`（完整 SHA：`f509c24bbbc92406f22bc1338e35c00e53e31025`，提交时间为 2026-11-13 03:28:10 Asia/Shanghai）。不要擅自合并。Move from Grok Bot 本身已在当前树中，不要把这 338 个提交当成启用该功能的前置；合并时再看上游后续对 Import 页的改动。
- 助手迁移：先用一个不重要的助手试导入。手动入口是 Settings → Import → Move from Grok Bot（未被品牌常量隐藏）。自动提示（`GROK_MOVE_NOTICE`）在 `brandStatus().source === "file"`、Team Cloud、托管成员或组织桌面时不会出现；当前默认 `source: "default"`，按代码仍可能弹出。导入完成后：先在原 Grok Bot 中关闭定时 Routine，再在 OPB 中启用对应 Routine；事件触发类从该 bot 的 `memory/grok-bot-event-routines.md` 核对并重绑。不要一次迁入全部「中枢、拾遗、生活」等助手。
- 排查仍直接读取 `brand().name` 做界面展示的调用点（`productName()` 在 `source !== "file"` 时已保持 PigeonBot，但 `applyBrand()` 会把 `brand().name` 换成服务端的 OpenMausBot）。
- 补改 `electron/system-tray.mjs` 里仍为 `"OpenMaus Bot"` 的托盘 tooltip（品牌化漏改）。
- 文档已在 `opb/docs` 提交（首个提交 `311b3513a`）；推送后把 GitHub 链接补进交接。不要直接写 `main`。

## 待用户决定

- locale 中硬编码的 MausBot / OpenMausBot 字样如何处理。实测（`opb/branding` 上 16 个 locale JSON，按子串 `MausBot` 统计，含 `OpenMausBot` 中的一段）：全部文件共 1699 处，其中 `OpenMausBot` 373 处；仅 `en.json` 为 203 处（191 个词条）。「约 80 处」不准确。已有 `{app}` 占位（仅 `en.json` 就有 227 个词条）会经 `t()` 换成 `productName()`，不必改这些 JSON。不要在 git 里批量硬改全部 locale 文件，也不要做无差别的 `replaceAll("MausBot", "PigeonBot")`。不应改名的对象包括：MausBot Cloud、the MausBot team（文中仍指上游维护者）、MausBot Relay、官方手机伴侣（如 “Update MausBot on your phone”）、以及 CLI 命令文本（如 ``openmausbot browser install``）。方案见品牌化设计文档，需用户拍板后再改。
- 是否更换图标。要换的话不能只换 3 个 `build/icon.*`，完整清单见 `docs/opb/design/2026-11-12-branding.md`。
- 是否建立 Gitee 私有镜像。
- 是否清理 sts-mbp 上的 `~/.local/opb-transfer`（2026-11-13 现场 `du -sh` 为 729M）和 sts-mac 上的 `~/.local/opb-stage`（现场 773M，内含 `OpenPigeonBot.app`，确是当时部署暂存副本）。这两项是运维清理，须用户授权后再做，不要自行删除。
- 数据目录保持 `~/.openmausbot`（合同已如此要求；若将来要改，必须先由用户评估）。

## 已知问题

- 未公证；0.1.154 构建刚结束时 Identifier=`Electron`；当时用 `codesign --deep --force --sign -` 补救后，已安装包 Identifier 已是 `com.stephenfeng.openpigeonbot`，但 `--deep` 仍可能覆盖 Helper 的 entitlements。`spctl` 仍拒绝，去掉隔离属性后可以启动。后续不应再使用 `--deep`。去隔离的完整 `xattr` 命令行未在证据中找到。
- `electron/system-tray.mjs` 第 23 行托盘 tooltip 仍是 `"OpenMaus Bot"`。
- 前端测试 7 个失败（见上，与上游在 macOS 上的表现一致，据 2026-11-12 记录）。
- 全量 vitest 尚未跑完，因此还有未覆盖的测试风险。
- 引擎登录与对话未验证，还不能把 sts-mac 部署视为功能已验收。
- 开机自启、主机控制、FDA 未配置，部分系统能力不可用。不要在签名修正前先授 FDA / 辅助功能。部署检查里对 System Events 的 osascript 报过「不允许辅助访问」（-25211）。

## 分阶段路线图

只列已核实或已约定的工作。运维清理见「待用户决定」，须授权后才做。

### P0

- 按签名方案稿，在 sts-mbp 上把构建改为 `-c.mac.identity="-"`，确认 entitlements 仍含 `disable-library-validation` 和 `allow-jit`，用 `codesign -dv` 验收 Identifier；不要再 `codesign --deep`。去隔离用 `xattr`。先完成签名验收，再考虑授 FDA / 辅助功能。
- 在 sts-mac 上验证引擎登录和实际对话（需用户配合的步骤不要擅自做）。替换已装版本须先得到用户同意。
- 端口已在 2026-11-13 11:44（Asia/Shanghai）实测：8799 / 8800 在听，8810 / 8811 未听。若之后进程重启，以当时 `lsof` 为准。
- 文档已在 `opb/docs` 提交（首个提交 `311b3513a`），由本任务推送到 `origin/opb/docs`。不要合并进 `main`。

### P1

- 分批评估并合并上游积压（2026-11-13 11:44 仍为 338 个提交，尖端 `f509c24bb`）。低风险且不影响合同与定制设计的可自行合并进 `opb/*`；有影响的交给用户评估。合并后跑检查，并确认前端 7 个失败没有新增回归。
- 用一个不重要的助手试做 Move from Grok Bot 迁移（入口与顺序见上文）。不要一开始就迁全部常用助手。

### P2

- 品牌文案精修：排查 `brand().name` 展示点；按用户决定处理 locale 硬编码字样（走 `{app}` / `productName()`，排除不应改名的对象）；补改托盘 tooltip。
- 若用户决定换图标，按完整资源清单一并替换，而不是只换 `build/icon.icns` / `.ico` / `.svg`。

## 下一步（与路线图一致，可由中枢调整）

1. P0：签名修正（sts-mbp 实测验收）→ 引擎与对话验证 → 文档入库。端口已实测，见上。
2. 中枢继续跟进 Tailscale 访问。
3. P1：分批看上游；用一个不重要的助手试迁移。
4. P2：文案与图标，待用户决定后再动。
5. 暂存目录清理等运维事项，等用户授权。

## 接手须知

- 开发在 sts-mbp，部署目标是 sts-mac。不要在 sts-mac 上做日常开发，也不要在用户电脑上直接跑未登记的命令。
- 所有开发、部署走 DeviceManager：开工先 sync、登记任务，收工更新文档和本 HANDOFF。
- 遵守 `OPB.md` 里的兼容性合同。尤其不要改 `/api/brand` 的默认 `brand.name: "OpenMausBot"`、`/api/health` 的 `app: "openmausbot"`、数据目录 `~/.openmausbot`、端口与配对协议。
- 低风险且不影响定制设计、不影响兼容性合同的上游更新，可自行合并进 `opb/*` 并部署；有影响的交给用户评估。
- 替换 sts-mac 上已安装的版本必须先得到用户同意。
- 不要用上游身份签名、公证或发到上游 release 仓库。不要再用 `codesign --deep` 补签。
- 不要在签名修正完成前授予 FDA 或辅助功能。
- 不要把密钥、token 或公司内部信息写进仓库文档。
- 上游 changelog 不要改；OPB 自己的记录只写在 `CHANGELOG-OPB.md`。
- 清理 `~/.local/opb-transfer` 或 `~/.local/opb-stage` 须用户授权，不要自行删除。
