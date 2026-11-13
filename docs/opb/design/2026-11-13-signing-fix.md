# 签名修正（electron-builder ad-hoc）

日期：2026-11-13  
状态：**未实施**  
针对：0.1.154 构建留下 Identifier=`Electron`，并在 sts-mac 上用 `codesign --deep` 补签的问题  
相关：`docs/opb/design/2026-11-13-sts-mac-deployment.md`  
electron-builder 版本：本仓库为 26.15.3

## 背景

0.1.154 在 sts-mbp 上使用：

```sh
CSC_IDENTITY_AUTO_DISCOVERY=false pnpm exec electron-builder --mac dir --arm64 \
  -c.mac.identity=null --publish never
```

对照 electron-builder 26.15.3 的 `app-builder-lib`：`mac.identity` 为 `null` 时，`MacPackager` 走 `handleNullIdentity()`，日志为 `skipped macOS code signing`，签名步骤被完全跳过。于是包里留下 Electron 预编译二进制的默认签名，Identifier 为 `Electron`。sts-mac 证据 `stage-before.txt` 记录的正是这一状态：`Identifier=Electron`，`flags=adhoc,linker-signed`，`Signature=adhoc`，`TeamIdentifier=not set`。

随后在 `~/.local/opb-stage/OpenPigeonBot.app` 上执行了 `codesign --deep --force --sign -`，并用 `codesign --verify --deep --strict` 核验通过。补签后（`stage-after.txt` 以及 2026-11-13 11:44 对 `/Applications/OpenPigeonBot.app` 的现场核对）：`Identifier=com.stephenfeng.openpigeonbot`，`Signature=adhoc`，`TeamIdentifier=not set`。`spctl --assess` 仍拒绝；去掉隔离属性后可以启动。已安装包目前没有 `com.apple.quarantine`，仅有 `com.apple.provenance`。去隔离的完整 `xattr` 命令行未在证据中找到。

`electron-builder.yml` 和 `package.json` 的 `package:mac` 脚本本身没有写 `identity`。上述 `identity=null` 只出现在当时的手工构建命令里。

Apple 不建议用 `codesign --deep` 做签名（TN2206 / `codesign(1)`）：它会把同一套选项和 entitlements 套到嵌套代码上，可能覆盖 Chromium Helper 应使用的 inherit 权限。已安装包 Identifier 已经不是 `Electron`，并不能说明 `--deep` 是可接受的长期做法，也不能代替在构建阶段用 `identity="-"` 验收。

若在构建流水线仍会产出 Identifier=`Electron` 的包时，或在尚未用新流程验收前，先授予完全磁盘访问（FDA）或辅助功能，TCC 可能绑到旧标识；以后再修正签名，权限可能失效。当前部署尚未配置这两项，应保持这个顺序：先修签名并验收，再考虑授权。

## 方案

状态为未实施。在 sts-mbp 的构建命令中，把 `identity=null` 改为显式 ad-hoc（26.15.3 的 CLI 字段是 `-c.mac.identity`，不要写成较新文档里的 `-c.mac.sign.identity`）：

```sh
CSC_IDENTITY_AUTO_DISCOVERY=false pnpm exec electron-builder --mac dir --arm64 \
  -c.mac.identity="-" --publish never
```

同版本源码里，`qualifier === "-"` 时会构造 ad-hoc 身份 `Identity("-")` 并进入内置签名流程。`hardenedRuntime` 在本仓库的 `electron-builder.yml` 中为 `true`，因此 entitlements 必须继续包含：

- `com.apple.security.cs.disable-library-validation`
- `com.apple.security.cs.allow-jit`

这两项已经写在 `build/entitlements.mac.plist` 里；`electron-builder.yml` 的 `entitlements` 与 `entitlementsInherit` 都指向该文件。切换命令时必须保留，不要删掉或换一份缺这两项的 plist。仓库里没有单独的 `entitlements.mac.inherit.plist`。

构建完成后，在 **sts-mbp** 上验收（尚未实测，不要把下面的命令结果写成已通过）：

```sh
codesign -dv --verbose=4 release/mac-arm64/OpenPigeonBot.app 2>&1 | grep "Identifier="
```

产物目录和 `.app` 文件名以实际输出为准（按 `productName` 应为 `OpenPigeonBot.app`）。预期主程序 Identifier 在构建完成后、未再手工 `--deep` 的情况下就不是 `Electron`，而应是 `com.stephenfeng.openpigeonbot`。各 Helper 的 Identifier 不一定等于同一个 appId，以实测为准，不要事先写成「全部签成 appId」。

安装到 sts-mac 时：

- 不要再执行 `codesign --deep`（包括当时用过的 `codesign --deep --force --sign -`）。
- 去隔离用 `xattr`（常见写法是对应用包执行 `xattr -cr`；完整参数和安装路径待部署时确认）。当时实际使用的 `xattr` 命令行仍待确认。
- 替换 `/Applications/OpenPigeonBot.app` 必须先得到用户同意。
- 先完成签名验收，再授予 FDA 和辅助功能。

## 接口影响

无 HTTP / WebSocket、配对协议或数据目录变更。只改变 macOS 桌面包的签名方式。不要用上游身份签名或公证，也不要发布到上游 release 仓库。

## 取舍

- 用 electron-builder 内置 ad-hoc，而不是继续 `identity=null` 再手工 `--deep`。这样由打包器按文件分别签名，避免把主程序 entitlements 套到 Helper 上。
- 仍做 ad-hoc、仍不公证：`spctl` / Gatekeeper 可能继续拒绝，需要 `xattr` 去隔离后才能启动。这比构建阶段留下 Identifier=`Electron` 更接近正确的 bundle id，但还不是可分发的 Developer ID 签名。是否改为本机自签证书或公证，不在本方案范围内，须另议。
- 暂不在修正前配置 FDA / 辅助功能，以免 TCC 绑到旧标识。

## 验证

尚未实施，待在 sts-mbp 上实测：

- 使用 `-c.mac.identity="-"` 能否成功打出 arm64 `dir` 包
- 主程序在未经手工 `--deep` 时，`codesign -dv` 的 Identifier 是否已经不是 `Electron`
- 应用在去隔离（`xattr`）后能否启动；`spctl` 的实际结果
- Helper 是否仍能启动（entitlements 是否被正确分别应用）

实施并替换 sts-mac 上的安装之后，再验证引擎登录、对话（替换须用户同意）。8799 / 8800 / 8810 / 8811 的当前监听情况已在 2026-11-13 11:44 实测，见部署文档。

## 回滚

- 构建命令可改回 `-c.mac.identity=null`，但那会再次留下 Identifier=`Electron`，不是推荐的长期状态。
- 已安装到 sts-mac 的包不会因为改构建脚本而自动替换；换回或卸装需要用户同意。
- 不要为了回滚重新执行 `codesign --deep`，除非有单独的、写明风险的决策。
- 数据目录 `~/.openmausbot` 在回滚安装时默认保留。
