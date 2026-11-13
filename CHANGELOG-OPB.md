# OpenPigeonBot 变更记录

只记录本 fork 的 OPB 变更。不要改上游 changelog。日期按倒序，时区为 Asia/Shanghai。

## 2026-11-13

- 在 sts-mac 部署 OPB 0.1.154。DeviceManager 任务 `TASK-20261113-task-f1028a6e`，发布 `1d4a6a9`，操作记录 `OP-20261113-result-4fadc16bac81`。
- 部署后 `/api/health` 正常。引擎登录和实际对话未验证。
- 安装与运行要点（详见 `docs/opb/design/2026-11-13-sts-mac-deployment.md`）：安装路径为 `/Applications/OpenPigeonBot.app`；数据目录 `~/.openmausbot`；只监听 `127.0.0.1:8799` / `127.0.0.1:8800`（8810 / 8811 未监听）；当时用 `-c.mac.identity=null` 构建，未公证，构建刚结束时 Identifier=`Electron`，在 `~/.local/opb-stage` 上执行 `codesign --deep --force --sign -` 后 Identifier 变为 `com.stephenfeng.openpigeonbot`（`Signature=adhoc`，`TeamIdentifier=not set`），`spctl --assess` 仍拒绝，去隔离后可以启动。去隔离的完整 `xattr` 命令行在证据中未留下，仍待确认。
- 对照 electron-builder 26.15.3 源码和当前树核实后，把签名修正方案写入 `docs/opb/design/2026-11-13-signing-fix.md`（状态：未实施）：改用 `-c.mac.identity="-"`，保留 entitlements 中已有的 `disable-library-validation` 与 `allow-jit`，不再使用 `codesign --deep`，去隔离改用 `xattr`，先修签名再授 FDA / 辅助功能。验收标为待在 sts-mbp 上实测。
- 核实 Move from Grok Bot 已在当前 `opb/branding` 树中；手动入口与自动提示条件已记入 `OPB.md` 和 `HANDOFF.md`。#5125 只是 Settings 里的版式修复，不是该功能的引入提交。
- 2026-11-13 11:44（Asia/Shanghai）在 sts-mac / sts-mbp 上只读核对了安装路径、签名、端口、`/api/health` 与 `/api/brand` 正文、暂存目录体积，以及上游仍为 `f509c24bb`、相对 `2d74ba961` 超前 338。结果已写入上述文档。
- 文档补齐（`opb/docs` 首个提交 `311b3513a`，完整 SHA：`311b3513a3b7a45904183ba853e4f43869da3acf`）：修订 `OPB.md`，新增并更新 `CHANGELOG-OPB.md`、`docs/opb/HANDOFF.md`、`docs/opb/design/2026-11-12-branding.md`、`docs/opb/design/2026-11-13-sts-mac-deployment.md`、`docs/opb/design/2026-11-13-signing-fix.md`。本条 SHA 由本分支第二个提交回填。

## 2026-11-12

- 建立公开 fork `5tephenFeng/OpenPigeonBot`。`main` 对齐上游 `milind-soni/OpenMausBot` 的 `2d74ba9616270b4a1df1ebfa4a912b55bd5cb4ea`（Window frame: round card corners, and the frame in the sidebar's colour in every skin (#4641)）。
- 在 `opb/branding` 完成品牌化（相对 `main` 超前 6、落后 0），任务号 `TASK-20261112-task-6f02e47b`。六个提交：
  - `3fd3f4f4b570` — OPB: name the app PigeonBot / OpenPigeonBot in the UI
  - `470eb4bed932` — OPB: package the desktop app as OpenPigeonBot
  - `8c7b654bf7f9` — OPB: fork notes, README banner and agent pointer
  - `40745da2b9fa` — OPB: tray tooltip tests follow the OpenPigeonBot name
  - `a5d117ba50a6` — OPB: component test expectations follow the PigeonBot / OpenPigeonBot names
  - `0ec29a9fb043` — OPB: last two test expectations for the fork docs link and the PigeonBot updates label
- 显示名为 PigeonBot / OpenPigeonBot，Bundle ID 为 `com.stephenfeng.openpigeonbot`，并新增当时版本的 `OPB.md`。HTTP / WebSocket 等兼容性合同范围内的接口未改。
- 检查结果（当时记录，本次文档工作未复跑）：类型检查、lint、Electron 检查、i18n 通过；Electron 测试 2038 项全过；前端测试 7 个失败，上游在 macOS 上同样失败；全量 vitest 未跑完。
