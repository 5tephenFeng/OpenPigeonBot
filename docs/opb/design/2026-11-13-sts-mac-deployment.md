# sts-mac 部署（0.1.154）

日期：2026-11-13  
部署目标：sts-mac  
构建机：sts-mbp  
版本：0.1.154  
任务：`TASK-20261113-task-f1028a6e`  
DeviceManager 发布：`1d4a6a9`  
操作记录：`OP-20261113-result-4fadc16bac81`  
现场复核：2026-11-13 11:44（Asia/Shanghai）

## 背景

品牌化完成后，需要把可运行的桌面包装到日常使用的机器 sts-mac 上，供后续验证引擎、对话和远程访问。sts-mac 只做部署目标，不在上面做日常开发。构建在 sts-mbp 完成。

## 方案

当时实际执行的流程（已发生，不是今后的标准做法）：

- 在 sts-mbp 上用 `-c.mac.identity=null` 打出未公证的 arm64 `dir` 包，版本 0.1.154。该选项在 electron-builder 26.15.3 中会完全跳过签名，于是留下链接器默认签名 Identifier=`Electron`（证据 `stage-before.txt`：`flags=adhoc,linker-signed`）。
- 从 sts-mbp 的 `~/SyncthingSplit/AI/OpenPigeonBot/release/mac-arm64/OpenPigeonBot.app` rsync 到 sts-mac 的 `~/.local/opb-stage/`（部署记录当时约 795MB；2026-11-13 11:44 现场 `du -sh` 为 773M，目录内确有 `OpenPigeonBot.app`）。
- 在暂存副本上做 ad-hoc 签名，操作记录中的命令是：`codesign --deep --force --sign -`，随后 `codesign --verify --deep --strict` 通过（证据 `codesign.txt` 仅留下 `replacing existing signature`）。补签后 Identifier 变为 `com.stephenfeng.openpigeonbot`（不再是 `Electron`），`Signature=adhoc`，`TeamIdentifier=not set`。
- 操作记录写了「清除隔离属性」，但 `~/.local/evidence/TASK-20261113-task-f1028a6e/` 与操作记录都没有留下完整的 `xattr` 命令行，具体参数仍待确认。`spctl --assess` 对暂存副本仍为 rejected；去隔离后用 `open` 启动成功。
- 用 `ditto` 安装到 `/Applications/OpenPigeonBot.app`。路径、版本与 Bundle ID 已现场核对。
- 替换已安装版本必须先得到用户同意。
- 数据目录使用已有约定 `~/.openmausbot`，不另建目录。部署记录还提到 Electron 用户目录在 `~/Library/Application Support/OpenPigeonBot`。
- 进程只监听 `127.0.0.1:8799` 和 `127.0.0.1:8800`，不对外网或局域网暴露这两个端口。远程访问只走应用内的 Tailscale。
- 未配置开机自启、主机控制、完全磁盘访问（FDA）。2026-11-13 现场检查：用户与系统 LaunchAgents（以及 LaunchDaemons 文件名）中没有与 OPB 有关的项。授 FDA / 辅助功能必须排在签名修正并验收之后，见 `docs/opb/design/2026-11-13-signing-fix.md`（未实施）。

## 接口影响

无代码接口变更。本次是把已有的 `opb/branding` 构建结果装到 sts-mac。

部署后的运行约定：

- `/api/health` 现场正文（2026-11-13 11:44 Asia/Shanghai）：`{"app":"openmausbot","pid":41342,"static":true,"capabilities":{"guardedMessages":1,"guardedRequests":1,"guardedFullAccess":1,"guardedOnBehalfOf":1},"box":"off"}`。无令牌等敏感字段。`app` 符合合同。
- `/api/brand` 现场正文：`{"brand":{"name":"OpenMausBot"},"source":"default","file":"/Users/stephen-feng/.openmausbot/brand.json"}`。
- 仍只应使用 `127.0.0.1` 上的默认 HTTP/WS 端口；不要为了远程访问而改监听地址。
- 数据仍写在 `~/.openmausbot`。

## 取舍

- 当时为了先能启动，采用了跳过签名再 `--deep` 补签并去隔离的办法，而不是当时就改用 `-c.mac.identity="-"`。补签后 Identifier 已改成 bundle id，但 `--deep` 可能覆盖 Helper 的 entitlements，`spctl` 仍拒绝。后续应以签名方案稿为准，不要重复这条路径。
- 只监听本机回环地址，降低误暴露风险；远程访问依赖应用内 Tailscale（开通由中枢跟进，完成情况待确认）。
- 暂不打开开机自启、主机控制和 FDA，减少未经用户明确同意的系统权限；部分能力因此不可用。也不要在签名修正前补授这些权限。
- 不把构建产物放进 Syncthing。

## 验证

已现场核对（2026-11-13 11:44 Asia/Shanghai，以及部署证据）：

- 安装路径 `/Applications/OpenPigeonBot.app`；版本 0.1.154；`CFBundleIdentifier`=`com.stephenfeng.openpigeonbot`
- 已安装包 `codesign -dv --verbose=2`：`Identifier=com.stephenfeng.openpigeonbot`，`Signature=adhoc`，`TeamIdentifier=not set`。无 `com.apple.quarantine`，仅有 `com.apple.provenance`
- `/api/health` 与 `/api/brand` 正文见上
- 只监听 `127.0.0.1:8799` / `127.0.0.1:8800`（当前 pid 41342）；8810 / 8811 未监听（部署当时亦然）
- 去隔离后可以启动（`spctl` 仍拒绝）

未做 / 未通过：

- 引擎登录未验证
- 实际对话未验证
- 开机自启、主机控制、FDA 未配置，因而也未验证。部署检查里对 System Events 的 osascript 报过「不允许辅助访问」（-25211）
- Tailscale 远程访问未完成验证
- 未在本次工作中复跑类型检查或测试套件（测试结果仍以 2026-11-12 的品牌化记录为准）
- 计划中的 ad-hoc 构建（`-c.mac.identity="-"`）尚未在 sts-mbp 上实测
- 去隔离所用的完整 `xattr` 命令行仍待确认（证据中没有）

## 回滚

已知：

- 替换 sts-mac 上已安装的版本需要用户同意，回滚安装同样需要用户同意，不能擅自覆盖或删除 `/Applications` 里的应用。
- 部署操作记录中的卸装提示：退出应用后删除 `/Applications/OpenPigeonBot.app`。`~/.openmausbot` 与 `~/Library/Application Support/OpenPigeonBot` 仅在用户明确要求时删除。
- Git 上回退 `opb/branding` 不会自动卸下已经装好的 0.1.154。
- 数据在 `~/.openmausbot`。回滚安装时默认不应删除该目录，除非用户明确要求。

待补：

- 完整卸装步骤中，是否还要清理启动项、签名收据或其它残留，待补（当前未发现 LaunchAgents 自启项）。
- `~/.local/opb-stage` 在回滚时是否删除、如何删除，待补。清理该目录须用户授权。现场确认该目录现为 773M，内含暂存的 `OpenPigeonBot.app`。
- 若要回到「未安装 OPB」的状态，如何确认端口已释放、进程已退出，待补。
- 构建机 sts-mbp 上构建缓存 / `~/.local/opb-transfer` 是否需要一并清理，待补（清理本身还要用户授权）。现场确认 `opb-transfer` 为 729M。
