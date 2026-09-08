---
title: iOS开发证书与Provisioning Profile创建指南：从CSR到真机签名
date: 2026-09-08
tags:
  - iOS
  - 代码签名
  - Provisioning-Profile
  - Apple-Developer
  - Fastlane
---

# iOS开发证书与Provisioning Profile创建指南：从CSR到真机签名

第一次给 iOS App 配真机签名时，Apple Developer 后台里会同时出现 Certificate、Identifier、Device、Profile、Capability 等好几个入口，很容易让人不知道先做什么、后做什么。

其实这件事可以拆成一条清晰的链路：先让 Apple 认可你的 App 身份，再创建可用于签名的证书，最后用 Provisioning Profile 把“App、证书、能力和设备”组合在一起。本文以**手动签名**为主，带你从创建 CSR（Certificate Signing Request，证书签名请求）开始，完成 Development Certificate 和 Development Provisioning Profile 的创建，并说明它和 Xcode 自动签名、Fastlane match 的关系。

本文讨论的是 Apple Developer 的代码签名体系，不涉及 App Store Connect API Key。后者用于上传 TestFlight、管理商店元数据和提交审核，和“能不能把 App 签名安装到设备”是两套不同的权限系统。

## 一、先建立正确的概念

在动手前，先把几个名词分清。很多签名问题并不是操作错了，而是把它们当成同一种文件。

| 名称 | 解决的问题 | 可以简单理解为 |
|---|---|---|
| Identifier / App ID | 这个 App 是谁 | App 的身份证，通常对应 Bundle ID |
| Certificate | 谁可以代表团队签名 | 团队认可的签名身份 |
| Private Key | 你是否真的拥有签名能力 | 证书背后的“钥匙”，必须严格保护 |
| Device | 哪些真机可安装开发包 | 开发测试的设备白名单 |
| Provisioning Profile | 这个 App 用哪张证书、允许哪些能力和设备 | 把前面几项装订成一份通行证 |
| Entitlements | App 实际请求的能力 | 如推送、App Groups、iCloud、Sign in with Apple |

可以把 Development 签名过程想成下面这样：

```text
明确 Bundle ID 和能力
        ↓
创建 CSR → Apple 签发开发证书 → 本机保留私钥
        ↓
登记测试设备（真机开发时）
        ↓
创建 Development Profile
        ↓
Profile 绑定 App ID + 开发证书 + 设备 + 能力
        ↓
Xcode 或 Fastlane 使用它签名并安装 App
```

其中最重要的一点是：**证书文件本身不等于签名能力。** 如果某台 Mac 只导入了 `.cer`，却没有创建 CSR 时生成的私钥，codesign 仍然无法用它签名。

## 二、开始前需要准备什么

开始之前，请确认你具备以下条件：

- 已加入 Apple Developer Program，并在团队中拥有创建证书/Profile 的权限；
- 项目已经确定主 App 的 Bundle ID，例如 `com.example.myapp`；
- 如果项目有 Widget、Share Extension、Notification Service Extension 等 Target，已经列出它们各自的 Bundle ID；
- 使用一台自己的 Mac 创建 CSR，且能访问“钥匙串访问”；
- 如果要真机开发，准备好测试 iPhone/iPad 的 UDID；
- 知道项目需要哪些 Capability，例如 Push Notifications、App Groups、iCloud 等。

Apple 当前的手动 Development Profile 页面要求 Account Holder 或 Admin 权限。如果你只是普通团队成员，通常应让管理员创建 Profile，或者直接使用 Xcode 的 Automatically manage signing，由 Xcode 代为管理开发证书、设备和 Profile。

如果项目暂时只跑 Simulator，严格来说不需要 Development Profile；Simulator App 不使用 Apple Developer 的真机签名链路。不过，只要要连接真机、导出 Development 包，或之后准备接入 Fastlane，就建议尽早把签名资产整理好。

### 主 App 和 Extension 要单独看待

一个常见误区是“我的 App 已经有 Bundle ID，所以 Widget 也应该自动可以签”。实际上，Widget 等 Extension 是独立的签名 Target，通常拥有不同 Bundle ID，例如：

```text
主 App：com.example.myapp
Widget：com.example.myapp.Widget
```

主 App 和 Widget 可能共享某些能力，比如 App Groups；但在 Apple Developer 后台中，它们仍要分别有对应的 Identifier，并在 Profile 和自动化签名配置中被完整覆盖。

## 三、第一步：创建 App ID 和配置能力

创建证书前，先确认 Apple Developer 后台已有正确的 Identifier。

进入 [Apple Developer](https://developer.apple.com/account/) 后，依次打开：

```text
Certificates, Identifiers & Profiles
→ Identifiers
→ 点击 +
→ App IDs
```

选择 App 类型后，填写描述名称和 Bundle ID。

### Explicit App ID 和 Wildcard App ID 怎么选

通常应选择 **Explicit App ID（明确 App ID）**，也就是完整写出 Bundle ID：

```text
com.example.myapp
```

Wildcard App ID 形如：

```text
com.example.*
```

它在非常简单的老项目中可能有用，但对现代 iOS 项目通常不够。只要使用推送通知、App Groups、iCloud、Sign in with Apple、Apple Pay 等能力，就需要明确 App ID。实际开发中，为了降低后续迁移成本，即使暂时没开这些能力，也更推荐使用 Explicit App ID。

### Capability 必须和工程保持一致

如果 Xcode 的 Signing & Capabilities 中启用了 App Groups、iCloud 或 Push Notifications，Apple Developer 后台对应 Identifier 也必须开启这些能力。否则经常会出现：

```text
Provisioning profile doesn't include the ... entitlement
```

或者：

```text
Profile doesn't match the entitlements file's values
```

遇到这类错误时，不要急着反复删除 Profile。应先对比三处是否一致：

1. Xcode Target 的 Signing & Capabilities；
2. `.entitlements` 文件中的值；
3. Apple Developer 中对应 Identifier 已启用的 Capability。

能力发生变化后，旧 Profile 未必包含新 entitlement，通常需要重新生成或更新 Profile。

## 四、第二步：创建 CSR

CSR 是发给 Apple 的“证书申请”。它包含公钥和申请信息；与之配对的私钥会保留在创建 CSR 的 Mac 的 Keychain 中。CSR 本身不是私钥，通常不属于最高敏感级别，但也没有必要提交到业务 Git 仓库。

### 1. 打开钥匙串访问

在 macOS 中打开：

```text
应用程序 → 实用工具 → 钥匙串访问
```

或者直接用 Spotlight 搜索“钥匙串访问”。

### 2. 发起证书请求

在顶部菜单选择：

```text
钥匙串访问 → 证书助手 → 向证书颁发机构请求证书
```

> 界面示意：在 Certificate Assistant 中填写开发者邮箱和一个便于识别的通用名称；本文不展示含邮箱或姓名的原始截图。

在弹出的窗口中：

1. 在“用户电子邮件地址”填写你的开发者邮箱；
2. 在“通用名称”填写方便识别的名称，例如 `MyApp Development Key 2026`；
3. “CA 电子邮件地址”保持为空；
4. 选择“保存到磁盘”；
5. 点击“继续”，选择一个安全的位置保存。

生成的文件通常名为：

```text
CertificateSigningRequest.certSigningRequest
```

![[CertificateSigningRequest.png|368]]

这个 CSR 可以上传到 Apple Developer 后台，但它本身不能用于签名，也不需要提交进 Git。

### 3. 为什么一定要在自己的 Mac 上生成 CSR

创建 CSR 的同时，钥匙串访问会生成一对密钥：公钥被放进 CSR，私钥保留在当前 Mac。Apple 后台下载的证书会与这个私钥匹配。

因此：

- 证书应由负责签名的人或受控的签名机器创建；
- CSR 可以上传给 Apple 或证书颁发机构，但不要把真正的私钥、`.p12` 或 Keychain 导出文件一并发出去；
- 更不要只备份 `.cer` 而忘了备份含私钥的签名身份；
- 如果换电脑，需要从旧机器导出带私钥的 `.p12`，或用 Fastlane match 统一恢复签名资产。

## 五、第三步：在 Apple Developer 创建 Development Certificate

回到 Apple Developer 后台，依次进入：

```text
Certificates, Identifiers & Profiles
→ Certificates
→ 点击 +
```

在证书类型中选择适合开发用途的开发证书。Apple 后台的命名会随时间调整，但原则不变：**真机调试用 Development；TestFlight/App Store 用 Distribution。** 如果你使用 Xcode 自动签名，Xcode 可能会代为创建或管理开发证书，这时不必重复手动创建。

接下来上传刚才创建的 `.certSigningRequest` 文件。

![[choose_file.png|492]]

点击继续后，Apple 会生成证书。下载生成的 `.cer` 文件：

> 生成成功后，页面会显示证书类型、创建者和到期时间。为避免泄露个人邮箱和姓名，本文不展示原始结果页截图。

下载后的文件类似：

![[download_development_cer.png|511]]

双击 `.cer` 文件，系统会将证书导入“钥匙串访问”。正常情况下，你应在“我的证书”分类中看到证书，并能展开看到下面的私钥。

```text
Apple Development: Your Name (TEAMID)
  └── private key
```

如果能看到证书，却不能展开出 private key，说明当前 Mac 没有对应私钥。最常见的原因是 CSR 在另一台 Mac 上生成，或曾经清理过 Keychain。此时不要新建一堆重复证书；应回到生成 CSR 的机器导出带私钥的身份，或按团队既定的 match 流程恢复。

### Development 与 Distribution 证书的区别

| 类型 | 用途 | 能否上传 TestFlight/App Store |
|---|---|---|
| Development Certificate | Xcode 真机运行、Development 包 | 不可以 |
| Distribution Certificate | Ad Hoc、TestFlight、App Store Archive | 可以，具体分发方式由 Profile/导出方式决定 |

开发证书和发布证书可以由同一个团队管理，但不是同一种证书。不要拿 Development Certificate 去导出 App Store IPA，也不要因为“Xcode 能跑”就认为发布签名已经准备好。

## 六、第四步：登记测试设备

如果你要创建传统的 Development Profile 或 Ad Hoc Profile，需要把测试设备登记到 Apple Developer 后台。

进入：

```text
Certificates, Identifiers & Profiles
→ Devices
→ 点击 +
```

填写设备名称和 UDID。获取 UDID 的常见方式：

- 在 Finder 中连接 iPhone/iPad，点击设备信息区的序列号字段，直到显示 UDID，然后复制；
- 在 Xcode 的 Devices and Simulators 窗口查看；
- 使用受信任的设备管理流程导出设备标识。

UDID 是设备唯一标识，应像普通设备资产信息一样管理。设备更换、退出测试范围或团队成员变化时，及时从测试清单中移除或停止使用对应 Profile。

设备登记完成后，已经下载过的旧 Profile 不会自动获得新设备。新增设备后，需要重新生成并重新下载 Development Profile，或者让 Xcode 自动签名重新同步。

## 七、第五步：创建 Development Provisioning Profile

现在可以创建 Profile。进入：

```text
Certificates, Identifiers & Profiles
→ Profiles
→ 点击 +
```

选择开发用途的 Profile 类型，然后按页面步骤完成：

1. 选择主 App 对应的 Explicit App ID；
2. 选择刚创建的 Development Certificate；
3. 选择允许安装的测试设备；
4. 为 Profile 起一个清晰名称，例如 `MyApp Development 2026`；
5. 生成并下载 `.mobileprovision` 文件；
6. 双击该文件导入 Xcode，或让 Xcode/fastlane 自动安装。

> 在“Select an App ID”步骤中，选择当前 Target 对应的 Explicit App ID。请勿在公开文章中展示真实 Team ID 或 Bundle ID。

对于含 Widget 的项目，主 App 和 Widget 通常都需要对应的 Profile。创建时要反复确认当前 App ID 属于哪个 Target，不要把主 App 的 Profile 误配给 Extension。

### Profile 里的内容到底是什么

一个 Development Profile 大致可以理解为以下集合：

```text
明确的 App ID
  + 可用的 Capability / Entitlements
  + 一张或多张开发证书
  + 一组已登记的测试设备
  + 有效期
```

任意一项不匹配都可能导致安装或签名失败。例如：新增测试机但 Profile 没重新生成、新增 App Group 但 Profile 还是旧版本、选错了证书、Widget 没有对应 Profile。

## 八、Offline support 7 day validity 是什么

创建某些开发或 Ad Hoc Profile 时，Apple Developer 后台可能会显示 **Offline support (7 day validity)** 选项。

> 该页面会显示 Offline support 选项；本文省略包含真实证书名称的原始截图。

它的核心取舍可以这样理解：

| 选项 | 适用情况 | 主要代价 |
|---|---|---|
| No（通常默认） | 日常开发、普通真机测试、Fastlane match | 设备需按 Apple 的常规验证机制使用网络环境 |
| Yes | 设备长期处于隔离网络、无法访问 Apple 验证服务的特殊测试 | Profile 有效期只有 7 天，必须频繁重新生成和安装 |

### 日常开发为什么建议选 No

对绝大多数项目，选择默认的 **No** 更合适：

- Profile 有效期按常规规则管理，通常不需要每周重建；
- 更适合持续开发、多人协作和自动化打包；
- Fastlane match 和 Xcode 的常规签名流程也更容易维护；
- 测试设备只要能在正常网络条件下完成 Apple 的验证即可。

### 什么时候才考虑 Yes

只有当测试设备确实处于隔离网络，长期无法访问 Apple 的验证服务，并且你能接受维护成本时，才考虑启用。典型场景包括特殊内网、离线现场演示或受严格网络限制的测试环境。

代价是明确的：Profile 只有 7 天有效，失效后 App 可能无法继续启动；你需要重新生成 Profile、重新签名、重新安装。这不是“更高级的模式”，而是为了离线场景做的有期限交换。

> Profile 生成页通常会显示 Profile 名称、App ID 和到期时间。公开记录时应使用占位示例，不展示真实项目标识。

如果你正在使用 Fastlane match，优先保持默认的常规 Profile。把频繁过期的 7 天离线 Profile 纳入自动化资产，通常只会增加维护成本和故障概率。

## 九、把证书和 Profile 安装到 Xcode

手动流程中：

- 双击 `.cer`，导入钥匙串；
- 双击 `.mobileprovision`，让 Xcode 识别并安装；
- 在 Xcode 选择主 App Target → Signing & Capabilities；
- 设置正确 Team；
- 根据团队策略选择 Automatically manage signing 或手动指定 Profile；
- 对每一个 Extension Target 重复检查 Team、Bundle ID 与 Capability。

### 自动签名和手动签名怎么选

| 方式 | 适合什么情况 | 注意事项 |
|---|---|---|
| Automatically manage signing | 个人开发、早期项目、简单真机调试 | Xcode 会尝试创建/更新资产；多人和 CI 下可控性较弱 |
| Manual signing | 已接入 Fastlane match、多个 Target、CI、正式发布 | 需要明确管理证书/Profile，但行为可复现 |

两种方式没有绝对好坏。项目刚起步时自动签名能减少操作；一旦需要多人协作、Widget、多环境或 CI，建议逐步迁移到受控的签名管理方案。最怕的不是选哪一种，而是本机自动签名、CI 手动签名、团队成员各自一套证书混着用。

### 当前项目的实际对应关系

当前工程采用 Manual signing，并按 SDK 和构建配置区分签名身份：真机 Debug 使用 Apple Development，Release 使用 Apple Distribution；主 App 和 Widget 分别指定对应的 `match Development ...` 或 `match AppStore ...` Profile。这里的省略号代表脱敏后的完整 Bundle ID。

这说明手动 Profile 不是“下载一次就结束”：工程的 Target、Bundle ID、签名身份、Profile specifier 和 entitlements 必须长期保持一致。只要新增 Widget、修改 App Groups 或切换分发方式，就要一起检查这些字段。

## 十、检查 Profile 是否真的包含预期内容

`.mobileprovision` 是签名配置文件，可以在本机查看它的有效期、UUID、App ID、Team 标识、设备列表和 Entitlements。不要直接用文本编辑器打开，而是让 macOS 的 `security` 工具解码：

```sh
security cms -D -i MyApp_Development.mobileprovision > /tmp/profile.plist

# 查看关键字段
/usr/libexec/PlistBuddy -c 'Print :Name' /tmp/profile.plist
/usr/libexec/PlistBuddy -c 'Print :ExpirationDate' /tmp/profile.plist
/usr/libexec/PlistBuddy -c 'Print :Entitlements:application-identifier' /tmp/profile.plist
/usr/libexec/PlistBuddy -c 'Print :ProvisionedDevices' /tmp/profile.plist
```

重点核对：

- `application-identifier` 是否对应当前 Team 和 Bundle ID；
- `com.apple.developer.team-identifier` 是否属于预期团队；
- `Entitlements` 是否包含项目实际使用的能力；
- `ProvisionedDevices` 是否包含本次测试设备；
- `ExpirationDate` 是否已经过期。

临时解码文件只用于排查，检查完成后可以删除。不要把包含团队标识、设备 UDID 或能力信息的完整 Profile 上传到公开 issue。

## 十一、如何安全备份和迁移签名身份

如果暂时不用 Fastlane match，至少要知道如何从旧 Mac 迁移签名能力。

在“钥匙串访问 → 我的证书”中，找到能展开显示 private key 的开发或发布证书，导出为 `.p12`，并设置强密码。这个 `.p12` 同时携带证书和私钥，才可以在新 Mac 上继续签名。

安全规则：

- `.p12`、私钥、`.mobileprovision` 不要提交到业务 Git 仓库；
- 不要用聊天工具、邮件或公开网盘随意传播私钥；
- 为 `.p12` 设置独立强密码；
- 只在受信任设备上导入；
- 团队协作时优先使用 Fastlane match 等统一机制，避免每个人手里散落一份证书；
- 证书疑似泄露时应立即在 Apple Developer 后台撤销并重新签发，而不是只删除本地文件。

## 十二、如何与 Fastlane match 衔接

当项目从“一个人一台电脑”发展到多人协作、需要换电脑、需要 CI 时，手动导入证书和 Profile 会越来越脆弱。这时可以使用 Fastlane match。

match 的做法是：把证书、私钥和 Profile 存放在**独立的私有、加密仓库**中，项目仓库只保存规则。

```text
业务仓库
  └── Fastfile：声明需要的 Bundle ID 和分发类型

私有 match 仓库
  └── 加密保存证书、私钥、Profile

开发机 / CI Runner
  └── match 下载、解密、导入 Keychain，再执行签名
```

脱敏示例：

```ruby
APP_IDENTIFIERS = [
  "com.example.myapp",
  "com.example.myapp.Widget"
].freeze

lane :install_development_signing do
  match(
    type: "development",
    app_identifier: APP_IDENTIFIERS,
    git_url: "git@github.com:example/ios-certificates.git",
    readonly: true
  )
end
```

日常开发和 CI 建议用：

```ruby
readonly: true
```

这表示当前机器只消费已有签名材料，不会擅自创建、修改或吊销团队证书。证书资产的创建、迁移、更新应在受控的本机环境中完成，再同步到 match 仓库。

如果想进一步了解 match、Archive、TestFlight、版本号与发布流程，可以阅读同目录文章：[iOS项目如何接入Fastlane](iOS项目如何接入Fastlane.md)。

## 十三、常见问题与排查顺序

### 1. 找不到可用的签名身份

**现象**：Xcode 或 `codesign` 报找不到 Development/Distribution certificate。

**优先检查**：

1. 钥匙串访问“我的证书”中是否有证书；
2. 证书下面能否展开看到 private key；
3. 证书是否属于当前选择的 Team；
4. 证书是否过期或已被撤销；
5. CI 是否创建、解锁并使用了正确 Keychain。

只有 `.cer` 没有私钥，不能解决签名问题。

### 2. Profile 不包含所需 entitlement

**现象**：

```text
Provisioning profile doesn't include the ... entitlement
```

**处理顺序**：检查 Identifier Capability → 检查 Xcode 的 Signing & Capabilities → 检查 `.entitlements` → 更新 Profile。不要只在 Xcode 里反复切换 Team。

### 3. 真机无法安装或提示设备不在 Profile 内

**原因**：设备 UDID 没登记，或者当前 Development/Ad Hoc Profile 创建时没有勾选这台设备。

**解决**：登记设备后重新生成 Profile，并在 Xcode 或 match 中更新安装的 Profile。

### 4. Widget 能编译，Archive 时却签名失败

**原因**：Widget 的 Identifier 或 Profile 漏配；也可能 App Group 等共享能力在主 App 和 Widget 中不一致。

**解决**：逐个检查 Target 的 Bundle ID、Team、Capability 和 Provisioning Profile。自动化中也要让 `app_identifier` 包含所有 Target。

### 5. App 一周后无法打开

**原因**：可能使用了 7 天 Offline support Profile；也可能使用了个人免费开发者账户或其他有期限的签名方式。不能只凭“7 天失效”就断言原因，先检查 Profile 类型和签名身份。

**解决**：在 Apple Developer 后台查看该 Profile 的配置与有效期；日常开发改用常规 Development Profile，并重新签名安装。

### 6. 新机器导入证书后还是不能签名

**原因**：只导入了 `.cer`，没有私钥；或者导入的私钥与证书不匹配。

**解决**：从旧机器导出带私钥的 `.p12`，或通过 match 恢复完整签名身份；不要在新机器上盲目多次创建证书。

### 7. Fastlane 或 CI 里 match 拉取失败

**原因**：证书仓库 SSH 权限、Deploy Key、`known_hosts`、`MATCH_PASSWORD` 或临时 Keychain 配置有问题。

**解决**：先检查仓库读取权限和密钥文件 `600` 权限；CI 中明确设置 `GIT_SSH_COMMAND`，并使用 `match readonly`。不要把证书仓库改公开来“解决”权限问题。

## 十四、最后的检查清单

在点击 Build 或 Archive 前，可以快速过一遍：

```text
[ ] 主 App 与所有 Extension 都有正确的 Explicit App ID
[ ] Capability 与 Xcode entitlements 一致
[ ] 本机证书在“我的证书”中能展开看到 private key
[ ] 已登记本次测试的真机 UDID（Development/Ad Hoc 场景）
[ ] Profile 的类型符合目标：Development / Ad Hoc / App Store
[ ] Profile 覆盖了正确的 App ID、证书、设备与能力
[ ] 未误用 7 天 Offline support Profile
[ ] 私钥、.p12、.p8 和 Profile 没有提交进业务仓库
[ ] 有 Widget/Extension 时，所有 Target 的签名配置均已检查
[ ] 需要 CI 时，签名资产已通过 match 或等价的受控方案管理
```

## 十五、与 Apple 官方流程的对应关系

本文的手动步骤与 Apple 官方 Development Profile 流程一一对应：先准备 App ID、开发证书和已登记设备，再在 Profiles 中选择 Development 类型、App ID、证书和设备，填写 Profile 名称后 Generate、Download。使用自动签名时，Xcode 会管理开发 Profile，不必重复执行这些手动步骤。

建议在 Apple 后台改版后，以官方页面的当前字段名称为准：

- [Create a development provisioning profile](https://developer.apple.com/help/account/provisioning-profiles/create-a-development-provisioning-profile/)
- [Register an App ID](https://developer.apple.com/help/account/identifiers/register-an-app-id/)
- [Register a device](https://developer.apple.com/help/account/devices/register-a-single-device/)

Apple 后台界面和证书名称会随 Xcode、平台政策调整，但“App ID + 证书 + 设备 + 能力 + Profile”这条关系不会变。

## 结语

iOS 签名看起来复杂，是因为它同时要回答四个问题：这个 App 是谁、谁可以签名、哪些能力被允许、包可以安装到哪里。把 App ID、证书与私钥、设备、Profile 这几层关系理清后，绝大多数问题都有固定的排查路径。

对个人项目来说，手动创建一次 Development Certificate 和 Profile，是理解 Apple 签名体系最好的起点；对多人协作和持续集成项目来说，下一步则是把这些资产收敛到 Fastlane match 等可控流程中。无论采用哪种方式，最重要的原则都一样：签名资产必须可追溯、私钥必须安全、每个 Target 都必须被完整覆盖。
