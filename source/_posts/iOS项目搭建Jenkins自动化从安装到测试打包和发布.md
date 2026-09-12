---
title: iOS项目搭建Jenkins自动化：从安装到测试、打包和发布
date: 2026-09-12
tags:
  - iOS
  - Jenkins
  - CI/CD
  - 自动化测试
  - 打包发布
  - Fastlane
  - macOS
---

# iOS 项目搭建 Jenkins 自动化：从安装到测试、打包和发布

做 iOS 开发的人，大概都经历过这种时刻：本地测试明明全绿，提测前却要手动开构建页面、等半小时、再登录 App Store Connect 传包。更怕的是，某天负责发版的同事休假，整条发布链路只有他电脑上能跑通。

Jenkins 的价值不是“替你点几个按钮”，而是把 iOS 项目里那些重复、容易出错、又必须可追溯的动作——测试、签名、归档、上传——变成一台长期运行、可被团队共享的自动化服务器。你把规则写进 `Jenkinsfile` 和 `Fastfile`，以后无论是自己在本机运行，还是让 Jenkins 在专用 Mac 上运行，走的都是同一套流程。

本文以 ExampleApp 的真实搭建过程为例（**所有项目名、仓库地址、Bundle ID、Team ID、账号、凭证 ID 均已脱敏**，统一用占位符替代），完整说明如何在 macOS 上从零搭建一套 Jenkins CI/CD：安装、配置、创建 Job，直到跑通测试、打包和发布。文章假设你已经用 Fastlane 把签名、测试、归档、上传逻辑收敛进了 `Fastfile`（参考同目录文章：[iOS项目如何接入Fastlane](/2026/09/07/iOS项目如何接入Fastlane/)），Jenkins 只做“编排层”。

```text
开发者 push / 提 PR
        ↓
Jenkins 接收 Webhook，按 Jenkinsfile 编排
        ↓
准备环境（Ruby / Xcode / 临时 Keychain / 凭证注入）
        ↓
fastlane ios test → 归档 → （备用）上传 TestFlight / 提交审核
        ↓
回写 GitHub 状态、归档 IPA/dSYM/xcresult、清理密钥
```

## 一、先用一句话理解 Jenkins 在 iOS 交付里的位置

Jenkins 是一套可长期运行在 macOS 主机上的开源自动化服务器。它不替代 Xcode，也不替代 Fastlane 或 App Store Connect；它做的是把“什么时候跑、跑哪条流水线、用什么节点、注入哪些凭证、产物怎么归档”这些编排决策集中管理起来。

你可以把它理解为 iOS 项目的“自动化调度台”：

```text
GitHub 事件（push / PR / 手动触发）
        ↓
Jenkins 选择对应 Job / 流水线
        ↓
在指定 macOS 节点上准备环境、注入凭证
        ↓
调用 fastlane lane：测试 / 归档 / 上传 TestFlight / 提交审核
        ↓
回写状态、归档产物、清理临时密钥
```

在 ExampleApp 中，最常用的触发长这样：

```text
向 test 分支 push 或提 PR   →  exampleapp-test（自动测试 + Debug 构建）
手动指定 commit/tag          →  exampleapp-build（归档，不上传）
手动指定不可变 commit/tag    →  exampleapp-testflight-backup（上传 TestFlight）
手动 + 两阶段确认            →  exampleapp-appstore-backup（提交审核）
```

这里有一个边界值得从第一天就定清楚：**Jenkins 只负责“调度与编排”，签名和商店逻辑全部下沉到 Fastfile。** 本地、Jenkins、其他 CI 用同一套 lane，避免三套脚本漂移。

## 二、Jenkins 在 iOS CI/CD 里到底能做什么

Jenkins 并不是一个单独的“打包工具”，它是“Job + 节点 + 凭证 + 插件”的组合。最常见的几类能力如下。

| 需求 | Jenkins 提供的能力 | ExampleApp 中的用法 |
|---|---|---|
| 监听代码事件自动触发 | GitHub Webhook + Branch Source | `test` 分支 push / PR 自动跑测试 |
| 在指定机器上运行 | 静态 Agent 节点 + Label | 专用 Mac mini，单执行器避免模拟器/钥匙串冲突 |
| 安全注入密钥 | Credentials Binding | match 密码、`.p8`、SSH 私钥只在运行时出现 |
| 串行化高风险动作 | Lockable Resources | `exampleapp-release-signing` 锁禁止并发签名 |
| 运行声明式流水线 | Pipeline 插件 | `Jenkinsfile` 定义阶段与步骤 |
| 展示测试报告 | JUnit 插件 | 读取 `fastlane/test_output/**/*.xml` |
| 回写仓库状态 | GitHub 插件 | 回写 `jenkins/exampleapp-all` 检查状态 |
| 归档产物 | Archive Artifacts | 保留 IPA、dSYM、xcresult、日志 |
| 人工审批闸门 | Pipeline `input` | App Store 提交审核前由发布负责人确认 |

你不必一口气把所有能力都接上。一个健康的演进顺序通常是：

```text
先让 Jenkins 跑通测试 → 再接归档 Job
→ 再接 TestFlight 备用上传 → 最后才是 App Store 两阶段提交
```

每往前走一步，先确认本机用同一条 `fastlane` lane 可复现，再交给 Jenkins。这样当问题出现时，你能区分是工程问题、签名问题，还是 Jenkins 节点/凭证问题。

## 三、搭建前先准备什么

Jenkins 本身安装很简单，但 iOS 发布所依赖的外部信息不少。先把它们分成三类，会清楚很多。

### 1. macOS 构建主机

建议使用一台专用 Mac mini，并使用专用 macOS 用户运行 Jenkins。不要用个人登录用户的钥匙串和桌面环境作为长期 CI 依赖。

至少准备：

```text
macOS 15 或已经验证过的更高版本
Xcode（同时安装可用的 iOS 17.6+ Simulator runtime）
Command Line Tools、Git、SSH
受支持的 Ruby、Bundler、Java 17（或 Jenkins 当前 LTS 支持的更高版本）
稳定的 GitHub、Apple Developer 和 App Store Connect 网络访问
至少 20% 可用磁盘空间
```

如果巡检发现主机只有过旧的 macOS、Xcode、Ruby，且没有 Java、fastlane 或 Jenkins，这种主机不能直接作为 CI 节点，必须先完成工具链标准化。

### 2. Apple 侧准备

1. 在 Apple Developer 中创建主 App 和 Widget 两个 App ID，并启用 iCloud、App Groups 等项目实际使用的能力。
2. 确认 App Group 与 CloudKit 容器，例如 `group.com.example.exampleapp`、`iCloud.com.example.exampleapp`。
3. 使用 fastlane match 管理开发和 App Store 分发证书。match 仓库 deploy key 必须是该证书仓库的只读私钥，不能使用 `.pub` 公钥文件。
4. 在 App Store Connect 创建最小权限 API Key，记录 Key ID、Issuer ID 和 `.p8` 私钥。`.p8` 不进入 Git。
5. App Store 发布还需要审核联系人文件。仓库提供 `fastlane/review_information.example.yml` 模板，真实文件放在 Jenkins 用户的受限路径，不要提交。

### 3. GitHub 侧准备

- 应用仓库地址示例：`https://github.com/your-org/ExampleApp.git`。
- 你需要仓库管理员权限来创建 Webhook、配置 Branch Source 和分支保护。
- Jenkins Webhook 地址必须能从 GitHub 访问。生产环境使用 HTTPS、VPN 或安全反向代理，不要把 Mac mini 的管理端口直接暴露到公网。

### 4. 机密字段：哪些可以进仓库，哪些绝对不行

下面这张表很重要。

| 内容 | 是否可提交到业务仓库 | 合理位置 |
|---|---|---|
| `Jenkinsfile`、Fastfile、Gemfile | 可以 | 业务仓库 |
| Job 名称、节点标签、非敏感配置 | 通常可以 | 仓库 / Jenkins 配置 |
| match 证书仓库地址 | 私有项目通常可以 | `Matchfile` 或 Fastfile |
| match 加密密码 `MATCH_PASSWORD` | 不可以 | Jenkins Credentials / 密码管理器 |
| `.p12`、Provisioning Profile | 不可以直接放业务仓库 | match 加密仓库 |
| App Store Connect `.p8` 私钥 | 绝对不可以 | Jenkins Secret File / 本机受限路径 |
| Apple ID 密码 | 不建议作为自动化凭据 | 尽量改用 App Store Connect API Key |
| App Review 联系人信息 | 不应进公共仓库 | 本机受限文件或 Jenkins 受限路径 |

**本项目的固定事实（脱敏示例）**

```text
主 App Bundle ID：      com.example.exampleapp
Widget Bundle ID：       com.example.exampleapp.Widget
Apple Team ID：          ABCDE12345
match 证书仓库：          git@example.com:your-org/ios-certificates.git（分支 main）
App Group：              group.com.example.exampleapp
CloudKit 容器：          iCloud.com.example.exampleapp
```

> 示例项目的 CI 基线：Swift 6、SwiftUI、SwiftData，最低 iOS 17.6；Xcode 与 iOS Simulator runtime 需匹配；Ruby / Bundler / fastlane 版本由 `Gemfile.lock` 锁定。所有命令行 `xcodebuild` 或 fastlane 测试/构建都要注入 `OTHER_SWIFT_FLAGS="-disable-sandbox"`，否则 Swift 宏插件可能报 `swift-plugin-server produced malformed response`。

## 四、安装 Jenkins：从 Homebrew 开始

下面以 Homebrew 安装 Jenkins LTS 为主。Jenkins LTS 会随插件和 Java 支持范围更新，安装时不要固定一个过旧的版本。

### 1. 安装 Homebrew（已有可跳过）

```sh
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew update
brew --version
```

安装结束后，按终端提示把 Homebrew 加入当前用户的 `PATH`。如果公司策略不允许 Homebrew，也可以从 [Jenkins 下载页](https://www.jenkins.io/download/) 下载 macOS/Generic Java package；后续 Jenkins 页面配置完全相同。

### 2. 安装 Java 和 Jenkins LTS

```sh
brew install --cask temurin@17
brew install jenkins-lts

export JAVA_HOME="$(/usr/libexec/java_home -v 17)"
java -version
brew services start jenkins-lts
brew services list | grep jenkins
```

如果 `temurin@17` 在你所在的 Homebrew 源中不可用，请按 Jenkins 当前 LTS 的 [Java 要求](https://www.jenkins.io/doc/book/platform-information/support-policy-java/) 安装受支持的 Temurin 17/21，然后再次执行 `java -version`。不要在 Jenkins 运行后随意升级 Java 大版本，先在测试主机验证插件兼容性。

默认监听 `http://localhost:8080`。确认端口：

```sh
curl -I http://127.0.0.1:8080/login
```

### 3. 获取初始管理员密码

在浏览器打开 `http://localhost:8080`。页面会提示解锁 Jenkins。Homebrew 服务常见密码路径是：

```sh
cat "$HOME/.jenkins/secrets/initialAdminPassword"
```

如果该路径不存在，使用页面提示的路径；也可以只查找 Jenkins Home 下的文件：

```sh
find "$HOME" -path '*/secrets/initialAdminPassword' -print 2>/dev/null
```

把输出粘贴到解锁页面。该密码只用于首次解锁，后续使用你创建的管理员账号。

### 4. 创建第一个管理员账号

在“创建第一个管理员用户”页面填写：

| 字段 | 建议值 |
| --- | --- |
| 用户名 | 团队专用英文名，例如 `ci-user` |
| 密码 | 密码管理器生成的长密码，不与 macOS 密码相同 |
| 确认密码 | 同上 |
| 全名 | 真实姓名或团队名称 |
| 邮箱 | 能接收 Jenkins 通知的团队邮箱 |

点击“保存并完成”。随后设置 Jenkins URL：本机实验可先填 `http://localhost:8080/`；要接收 GitHub Webhook，必须改成外部可访问的 HTTPS 地址，例如 `https://jenkins.example.com/`。

## 五、安装必要的插件

首次向导可以选择“安装推荐插件”，完成后再补齐下面的插件。若选择“选择插件安装”，按表格搜索名称安装。

### 1. 必装插件

| 插件 | 用途 |
| --- | --- |
| Pipeline | 运行 Jenkinsfile 声明式流水线 |
| Git | Checkout Git 仓库 |
| GitHub | GitHub 服务器配置和状态回写 |
| GitHub Branch Source | Multibranch Pipeline 发现分支和 Pull Request |
| Credentials Binding | 在 `withCredentials` 中绑定密钥 |
| JUnit | 展示 `fastlane/test_output` 的 XML 报告 |
| Lockable Resources | 提供 `exampleapp-release-signing` 签名锁 |
| Timestamper | 日志显示时间戳 |
| AnsiColor | 显示 fastlane 彩色日志 |
| Workspace Cleanup | 清理工作区 |

`Pipeline` 会自动带上基础依赖。如果 Jenkins 提示依赖缺失，按页面的“安装依赖”一起安装。使用经典 Pipeline 页面即可，不必为了本项目安装 Blue Ocean。

### 2. 安装步骤

1. 点击左侧“Manage Jenkins”。
2. 点击“Plugins”（旧版本可能显示“Manage Plugins”）。
3. 打开“Available plugins”，逐个搜索并勾选上表插件。
4. 点击“Install”。
5. 安装完成后勾选“Restart Jenkins when installation is complete and no jobs are running”，或手动重启。
6. 回到“Installed plugins”，确认插件状态为 Enabled。

插件升级应在维护窗口进行，并在升级后重新跑一次测试 Job。

## 六、Manage Jenkins 逐项配置

### 1. System：Jenkins 地址、邮件和全局选项

路径：`Manage Jenkins` → `System`。

1. **Jenkins URL**：填写可被 GitHub 访问的 HTTPS 根地址，并保留末尾 `/`，例如 `https://jenkins.example.com/`。
2. **System Admin e-mail address**：填写维护邮箱。
3. **Locale/时区**：团队在中国时可使用 `Asia/Shanghai`，统一发布记录时间。
4. **GitHub Servers**：新增服务器，名称填 `github.com`，API URL 保持 `https://api.github.com`；选择下一节创建的 GitHub 凭证。
5. **Global properties**：只能放非敏感变量，例如 `CI=true`。不要放 `MATCH_PASSWORD`、`.p8` 内容或 SSH 私钥。

### 2. Security：关闭匿名访问

路径：`Manage Jenkins` → `Security`。

- 勾选 **Enable security**。
- Authorization 选择 **Matrix-based security** 或团队验证过的 Role-based 策略。
- 只给管理员 `Overall/Administer`；普通开发者只给读取 Job、查看日志、启动测试所需的权限。
- 取消匿名用户的管理和读取权限。
- 保持 CSRF 防护（crumb issuer）开启。
- API Token 只给个人账号或专用 GitHub/Jenkins 账号，使用最小权限并定期轮换。
- 不允许 Job 通过参数直接执行任意 shell 片段，流水线只接受校验过的 SHA、tag 和布尔确认参数。

保存后用无痕窗口确认未登录用户不能查看 Job 日志或凭证。

### 3. Credentials：创建五个发布凭证

路径：`Manage Jenkins` → `Credentials` → `System` → `Global credentials (unrestricted)` → `Add Credentials`。

凭证 ID 必须与仓库 Jenkinsfile 完全一致：

| ID | Kind | 填写方式 | 使用范围 |
| --- | --- | --- | --- |
| `exampleapp-match-deploy-key` | SSH Username with private key | Username 通常填 `git`；Private Key 粘贴 match 仓库只读私钥 | 两个备用发布 Job |
| `exampleapp-match-password` | Secret text | match 加密密码 | 两个备用发布 Job |
| `exampleapp-asc-key-id` | Secret text | App Store Connect Key ID | 两个备用发布 Job |
| `exampleapp-asc-issuer-id` | Secret text | App Store Connect Issuer ID（UUID） | 两个备用发布 Job |
| `exampleapp-asc-p8` | Secret file | 上传 `.p8` 私钥文件 | 两个备用发布 Job |

逐个创建：选择 Kind → Scope 选 `Global` → 手动填精确 ID → 填 Description → 点击“Create”。Description 只写用途，不写私钥内容。

测试 Job（例如 `ci/jenkins/test.Jenkinsfile`）**严禁绑定这五个凭证**。它只做测试和 Debug 构建，即使测试失败也不应访问 App Store Connect。

另外准备一个用于 GitHub 状态回写的细粒度 GitHub Token，只授予仓库状态/Checks 所需权限。不要把 Token 写入 Jenkinsfile。

### 4. Lockable Resources：创建签名锁

路径：`Manage Jenkins` → `System` → **Lockable Resources Manager**。

新增：

```text
Name: exampleapp-release-signing
Description: ExampleApp 发布签名和 match 资源，禁止并发
Labels: exampleapp-release
```

两个备用发布流水线会在签名阶段执行 `lock(resource: 'exampleapp-release-signing')`；普通测试 Job 不需要此锁。

### 5. Nodes：配置 Mac 节点和标签

路径：`Manage Jenkins` → `Nodes` → `New Node`。

推荐建立静态节点：

1. Name：`exampleapp-mac-mini`。
2. Type：`Permanent Agent`。
3. Number of executors：`1`，避免模拟器和钥匙串并发冲突。
4. Remote root directory：例如 `/Users/ci-user/jenkins-agent`，必须属于 Jenkins 用户。
5. Labels：`exampleapp macos ios xcode-26.3`。
6. Usage：选择“Only build jobs with label expressions matching this node”。
7. Launch method：按网络选择 SSH 或 inbound agent，连接后确认节点 Online。

仓库中的 label 有差异：

- 测试 Jenkinsfile：`exampleapp && macos && ios`。
- 根 `Jenkinsfile`、两个备用发布 Jenkinsfile：`exampleapp && xcode-26.3`。

因此同一节点设置四个标签最简单。测试流水线会运行选择模拟器的脚本动态选择设备，不要把设备名称写死在节点配置里。

### 6. GitHub 服务器和状态回写

在 `Manage Jenkins` → `System` → `GitHub Servers` 中配置：

```text
Name: github.com
API URL: https://api.github.com
Credentials: 前面创建的细粒度 GitHub Token 或 GitHub App 凭证
```

点击“Test connection”成功后保存。根 `Jenkinsfile` 会回写 `jenkins/exampleapp-all`：

| 状态 | 含义 |
| --- | --- |
| `PENDING` | Jenkins 已接收，正在排队或运行 |
| `SUCCESS` | 必需阶段通过 |
| `FAILURE` | 测试、完整性检查或构建失败 |
| `ERROR` | Jenkins、凭证或节点基础设施异常 |

如果插件不支持状态回写，日志中的警告不代表测试成功；先修复 GitHub 插件和凭证。

### 7. Tools：固定 Jenkins 使用的工具

路径：`Manage Jenkins` → `Tools`（旧版本显示 `Global Tool Configuration`）。

1. **JDK installations**：新增一个 JDK，名称填 `temurin-17`。如果 Jenkins 节点已通过 Homebrew/安装包提供 Java，取消“自动安装”，填写实际路径，并用 `echo "$JAVA_HOME"` 确认。
2. **Git installations**：新增 `system-git`，路径填 `/usr/bin/git` 或 `which git` 输出的路径。不要让 Job 运行时随机下载 Git。
3. **Ruby、Xcode、Simulator**：不在这里使用自动安装器。Ruby 由节点固定的 Homebrew/RVM 环境提供；Xcode 由 macOS 节点管理员安装到 `/Applications/Xcode.app`，流水线用 `xcode-select -p` 和 `xcodebuild -version` 校验。
4. 如果 Jenkins 服务启动时没有读取用户的 shell 配置，在节点配置的 Environment variables 中设置非敏感的 `PATH`、`JAVA_HOME` 和 `DEVELOPER_DIR`；不要设置凭证变量。

保存后在节点执行一次“运行命令”或临时测试 Job，确认 `java -version`、`git --version`、`ruby --version`、`bundle --version` 和 `xcodebuild -version` 均来自预期路径。工具版本漂移时先修节点，不要在 Jenkinsfile 里用 `sudo` 临时切换系统工具。

## 七、准备 Xcode、Ruby 和项目依赖

### 1. Xcode 和 Simulator

在 Jenkins 使用的 macOS 用户下执行：

```sh
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
sudo xcodebuild -license accept
xcode-select -p
xcodebuild -version
xcrun simctl list runtimes
xcrun simctl list devices available
```

确认 Xcode 与 iOS runtime 匹配，并且有 iOS 17.6 或更高的可用 iPhone Simulator。不要只用“iPhone 17 Pro”名称判断设备是否存在。常见根因是 Xcode/runtime 不匹配，导致 `Runtime build 'xxx' not found` 或 `No device found with name 'iPhone 17 Pro'`。

### 2. Ruby、Bundler 和 fastlane

```sh
brew install ruby@3.3
echo 'export PATH="$(brew --prefix ruby@3.3)/bin:$PATH"' >> "$HOME/.zprofile"
export PATH="$(brew --prefix ruby@3.3)/bin:$PATH"
ruby --version
gem install bundler -v 4.0.16
bundle --version
```

项目的 `Gemfile.lock` 已锁定 Ruby、Bundler 和 fastlane 版本。仓库根目录执行：

```sh
bundle install --jobs 4 --retry 3
bundle exec fastlane --version
```

所有 CI 命令都使用 `bundle exec`，避免调用系统里的另一套 fastlane。

### 3. 第一次本地复现

```sh
cd /path/to/ExampleApp
export OTHER_SWIFT_FLAGS='-disable-sandbox'
bundle install --jobs 4 --retry 3
ruby -c fastlane/Fastfile
xcodebuild -list -project ExampleApp.xcodeproj OTHER_SWIFT_FLAGS="$OTHER_SWIFT_FLAGS"
bundle exec fastlane ios metadata_validate
bundle exec fastlane ios test
```

预期产物：

```text
fastlane/test_output/ExampleAppTests.xml
fastlane/test_output/ExampleAppTests.xcresult
fastlane/test_output/logs/
build/Build/Products/Debug-iphonesimulator/ExampleApp.app
```

这一步失败时先修复主机或项目环境，不要在 Jenkins 页面反复点击构建。

## 八、测试、构建、上传脚本怎么写

项目把签名和商店逻辑集中在 Fastfile，把 Jenkins 保留为编排层。本地、Jenkins 和 GitHub Actions 使用同一套 lane，避免三套脚本漂移。

### 1. 测试脚本

`ci/jenkins/test.Jenkinsfile` 的核心步骤如下：

```bash
#!/usr/bin/env bash
set -euo pipefail

export CI=true
export OTHER_SWIFT_FLAGS='-disable-sandbox'
bundle install --jobs 4 --retry 3
ruby -c fastlane/Fastfile
xcodebuild -list -project ExampleApp.xcodeproj
bundle exec fastlane ios metadata_validate
bundle exec fastlane ios test

mkdir -p "$WORKSPACE/build/jenkins/DerivedData"
xcodebuild \
  -project ExampleApp.xcodeproj \
  -scheme ExampleApp \
  -configuration Debug \
  -derivedDataPath "$WORKSPACE/build/jenkins/DerivedData" \
  -destination "$SIMULATOR_DESTINATION" \
  OTHER_SWIFT_FLAGS="$OTHER_SWIFT_FLAGS" \
  build
```

`SIMULATOR_DESTINATION` 由选择模拟器的脚本产生，例如 `platform=iOS Simulator,id=<UDID>`。fastlane `test` 会生成 JUnit XML、日志和 `.xcresult`，Jenkins 用 JUnit 插件读取 XML。

### 2. 开发包和 App Store 归档脚本

开发包：

```bash
export OTHER_SWIFT_FLAGS='-disable-sandbox'
bundle exec fastlane ios exampleapp_build_dev
```

App Store 归档（只生成本地 IPA）：

```bash
export OTHER_SWIFT_FLAGS='-disable-sandbox'
export BUILD_NUMBER='123'   # 仅发布构建设置，必须唯一
bundle exec fastlane ios exampleapp_archive_appstore
```

产物目录：

```text
fastlane/build_output/ios/exampleapp/dev/
fastlane/build_output/ios/exampleapp/appstore/
```

`BUILD_NUMBER` 会写入工程 build setting，所以必须在干净 checkout 中运行，并记录版本号、build number 和 commit SHA。普通测试不要占用 App Store Connect 的正式 build number。

### 3. 安全的签名脚本

签名任务必须使用临时钥匙串。凭证由 Jenkins `withCredentials` 注入，绝不能写死：

```bash
set -euo pipefail
export MATCH_KEYCHAIN_NAME="exampleapp-ci-${BUILD_TAG}"
export MATCH_KEYCHAIN_PASSWORD="$(uuidgen)"
security create-keychain -p "$MATCH_KEYCHAIN_PASSWORD" "$MATCH_KEYCHAIN_NAME"
security set-keychain-settings -lut 21600 "$MATCH_KEYCHAIN_NAME"
security unlock-keychain -p "$MATCH_KEYCHAIN_PASSWORD" "$MATCH_KEYCHAIN_NAME"
security list-keychains -d user -s "$MATCH_KEYCHAIN_NAME" "$HOME/Library/Keychains/login.keychain-db"
security default-keychain -d user -s "$MATCH_KEYCHAIN_NAME"

install -d -m 700 "$HOME/.ssh"
cp "$MATCH_KEY_FILE" "$HOME/.ssh/exampleapp_match_key"
chmod 600 "$HOME/.ssh/exampleapp_match_key"
ssh-keyscan github.com >> "$HOME/.ssh/known_hosts"
export GIT_SSH_COMMAND="ssh -i $HOME/.ssh/exampleapp_match_key -o IdentitiesOnly=yes"

install -d -m 700 "$WORKSPACE/.ci-secrets"
cp "$ASC_KEY_FILEPATH" "$WORKSPACE/.ci-secrets/AuthKey.p8"
chmod 600 "$WORKSPACE/.ci-secrets/AuthKey.p8"
export ASC_KEY_FILEPATH="$WORKSPACE/.ci-secrets/AuthKey.p8"
export FASTLANE_WWDR_USE_HTTP1_AND_RETRIES=true
bundle exec fastlane ios beta
```

Jenkins `post { always { ... } }` 必须恢复 login keychain 并删除 `exampleapp_match_key`、`AuthKey.p8`、临时钥匙串和工作区密钥。清理脚本只允许清理白名单路径；测试流水线最终还会用 `deleteDir()` 清空专用工作区。

### 4. TestFlight 和 App Store 上传

TestFlight 只上传、不自动分发外部测试组：

```bash
bundle exec fastlane ios beta
```

正式提交审核：

```bash
CONFIRM_RELEASE=true AUTOMATIC_RELEASE=false \
  bundle exec fastlane ios release
```

`release` 确认模式要求当前提交精确匹配 `v<marketing-version>-build.<N>`，审核通过后仍由 App Store Connect 人工发布。不要提供“自动公开发布”参数。

发布前检查：

```bash
bundle exec fastlane ios asc_verify
bundle exec fastlane ios metadata_validate
bundle exec fastlane ios precheck
```

### 5. Fastlane lane 对照

| Lane | 作用 | 外部状态 |
| --- | --- | --- |
| `exampleapp_dev` | match 拉取主 App + Widget 开发证书/profile | 无 |
| `exampleapp_appstore` | match 拉取 App Store 分发证书/profile | 无 |
| `test` | 动态选择 Simulator，运行测试并输出 JUnit/xcresult | 无 |
| `exampleapp_build_dev` | Development IPA、archive、dSYM | 无 |
| `exampleapp_archive_appstore` | App Store IPA、archive、dSYM | 无 |
| `asc_verify` | 验证 App Store Connect API 只读权限 | 无 |
| `next_build_number` | 读取本地和 ASC 的下一 build number | 无 |
| `release_notes` | 从 feat/fix/perf 提交生成中文发布说明 | 无 |
| `release_prepare` | 默认 dry-run；确认后递增 build、提交并创建 tag | 修改 Git，需确认 |
| `beta` | 上传 TestFlight，默认不分发 | 上传 |
| `release` | 测试、归档、precheck，确认后上传并提交审核 | 提交审核 |
| `metadata_validate` | 校验多语言商店元数据和 HTTPS 链接 | 无 |
| `precheck` | App Store Connect 元数据预检查 | 无 |
| `metadata_upload` | 上传元数据和配置的截图，不上传 IPA/不提交审核 | 元数据变化 |
| `screenshots` | 生成多语言 iPhone/iPad 截图 | 无 |

版本发布顺序必须是：确定唯一 build number → 提交 build number 变更 → 创建匹配 tag → 推送 commit 和 tag → 以该 tag 测试、归档和发布。不能先创建 tag 再修改 build number。

## 九、创建自动化测试 Job

测试 Job 使用 Multibranch Pipeline，让 Jenkins 自动发现 `test` 分支和 Pull Request。

### 1. 创建 Multibranch Pipeline

1. 首页点击“New Item”。
2. 名称填 `exampleapp-test`，选择“Multibranch Pipeline”，点击“OK”。
3. “Branch Sources” → “Add source” → “GitHub”。
4. GitHub server 选择 `github.com`，Owner 填 `your-org`，Repository 填 `ExampleApp`。
5. Credentials 选择仓库读取凭证；公开仓库可留空，但推荐使用细粒度 GitHub App/PAT。
6. Behaviors 选择发现 `test` 分支和 Pull Request。PR 策略选择构建 head revision 时，测试的是 PR 当前 commit；若团队需要验证合并结果，再改为 merge revision。
7. “Build Configuration” → Mode 选“by Jenkinsfile”，Script Path 填：

```text
ci/jenkins/test.Jenkinsfile
```

8. Orphaned Item Strategy 保留最近 20 个分支任务。
9. Scan trigger 可设 5 分钟作为 Webhook 故障兜底，正常触发仍依赖 Webhook。
10. 保存后点击“Scan Multibranch Pipeline Now”。

### 2. 测试 Pipeline 阶段

`ci/jenkins/test.Jenkinsfile` 会：清理工作区并 checkout 精确 commit；动态选择 Simulator；执行依赖、Fastfile 语法、工程列表和三语元数据校验；执行 `fastlane ios test`；执行 Debug build；发布 JUnit、日志、xcresult 和环境信息；最后清理 DerivedData、测试输出并 `deleteDir()`。

它不会调用 `match`、`beta` 或 `release`，也不绑定 App Store Connect 凭证。

### 3. 配置 GitHub Webhook

GitHub 仓库 `Settings` → `Webhooks` → `Add webhook`：

```text
Payload URL: https://jenkins.example.com/github-webhook/
Content type: application/json
Secret: 与 Jenkins webhook 校验配置相同的随机字符串
Events: Pushes、Pull requests、Ping
Active: 勾选
```

保存后在 Recent Deliveries 发送 `ping`，确认 Jenkins 返回 200。再向 `test` push 一个安全提交，检查是否创建构建。

不同版本的 GitHub Branch Source/GitHub 插件对 webhook secret 的入口名称不同：如果 `Manage Jenkins → System → GitHub Servers` 提供 secret 校验字段，就填入与 GitHub Webhook 相同的随机值；如果页面没有该字段，使用 GitHub App 的托管 Webhook，或在 HTTPS 反向代理层校验 GitHub 的 HMAC 签名。不要仅仅在 GitHub 页面填写 Secret，却没有任何一层实际校验它。

## 十、创建打包 Job

如果团队希望把“测试”和“签名归档”显示成两个独立的 Job，可以创建一个只执行归档阶段 A 的手动 Job。它复用 `ci/jenkins/appstore-backup.Jenkinsfile`，不复制签名逻辑，也不会上传商店。

### 1. 创建方式

1. 首页 → “New Item” → 名称 `exampleapp-build`。
2. 选择“Pipeline” → “OK”。
3. Build Triggers 全部不勾选。
4. Pipeline → Definition 选择“Pipeline script from SCM”。
5. SCM 选择 Git，Repository URL 填应用仓库地址，Credentials 选择仓库读取凭证。
6. Branches to build 填 `*/main` 或要打包的受信任分支。
7. Script Path 填 `ci/jenkins/appstore-backup.Jenkinsfile`。
8. 保存后点击一次“Build Now”，让 Jenkins 解析参数。

### 2. 打包参数

点击“Build with Parameters”：

| 参数 | 示例 | 填写方式 |
| --- | --- | --- |
| `RELEASE_REF` | `a1b2c3d` 或 `v1.0.0-build.7` | 必填，固定要打包的 commit/tag |
| `BUILD_NUMBER` | `8` | 必填，唯一正整数 |
| `CONFIRM_PRECHECK` | 勾选 | 允许执行测试、签名归档和 precheck |
| `CONFIRM_RELEASE` | 不勾选 | 保持阶段 B 关闭，绝不上传或提交审核 |

阶段 A 成功后，在 Job 的 Artifacts 中下载：

```text
fastlane/build_output/ios/exampleapp/appstore/ExampleApp.ipa
fastlane/build_output/ios/exampleapp/appstore/ExampleApp.xcarchive/
fastlane/test_output/
commit.txt
environment.txt
```

这个 Job 仍然需要 App Store 分发凭证，因为归档要验证主 App 和 Widget 的签名；它不应被当作无凭证的普通测试 Job。若只需要快速编译检查，使用测试 Job 的 Debug build。

## 十一、创建 TestFlight 备用发布 Job

该 Job 只在默认 CI 或其 macOS runner 故障、需要恢复 TestFlight 上传时使用，必须手动触发。

### 1. 创建 Pipeline Job

1. 首页 → “New Item” → 名称 `exampleapp-testflight-backup`。
2. 选择“Pipeline” → “OK”。
3. Build Triggers 全部不勾选。
4. Pipeline → Definition 选择“Pipeline script from SCM”。
5. SCM 选择 Git；Repository URL 填 `git@example.com:your-org/ExampleApp.git` 或 HTTPS；Credentials 选择应用仓库读取凭证。
6. Branches to build 填 `*/main` 或团队指定的受信任发布分支。
7. Script Path 填 `ci/jenkins/testflight-backup.Jenkinsfile`。
8. 保存后先点击一次“Build Now”让 Jenkins 解析 Jenkinsfile；参数通常从第二次运行开始显示。

### 2. 运行参数

| 参数 | 示例 | 要求 |
| --- | --- | --- |
| `RELEASE_REF` | `a1b2c3d`、完整 SHA 或 `v1.0.0-build.7` | 必填，只接受 SHA 或 `v*` tag |
| `CONFIRM_UPLOAD` | 勾选 | 不勾选会在参数校验阶段失败 |

流水线会 checkout 指定 ref，运行完整测试，然后在 `exampleapp-release-signing` 锁内创建临时钥匙串、读取 match 和 ASC 凭证，最后调用 `bundle exec fastlane ios beta`。它会校验 App 与 Widget dSYM，上传后默认不分发外部测试组。

运行前把 Jenkins 构建 URL、commit SHA、版本号和 build number 记录到受控发布记录。完成后下载 `commit.txt`、`environment.txt`、JUnit、xcresult、IPA、archive 和 dSYM artifact。

## 十二、创建 App Store 备用发布 Job（两阶段闸门）

App Store 备用发布采用两阶段：阶段 A 只测试、归档和预检查；阶段 B 由发布负责人批准后提交审核。

### 1. 配置审核联系人文件

以 Jenkins 运行用户执行：

```sh
mkdir -p "$HOME/.config/exampleapp/appstoreconnect"
cp /path/to/ExampleApp/fastlane/review_information.example.yml \
  "$HOME/.config/exampleapp/appstoreconnect/review_information.yml"
chmod 600 "$HOME/.config/exampleapp/appstoreconnect/review_information.yml"
```

编辑真实姓名、电话、邮箱和审核说明。独立 `appstore-backup.Jenkinsfile` 没有绑定审核信息 Secret，因此必须使用默认路径，或在节点环境提供 `ASC_REVIEW_INFO_FILEPATH`。内容不得打印或归档。

### 2. 创建 Job

按 TestFlight Job 的步骤创建 Pipeline，改为：

```text
Job name: exampleapp-appstore-backup
Script Path: ci/jenkins/appstore-backup.Jenkinsfile
Build Triggers: 全部关闭
```

确保 Jenkins 用户属于可通过 `input` 审批的 `exampleapp-release-managers` 用户组。若没有组管理能力，应在代码评审后把 Jenkinsfile 的 `submitter` 改为实际用户 ID，不能删除人工审批。

### 3. 参数和两个阶段

| 参数 | 示例 | 要求 |
| --- | --- | --- |
| `RELEASE_REF` | `v1.0.0-build.7` 或完整 SHA | 必填，必须可由完整 Git clone 获取 |
| `BUILD_NUMBER` | `8` | 必填，唯一正整数 |
| `CONFIRM_PRECHECK` | 阶段 A 勾选 | 允许预演，否则失败 |
| `CONFIRM_RELEASE` | 阶段 A 不勾选；正式提交时勾选 | 进入审批和阶段 B |

阶段 A：checkout 指定 ref → 安装依赖 → 元数据校验 → 完整测试 → 在签名锁内归档 → `precheck`。它只产生 IPA、archive、dSYM 和报告，不上传、不提交审核。

阶段 B：只有 `CONFIRM_RELEASE=true` 才显示 Jenkins `input`；审批人限制为 `exampleapp-release-managers`；再次比较当前 checkout SHA 与 `commit.txt`；调用 `CONFIRM_RELEASE=true bundle exec fastlane ios release`；上传 IPA 和三语元数据并提交审核，固定 `automatic_release=false`。审核通过后仍由 App Store Connect 人工公开发布。

一个容易被忽略的 Jenkins 细节：每个 `sh` 步骤都是新的 shell，阶段 A 中 `export` 的 `GIT_SSH_COMMAND`、`MATCH_KEYCHAIN_NAME` 和 `ASC_KEY_FILEPATH` 不会自动传到阶段 B。首次启用真实发布前，必须在 Jenkins 上做一次低风险预演，并确认阶段 B 也能访问 match 仓库和临时钥匙串；若日志显示 SSH 或签名失败，应把阶段 A 的 SSH/钥匙串初始化片段复制到阶段 B，或把两步合并到同一个 `sh`/共享脚本中。不能假设前一个阶段的 `export` 会跨步骤生效。

## 十三、Artifact、报告和清理

每个 Job 至少归档：

```text
commit.txt
environment.txt
simulator.env
fastlane/test_output/**
fastlane/build_output/ios/exampleapp/**
build/jenkins/**
build/*.log
**/*.xcresult/**
**/*.ipa
**/*.dSYM/**
```

Jenkins JUnit pattern：`fastlane/test_output/**/*.xml`。

建议保留：普通测试 14 天、`test` 候选归档 30 天、备用发布至少 180 天，正式发布按团队审计策略长期保留。每次构建至少能追溯 repository、branch/tag、commit SHA、commit message、版本号、build number、Xcode、macOS、Job 和 Jenkins build number。

`post { always { ... } }` 必须恢复 login keychain，删除临时钥匙串、match deploy key、`AuthKey.p8`、审核信息临时文件、DerivedData 和输出目录。禁止对 `$HOME`、`/Users`、工作区根目录或系统根目录做递归删除。

## 十四、GitHub 分支与 CI/CD 规则

```text
feature/* / fix/* / refactor/* / hotfix/*
        ↓ Pull Request
test    ← Jenkins 自动测试和 Debug 构建
        ↓ 人工审核
main    ← 默认 CI 归档和正式发布
```

完成 Webhook、状态回写和误报演练后，再在 GitHub `Settings` → `Branches` 中启用 Pull Request、审批、状态检查 required check、限制直接 push 和合并后自动删除分支。

不要只检查分支名。测试、归档和上传必须使用同一个不可变 commit；测试完成后如果分支又有新提交，旧状态不能用于合并或发布。

## 十五、这次搭建中遇到的真实问题与解决方案

下面这些不是“理论上可能发生”的问题，而是 ExampleApp 在搭建 Jenkins 与后续 CI 时实际验证过或遇到过的情况。

### 问题 1：命令行构建时出现 Swift 宏找不到

**现象**

命令行 `xcodebuild` 或 Fastlane 测试时出现大量类似错误：

```text
could not be found for macro 'Model()'
swift-plugin-server produced malformed response
```

**原因**

项目使用 SwiftData 的宏。当前命令行环境中，Swift 宏插件沙箱导致插件服务无法正常启动；Xcode IDE 内构建却可能正常，所以问题很容易被误判成代码问题。

**解决方案**

在所有命令行构建入口统一传递：

```bash
export OTHER_SWIFT_FLAGS='-disable-sandbox'
```

无论是 `xcodebuild`、`fastlane ios test`、归档还是截图，都带上它。关键不是这一个参数本身，而是不要只在某一条命令里临时补上；否则本机测试通过、Jenkins 或 release 又失败，问题会反复出现。

### 问题 2：Simulator / runtime 不匹配

**现象**

`xcrun simctl list devices available` 找不到目标设备，或 `xcodebuild` 报 `Runtime build 'xxx' not found`、`No device found with name 'iPhone 17 Pro'`。

**原因**

Xcode 版本与 iOS Simulator runtime 不匹配，或只按“iPhone 17 Pro”名称判断设备是否存在。

**解决方案**

```sh
xcode-select -p
xcodebuild -version
xcrun simctl list runtimes
xcrun simctl list devices available
xcodebuild -showdestinations -project ExampleApp.xcodeproj -scheme ExampleApp
```

不要只按设备名称判断。让选择模拟器的脚本挑选满足最低 iOS 版本的可用 iPhone UDID，避免把设备名写死在节点配置里。

### 问题 3：“没有测试报告”

**现象**

Jenkins JUnit 报告为空，构建看似“成功”却没有任何测试结论。

**原因**

多数时候测试根本没有启动，真正原因是 Xcode、runtime 或宏插件失败；只有测试成功后才检查 XML 报告。

**解决方案**

先看 `fastlane/test_output/logs` 和 xcodebuild 输出，确认测试真正执行并生成 `*.xcresult`。不要只看“构建绿了”就认为测试通过了——确保 JUnit 插件确实读到了 `fastlane/test_output/**/*.xml`。

### 问题 4：WWDR / 签名失败

**现象**

归档或上传时报证书/签名相关错误，临时钥匙串似乎没生效。

**原因**

临时钥匙串未解锁、未进入搜索列表、未设为默认，或 WWDR 证书拉取被网络限制。

**解决方案**

检查临时钥匙串已解锁、`security set-keychain-settings -lut 21600` 已设置、临时钥匙串位于搜索列表并是默认钥匙串，同时设置：

```bash
export FASTLANE_WWDR_USE_HTTP1_AND_RETRIES=true
```

构建结束恢复 login keychain，否则下一次构建可能继续使用已删除的钥匙串。

### 问题 5：match clone 失败

**现象**

`match` 无法 clone 私有证书仓库，或报 SSH permission denied、502、超时。

**原因**

SSH 私钥没有读取权限、Deploy Key 没加到正确仓库、`known_hosts` 缺 GitHub 指纹，或代理配置错误。

**解决方案**

```sh
ssh-keygen -y -f /path/to/private-key > /tmp/match-key.pub
git ls-remote git@example.com:your-org/ios-certificates.git
```

确认凭证是证书仓库的只读私钥，不是 `.pub` 文件，也不是个人源仓库密钥。CI 里显式设置 `GIT_SSH_COMMAND` 指定唯一 Deploy Key；本机代理异常时，用已验证可用的代理做一次性 Git 覆盖，不要永久污染全局配置。

### 问题 6：ASC `.p8` 路径无效

**现象**

Fastfile 报 `.p8` 后缀检查失败或找不到私钥。

**原因**

Jenkins Secret File 路径可能没有 `.p8` 后缀，而 Fastfile 会检查后缀。

**解决方案**

复制到工作区 `.ci-secrets/AuthKey.p8`、设置权限 `600`，导出 `ASC_KEY_FILEPATH`，结束后删除。不要直接把原始 Secret File 路径传给只认 `.p8` 后缀的检查。

### 问题 7：Widget 不刷新

**现象**

主 App 数据已更新，但 Widget 不刷新。

**原因**

Widget 不直接读 SwiftData。若 provisioning profile 没有 App Group 能力，真机 `containerURL` 可能为 nil。

**解决方案**

确认主 App 写入 App Group 的 `widget_snapshot.json` 并调用 `WidgetCenter.shared.reloadAllTimelines()`。Jenkins 不应尝试让 Widget 直接读 SwiftData，也不能把数据库搬到 App Group。Apple Developer 后台、match、Fastfile、CI 签名检查四处都要包含 Widget 的 Bundle ID 与 App Group 能力。

### 问题 8：tag / build number 不一致

**现象**

正式 `release` 报当前提交不匹配 `v<version>-build.<N>`，或上传后被拒“build number 已存在”。

**原因**

发布要求精确匹配 tag；或并行构建重复使用同一个 build number。

**解决方案**

正确顺序是提交 build number、创建 tag、推送 commit 和 tag，再以该 tag 发布。上传失败后不要复用同一 ASC build number；频繁并行发布时应串行化或用外部构建号策略协调。

### 问题 9：截图尺寸不符合要求

**现象**

生成的截图被 App Store Connect 拒收，提示尺寸不符合槽位。

**原因**

不同设备型号输出的像素尺寸不同，不能把某一型号的截图直接当作某尺寸槽位。

**解决方案**

生成后用 `sips -g pixelWidth -g pixelHeight file.png` 检查实际像素，确认与 App Store Connect 要求的槽位尺寸一致后再上传。

### 问题 10：发布 Job 跨阶段环境变量丢失

**现象**

阶段 A 已配置好 `GIT_SSH_COMMAND`、临时钥匙串，阶段 B 却报 SSH 或签名失败。

**原因**

Jenkins 每个 `sh` 步骤都是新的 shell，阶段 A 的 `export` 不会自动传到阶段 B。

**解决方案**

把 SSH/钥匙串初始化片段复制到阶段 B，或把两步合并到同一个 `sh`/共享脚本中。不能假设前一个阶段的 `export` 会跨步骤生效；首次启用真实发布前必须在 Jenkins 上做一次低风险预演。

## 十六、一份可照着执行的日常清单

### 日常开发

```sh
bundle install
bundle exec fastlane ios test
```

测试通过后再提交代码。不要等到发布日才第一次运行自动化。

### 需要给测试人员一个包

```sh
# 本地归档，先检查产物
bundle exec fastlane ios exampleapp_archive_appstore

# 需要上传 TestFlight 时（Jenkins 备用 Job 或手动）
bundle exec fastlane ios beta
```

### 需要修改商店文案

```sh
# 先做本地格式校验
bundle exec fastlane ios metadata_validate

# 再做 App Store Connect 预检
bundle exec fastlane ios precheck

# 确认后只上传文案
bundle exec fastlane ios metadata_upload
```

### 准备正式审核（Jenkins 备用 Job）

1. 手动触发 `exampleapp-build`，填 `RELEASE_REF` 与唯一 `BUILD_NUMBER`，勾选 `CONFIRM_PRECHECK`，下载归档产物自查。
2. 手动触发 `exampleapp-testflight-backup`，必须勾选 `CONFIRM_UPLOAD`，确认临时钥匙串和 secret 已清理。
3. 手动触发 `exampleapp-appstore-backup` 阶段 A（勾选 `CONFIRM_PRECHECK`），确认只生成预演产物，不上传、不提交审核。
4. 阶段 B 由 `exampleapp-release-managers` 审批，确认 SHA 未变化、`CONFIRM_RELEASE=true` 生效、`automatic_release=false`。
5. 检查 artifact 中的 commit、版本、build number、IPA、dSYM、JUnit、xcresult 和日志，确认没有 `.p8`、SSH 私钥或审核联系人文件。

正式提交前，还应确认：当前工作树干净、当前提交具有正确 tag、TestFlight 已验证、审核联系信息已准备好、隐私链接和商店元数据正确。

## 结语

Jenkins 最好的使用方式，不是追求“一台服务器把所有事情都做掉”，而是把 iOS 交付里的每个高风险环节做成边界明确、可单独验证的流水线：

- 测试 Job 只负责可靠测试和报告，不碰发布凭证；
- 打包 Job 只负责签名归档导出 IPA，不自动上传；
- TestFlight / App Store 备用 Job 在签名锁与人工闸门之后才允许外部动作；
- Fastlane 只负责把受控签名材料安全地带到当前机器并产出制品；
- Jenkins 只负责在合适的时机、用合适的节点、注入合适的凭证来调用这些能力。

当这些规则被写进仓库、被本机和 CI 共同执行后，发版就不再依赖某位同事记得住一套手工步骤。它变成了一条可阅读、可复现、可审计的工程流程——这才是 Jenkins 在 iOS 项目里真正长期有价值的地方。

