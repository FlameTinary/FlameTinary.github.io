---
title: iOS项目如何接入Fastlane
date: 2026-09-07
tags:
  - iOS
  - Fastlane
  - 自动化
  - 证书签名
  - TestFlight
---

# iOS项目如何接入Fastlane

做 iOS 开发的人，大概都经历过这种时刻：本地调试明明很顺，临近提测才开始手动改版本号、找证书、切 Scheme、Archive、导 IPA、登录 App Store Connect、传 TestFlight。中间只要漏一步，或者用错一张 Profile，就得从头再来。

Fastlane 的价值不是“替你点几个按钮”，而是把这些重复、容易出错、又必须可追溯的动作变成代码。你把规则写进 `Fastfile`，以后无论是自己在终端运行，还是让 GitHub Actions 的 macOS Runner 运行，走的都是同一套流程。

本文以 HabitPulse 的真实接入过程为例，完整说明一个 SwiftUI iOS 项目如何接入 Fastlane：它能做什么、项目里常用哪些功能、证书怎么管理、前置字段如何准备，以及接入过程中实际遇到的问题和解决办法。项目包含主 App 和 Widget Extension，所有账号、仓库地址、Bundle ID、Team ID 与密钥内容均已脱敏。

## 一、先用一句话理解 Fastlane

Fastlane 是一套用 Ruby 配置的移动端自动化工具集。它不替代 Xcode，也不替代 Apple Developer 或 App Store Connect；它做的是把这些平台和命令行工具串成稳定的流水线。

你可以把它理解为 iOS 项目的“自动化总控台”：

```text
开发者输入一条 Fastlane 命令
        ↓
Fastlane 选择正确的证书/Profile、调用 xcodebuild
        ↓
测试 / 归档 / 导出 IPA / 生成截图 / 上传 TestFlight / 提交审核
        ↓
输出 IPA、dSYM、xcresult、日志、发布说明等可追溯产物
```

在 HabitPulse 中，最常用的命令长这样：

```sh
# 用固定模拟器运行测试，并生成报告
bundle exec fastlane ios test

# 安装开发签名材料（主 App + Widget）
bundle exec fastlane ios habitpulse_dev

# 生成开发 IPA、archive、dSYM、xcresult
bundle exec fastlane ios habitpulse_build_dev

# 生成 App Store IPA、archive、dSYM、xcresult，但不上传
bundle exec fastlane ios habitpulse_archive_appstore

# 上传 TestFlight
bundle exec fastlane ios beta
```

这里有一个习惯值得从第一天就建立：**始终使用 `bundle exec`。**

```sh
bundle exec fastlane ios test
```

不要随手运行系统全局的：

```sh
fastlane ios test
```

因为前者一定使用项目 `Gemfile.lock` 锁定的 Fastlane 版本，团队成员和 CI 才会保持一致；后者可能悄悄调用你电脑里另一个版本的 Fastlane，今天能跑、明天升级后就变了。

## 二、Fastlane 到底能做什么

Fastlane 并不是一个单独的“打包工具”，它是许多 action 的集合。最常见的几类能力如下。

| 需求 | 常见 Fastlane action | HabitPulse 中的用法 |
|---|---|---|
| 执行单元/UI 测试 | `run_tests` / `scan` | 固定模拟器运行测试，保存 JUnit、`.xcresult` 和日志 |
| 管理证书与 Profile | `match` | 主 App 与 Widget 使用同一份受加密保护的签名仓库 |
| Archive 与导出 IPA | `build_app` / `gym` | 导出开发版或 App Store 版 IPA、archive 与 dSYM |
| 读取和更新版本号 | `get_version_number`、`get_build_number`、`increment_build_number` | 生成唯一 TestFlight build number，准备发布 tag |
| 调用 App Store Connect API | `app_store_connect_api_key` | 无需 Apple ID 会话即可读取构建号、上传或预检 |
| 上传 TestFlight | `upload_to_testflight` / `pilot` | 上传构建并附上中文更新说明，默认不外部分发 |
| 管理商店元数据 | `deliver`、`check_app_store_metadata` / `precheck` | 校验多语言元数据、上传元数据、提交审核 |
| 生成商店截图 | `snapshot` | 通过稳定的 UI Test 批量生成多语言、多设备截图 |
| 读写 Git 信息 | `ensure_git_status_clean`、`add_git_tag` | 发布时检查工作树与 tag，避免“代码和包不是同一份” |

你不必一口气把所有功能都接入。一个健康的演进顺序通常是：

```text
先测试 → 再归档 → 再管理签名 → 再上传 TestFlight
→ 再管理元数据和截图 → 最后才是审核提交与 CI
```

每往前走一步，先确保本机可复现，再交给 CI。这样当问题出现时，你能知道是工程问题、签名问题，还是 CI 环境问题。

## 三、接入前先准备什么

Fastlane 本身安装很简单，但 iOS 发布所依赖的外部信息不少。先把它们分成三类，会清楚很多。

### 1. 工程内必须明确的信息

至少要确认下面这些值，而不是在 Fastfile 里“猜”。

| 字段 | 例子 | 用途 |
|---|---|---|
| Xcode 工程或 Workspace | `HabitPulse.xcodeproj` | 测试、Archive 的入口 |
| Scheme | `HabitPulse` | 告诉 xcodebuild 构建哪个方案 |
| 主 App Bundle ID | `com.example.habitpulse` | 签名和 App Store Connect 的应用标识 |
| 所有 Extension Bundle ID | `com.example.habitpulse.Widget` | Widget、Share Extension 等也要签名 |
| Apple Team ID | `ABCDE12345` | 导出选项、签名校验 |
| 版本号与构建号策略 | `1.0` / `42` | App Store 与 TestFlight 要求 build number 唯一 |
| 测试设备与 Runtime | `iPhone 17 Pro (26.3)` | 使本机和 CI 的测试一致 |

最容易漏掉的是 Extension。只要工程里有 Widget、Share Extension、Notification Service Extension 等 Target，签名就不是“主 App 一个 Bundle ID”这么简单。HabitPulse 的 `APP_IDENTIFIERS` 显式包括主 App 和 Widget：

```ruby
APP_IDENTIFIERS = [
  "com.example.habitpulse",
  "com.example.habitpulse.Widget"
].freeze
```

在 Apple Developer 后台、match、Fastfile、CI 签名校验中，都要以这份完整列表为准。

### 2. 本地工具链

HabitPulse 把 Ruby、Fastlane 和 Bundler 版本锁进 `Gemfile`：

```ruby
ruby "~> 3.3.0"

source "https://rubygems.org"

gem "fastlane", "2.238.0"
gem "bundler", "4.0.16"
```

第一次接入时，在项目根目录运行：

```sh
bundle install
bundle exec fastlane --version
```

此外你还需要：

- macOS 和可用的 Xcode；
- 项目最低系统版本所需的 iOS Simulator Runtime；
- 已登录并有相应权限的 Apple Developer 账号（首次准备签名材料时需要）；
- Git 可访问私有证书仓库；
- 如果计划上传 TestFlight 或提交审核，还要有 App Store Connect API Key。

### 3. 机密字段：哪些可以进仓库，哪些绝对不行

下面这张表很重要。

| 内容 | 是否可提交到业务仓库 | 合理位置 |
|---|---|---|
| `Fastfile`、`Gemfile`、元数据模板 | 可以 | 业务仓库 |
| Bundle ID、Scheme、非敏感 Team ID | 通常可以 | Fastfile/xcconfig |
| match 证书仓库地址 | 可视仓库权限而定，私有项目通常可以 | `Matchfile` 或 Fastfile |
| match 加密密码 | 不可以 | 本机安全环境变量、CI Secret |
| `.p12`、Provisioning Profile | 不可以直接放业务仓库 | match 加密仓库 |
| App Store Connect `.p8` 私钥 | 绝对不可以 | 密码管理器、本机安全路径、CI Secret |
| Apple ID 密码 | 不建议作为自动化凭据 | 尽量改用 App Store Connect API Key |
| App Review 联系人信息 | 不应进公共仓库 | 本机受限文件或 CI Secret |

HabitPulse 提供了被 `.gitignore` 忽略的本地环境变量模板：

```dotenv
# fastlane/.env（仅本机使用，不提交）
ASC_KEY_ID=
ASC_ISSUER_ID=
ASC_KEY_FILEPATH=
ASC_REVIEW_INFO_FILEPATH=
```

其中 `ASC_KEY_FILEPATH` 指向本机的 `.p8` 文件；文件权限也应该收紧：

```sh
chmod 600 /安全路径/AuthKey_XXXXXXXXXX.p8
```

## 四、开始接入：初始化和目录结构

在一个已有 iOS 项目的根目录执行：

```sh
bundle exec fastlane init
```

Fastlane 会引导创建 `fastlane/` 目录。实际项目里，建议把目录逐步整理成这样：

```text
fastlane/
├── Fastfile                         # 各条自动化 lane 的核心定义
├── Appfile                          # App 标识等基础配置（按需）
├── Matchfile                        # match 的默认签名仓库配置（按需）
├── .env.example                     # 环境变量模板，不含真实密钥
├── review_information.example.yml   # 审核联系信息模板，不含真实个人资料
├── metadata/                        # App Store 多语言文案
│   ├── zh-Hans/
│   ├── en-US/
│   └── fr-FR/
├── screenshots/                     # snapshot 生成的截图
├── test_output/                     # JUnit、xcresult、测试日志
├── build_output/                    # IPA、archive、dSYM、构建 xcresult
└── release_notes/                   # 自动生成的发布说明
```

其中 `Fastfile` 是核心。Fastlane 里的可执行单元叫 **lane**，你可以把它理解为一条命名的自动化任务。

```ruby
default_platform(:ios)

platform :ios do
  desc "运行 iOS 测试"
  lane :test do
    run_tests(
      project: "HabitPulse.xcodeproj",
      scheme: "HabitPulse"
    )
  end
end
```

上面定义之后，运行方式是：

```sh
bundle exec fastlane ios test
```

`ios` 对应 `platform :ios`，`test` 对应 `lane :test`。lane 不一定要复杂；把一个动作做清楚、可单独验证，比写一条无所不包的“全自动超级 lane”更容易维护。

## 五、先做好公共配置和前置校验

真实项目中，最开始的 Fastfile 很容易充满散落的字符串：工程路径写一遍、Scheme 写一遍、Bundle ID 写一遍、输出路径又写一遍。HabitPulse 在接入的第一阶段把它们抽成常量：

```ruby
REPOSITORY_ROOT = File.expand_path("..", __dir__)
PROJECT = File.join(REPOSITORY_ROOT, "HabitPulse.xcodeproj")
SCHEME = "HabitPulse"
DEFAULT_TEAM_ID = "ABCDE12345"
APP_IDENTIFIERS = [
  "com.example.habitpulse",
  "com.example.habitpulse.Widget"
].freeze
APP_IDENTIFIER = APP_IDENTIFIERS.first
OUTPUT_ROOT = File.join(REPOSITORY_ROOT, "fastlane/build_output/ios/habitpulse")
TEST_OUTPUT_ROOT = File.join(REPOSITORY_ROOT, "fastlane/test_output")
TEST_DEVICE = "iPhone 17 Pro (26.3)"
```

然后给各 lane 一个统一的入口校验：

```ruby
private_lane :verify_fastlane_environment do |options|
  ensure_bundle_exec
  UI.user_error!("找不到 Xcode 工程：#{PROJECT}") unless File.exist?(PROJECT)

  if options[:requires_match_password] && ENV["MATCH_PASSWORD"].to_s.empty?
    UI.user_error!("缺少 MATCH_PASSWORD，无法解密 match 证书仓库")
  end

  team_id = ENV["FASTLANE_TEAM_ID"] || DEFAULT_TEAM_ID
  UI.user_error!("FASTLANE_TEAM_ID 必须是 10 位 Apple Team ID") unless team_id.match?(/\A[A-Z0-9]{10}\z/)
end
```

这类“先失败”的设计看似啰嗦，其实能省很多时间。比如 `.p8` 路径填错、忘记设置 `MATCH_PASSWORD`、用错 Ruby 环境时，应该在一分钟内给出清晰错误，而不是等五分钟 Archive 后才报一个难懂的签名失败。

## 六、测试自动化：从能跑到可排查

Fastlane 的测试 action 通常是 `run_tests`，也常被称为 `scan`。HabitPulse 的测试 lane 做了几件比“直接跑测试”更完整的事：清理上次报告、固定设备、输出日志、输出 JUnit、输出 `.xcresult`。

```ruby
lane :test do
  verify_fastlane_environment

  FileUtils.rm_rf(TEST_OUTPUT_ROOT)
  FileUtils.mkdir_p(File.join(TEST_OUTPUT_ROOT, "logs"))

  run_tests(
    project: PROJECT,
    scheme: SCHEME,
    devices: [TEST_DEVICE],
    ensure_devices_found: true,
    clean: true,
    output_directory: TEST_OUTPUT_ROOT,
    output_types: "junit",
    output_files: "HabitPulseTests.junit",
    buildlog_path: File.join(TEST_OUTPUT_ROOT, "logs"),
    result_bundle: true,
    result_bundle_path: File.join(TEST_OUTPUT_ROOT, "HabitPulseTests.xcresult"),
    xcargs: SWIFT_SANDBOX_XCARGS
  )
end
```

运行后重点看这些产物：

| 产物 | 作用 |
|---|---|
| `HabitPulseTests.xcresult` | 用 Xcode 打开后可查看失败用例、附件与测试层级 |
| `HabitPulseTests.junit` | 适合 CI 系统解析和统计 |
| `logs/` | 最接近原始 `xcodebuild` 输出，签名或编译失败时最有用 |

### 为什么要固定模拟器

“在我的电脑上能跑”不等于 CI 上能跑。设备型号、iOS Runtime、Xcode 版本不同，都可能让 UI 测试出现布局、权限弹窗或系统行为差异。HabitPulse 固定到 `iPhone 17 Pro (26.3)`，并在 CI 安装相同 Xcode 版本。

不要盲目照搬这个型号和版本；你应该选择你本机与 CI 都可获得的一组组合。关键原则是：**把它当作工程配置维护，而不是临时参数。**

## 七、iOS 签名到底在管理什么

这是 Fastlane 接入最容易卡住、也最值得理解的部分。

### 1. 四个容易混淆的东西

| 名称 | 它证明什么 | 通常在哪里 |
|---|---|---|
| App ID / Identifier | 这个 Bundle ID 属于你的开发团队 | Apple Developer 后台 |
| Certificate（证书） | 谁可以为团队签名 | Apple Developer 后台与本机 Keychain |
| Private Key（私钥） | 证书真正可用于签名的关键 | 创建证书的 Keychain；必须严格保护 |
| Provisioning Profile | 某个 App ID、证书、能力、设备或分发方式的组合许可 | Apple 后台和本机 |

可以这样粗略理解：Bundle ID 是“这是谁的 App”，证书和私钥是“谁有权签名”，Profile 是“这份签名允许以什么方式安装或分发”。

因此，一个包含 Widget 的 App Store Archive 至少要确保：主 App 和 Widget 都有正确的 App ID、所需 Capability、相应的签名证书和正确类型的 Profile。

### 2. 为什么不建议手工到处导入 `.p12` 和 Profile

小项目刚开始时，手动导入也能用。但一旦需要换电脑、多人协作或者接入 CI，你会遇到：

- 私钥只在某一台老电脑里，其他机器虽然有证书却无法签名；
- Profile 到期后，大家不知道该谁更新；
- 新增 Widget 后，主 App 能编译、Extension 没有 Profile；
- CI 缺证书、缺 Keychain、缺 Profile，错误信息还不直观；
- 有人误点了“重新生成”，导致其他人的签名材料失效。

这正是 `match` 的用武之地。

## 八、使用 match 管理证书和 Profile

Fastlane match 的核心思路是：把证书、私钥和 Profile 放进**独立的私有 Git 仓库**，并且加密保存。业务项目不保存敏感签名文件，只保存“去哪里拿、拿哪些 Bundle ID”的规则。

```text
业务仓库
  └── Fastfile：声明需要哪些 Bundle ID、什么分发类型

私有 match 仓库（加密）
  └── Certificates / Profiles / 加密材料

本机或 CI
  └── match 下载并解密 → 导入临时 Keychain → xcodebuild 签名
```

### 1. 初始化 match

初次使用可执行：

```sh
bundle exec fastlane match init
```

之后配置 `Matchfile` 或在 Fastfile 中传入仓库地址。脱敏示例：

```ruby
MATCH_GIT_URL = "git@github.com:example/ios-certificates.git"
MATCH_GIT_BRANCH = "main"
```

首次创建或迁移证书，需要在受控的开发机上执行相应 match 命令，并登录有 Apple Developer 权限的账号。完成后，日常开发机和 CI 应优先使用只读模式。

### 2. 为开发版和 App Store 版分别定义 lane

HabitPulse 这样定义：

```ruby
lane :habitpulse_dev do
  verify_fastlane_environment(requires_match_password: true)
  match(
    type: "development",
    app_identifier: APP_IDENTIFIERS,
    git_url: MATCH_GIT_URL,
    git_branch: MATCH_GIT_BRANCH,
    readonly: true
  )
end

lane :habitpulse_appstore do
  verify_fastlane_environment(requires_match_password: true)
  match(
    type: "appstore",
    app_identifier: APP_IDENTIFIERS,
    git_url: MATCH_GIT_URL,
    git_branch: MATCH_GIT_BRANCH,
    readonly: true
  )
end
```

分别运行：

```sh
bundle exec fastlane ios habitpulse_dev
bundle exec fastlane ios habitpulse_appstore
```

`development` 用于开发安装，`appstore` 用于 TestFlight 和 App Store 分发。不要拿开发 Profile 去导出 App Store IPA，也不要为了“让 CI 过”随意切成 automatic signing；这样虽然一时可能绕过去，后续会更难追踪。

### 3. 为什么 `readonly: true` 是 CI 的正确默认值

CI 应该是一个消费者，而不是证书的管理员。

```ruby
readonly: true
```

意味着 CI 只会下载、解密、安装已有材料。如果材料缺失或过期，它会明确失败，而不会擅自新建、吊销或替换团队证书。需要修复签名资产时，应该由有权限的人在受控的本机环境中处理，再把更新后的加密材料提交进 match 仓库。

特别不要在日常 CI 里用 `match --force`。它可能导致重新创建甚至影响既有材料；只有明确诊断到证书损坏、换新机等场景，才应由负责人谨慎执行。

### 4. match 需要哪些秘密

通常至少有两个：

| 名称 | 含义 |
|---|---|
| `MATCH_PASSWORD` | 解密 match 仓库中材料的密码 |
| 证书仓库读取凭据 | 例如只读 Deploy Key 对应的 SSH 私钥 |

`MATCH_PASSWORD` 不是 Apple ID 密码。它是你为 match 加密仓库设置的密码，应该存放在密码管理器、本机安全环境变量或 CI Secret 中。

## 九、归档和导出 IPA：build_app 的正确用法

Fastlane 中的 `build_app`（别名 `gym`）负责把 `xcodebuild archive` 和导出 IPA 的步骤封装起来。HabitPulse 将开发版和 App Store 版拆开，输出目录也拆开，避免产物互相覆盖。

### 1. 开发版打包

```ruby
lane :habitpulse_build_dev do
  habitpulse_dev

  build_app(
    project: PROJECT,
    scheme: SCHEME,
    configuration: "Debug",
    output_directory: File.join(OUTPUT_ROOT, "dev"),
    output_name: "HabitPulse_dev",
    export_method: "development",
    export_options: {
      teamID: ENV["FASTLANE_TEAM_ID"] || DEFAULT_TEAM_ID,
      compileBitcode: false
    },
    clean: true,
    archive_path: File.join(OUTPUT_ROOT, "dev", "HabitPulse.xcarchive"),
    result_bundle: true,
    result_bundle_path: File.join(OUTPUT_ROOT, "dev", "HabitPulse.xcresult"),
    xcargs: SWIFT_SANDBOX_XCARGS
  )
end
```

运行：

```sh
bundle exec fastlane ios habitpulse_build_dev
```

它输出一个 development IPA，适合受 Profile 限制的开发安装场景。

### 2. App Store 归档

```ruby
lane :habitpulse_archive_appstore do
  habitpulse_appstore

  version = get_version_number(xcodeproj: PROJECT, target: SCHEME)
  build_number = ENV["BUILD_NUMBER"] || get_build_number(xcodeproj: PROJECT)
  increment_build_number(build_number: build_number, xcodeproj: PROJECT) if ENV["BUILD_NUMBER"]

  build_app(
    project: PROJECT,
    scheme: SCHEME,
    configuration: "Release",
    output_directory: File.join(OUTPUT_ROOT, "appstore"),
    output_name: "HabitPulse",
    export_method: "app-store",
    export_options: {
      teamID: ENV["FASTLANE_TEAM_ID"] || DEFAULT_TEAM_ID,
      uploadSymbols: true,
      uploadBitcode: false
    },
    clean: true,
    archive_path: File.join(OUTPUT_ROOT, "appstore", "HabitPulse.xcarchive"),
    result_bundle: true,
    result_bundle_path: File.join(OUTPUT_ROOT, "appstore", "HabitPulse.xcresult"),
    xcargs: SWIFT_SANDBOX_XCARGS
  )
end
```

执行时可以临时传入构建号：

```sh
BUILD_NUMBER=42 bundle exec fastlane ios habitpulse_archive_appstore
```

这条 lane 的一个重要边界是：**它只归档和导出 IPA，不上传。**把“构建一个可发布包”和“把包发到外部平台”拆开后，日常验证更安全，CI 也能稳定保留一份主干可发布的制品。

### 3. 为什么要同时保留 IPA、archive、dSYM 和 xcresult

| 产物 | 为什么要留 |
|---|---|
| `.ipa` | 上传 TestFlight 或给分发渠道使用 |
| `.xcarchive` | 复查归档内容、签名和 dSYM 的来源 |
| `.dSYM` | 日后崩溃日志符号化需要 |
| `.xcresult` | 构建或测试失败时排查 Xcode 级别细节 |

HabitPulse 统一输出到：

```text
fastlane/build_output/ios/habitpulse/dev/
fastlane/build_output/ios/habitpulse/appstore/
fastlane/test_output/
```

在 CI 中，这些目录还会作为 Artifact 上传。不要只保留一份 IPA；真遇到问题时，archive 与完整日志往往更有价值。

## 十、版本号、构建号和发布说明

### 1. 先分清两个版本字段

| 字段 | Xcode 中常见键 | 例子 | 规则 |
|---|---|---|---|
| 面向用户的版本 | `MARKETING_VERSION` / `CFBundleShortVersionString` | `1.0` | 只有产品版本变化时才增加 |
| 构建号 | `CURRENT_PROJECT_VERSION` / `CFBundleVersion` | `42` | 每次上传到 TestFlight 必须唯一且递增 |

很多“上传被拒绝”的问题都来自 build number 重复。HabitPulse 的思路是先读取本地构建号和 App Store Connect 上已经存在的最大值，再取更大的下一个值：

```ruby
private_lane :next_testflight_build_number do
  local_build = get_build_number(xcodeproj: PROJECT).to_i
  uploaded_build = latest_testflight_build_number(
    api_key: lane_context[SharedValues::APP_STORE_CONNECT_API_KEY],
    app_identifier: APP_IDENTIFIER,
    initial_build_number: 0
  ).to_i
  [local_build + 1, uploaded_build + 1].max
end
```

使用：

```sh
bundle exec fastlane ios next_build_number
```

### 2. 从 Conventional Commits 生成发布说明

把提交信息规范化后，发布说明不必每次手写。HabitPulse 只提取 `feat`、`fix`、`perf`，过滤 `docs`、`chore` 等不面向用户的提交：

```ruby
subjects = sh("git log --no-merges --pretty=format:%s #{previous_tag}..HEAD").lines.map(&:strip)
notes = subjects.filter_map do |subject|
  next unless subject.match?(/^(feat|fix|perf)(\([^)]*\))?:/)
  "- #{subject.sub(/^(feat|fix|perf)(\([^)]*\))?:\s*/, "")}"
end
```

例如提交：

```text
feat(reminder): 新增提醒管理页面
fix(widget): 修复快照刷新延迟
docs(readme): 补充安装说明
```

生成的面向用户说明是：

```text
- 新增提醒管理页面
- 修复快照刷新延迟
```

这不是让开发者偷懒，而是让提交信息成为可复用的工程记录。前提是提交标题真的描述了用户能感知的变化。

### 3. 用 tag 约束正式发布

HabitPulse 的发布 tag 采用：

```text
v<版本号>-build.<构建号>
```

例如：

```text
v1.0-build.42
```

发布 lane 会确认当前 `HEAD` 精确对应这个 tag。这样你不会出现“本来准备发布 A 提交，实际打出来的是后来又混进几个提交的 B 包”。

`release_prepare` 默认是 dry-run：会计算下一构建号和生成发布说明，但不会修改工程，也不会创建 tag。只有明确设置确认变量才会写入：

```sh
# 默认只预览
bundle exec fastlane ios release_prepare

# 确认后才会修改 build number 并创建 tag
CONFIRM_RELEASE_PREPARE=true bundle exec fastlane ios release_prepare
```

## 十一、接入 App Store Connect API Key

过去 Fastlane 经常依赖 Apple ID 登录、双重验证和会话 Cookie。现在更推荐 App Store Connect API Key：它适合本机与 CI，也更容易收紧权限和撤销。

### 1. 需要准备的三个字段

| 环境变量 | 含义 |
|---|---|
| `ASC_KEY_ID` | App Store Connect 创建 API Key 后的 Key ID，通常是 10 位字符 |
| `ASC_ISSUER_ID` | App Store Connect 页面提供的 Issuer ID，UUID 格式 |
| `ASC_KEY_FILEPATH` | 本机 `.p8` 私钥文件的绝对路径 |

`.p8` 文件创建后通常只有一次下载机会。下载后立即保存到密码管理器或受保护路径，绝对不要提交到 Git。

Fastfile 可以做格式和权限检查：

```ruby
private_lane :app_store_connect_api_key_from_environment do
  key_id = ENV["ASC_KEY_ID"]
  issuer_id = ENV["ASC_ISSUER_ID"]
  key_filepath = ENV["ASC_KEY_FILEPATH"]

  UI.user_error!("ASC_KEY_ID 格式错误") unless key_id&.match?(/\A[A-Z0-9]{10}\z/)
  UI.user_error!("ASC_ISSUER_ID 格式错误") unless issuer_id&.match?(/\A[0-9a-fA-F-]{36}\z/)
  UI.user_error!("缺少 .p8 私钥") unless key_filepath&.end_with?(".p8") && File.file?(key_filepath)
  UI.user_error!("私钥权限过宽") unless (File.stat(key_filepath).mode & 0o077).zero?

  app_store_connect_api_key(
    key_id: key_id,
    issuer_id: issuer_id,
    key_filepath: key_filepath,
    duration: 1_200,
    in_house: false
  )
end
```

先做一个只读验证，比直接上传更稳：

```sh
bundle exec fastlane ios asc_verify
```

它能先确认 Key 是否有效、权限是否足够、网络和配置是否正确。

### 2. API Key 与签名证书不是同一回事

这是新手常混淆的点：

- `match` 管理 **Apple Developer 签名证书和 Provisioning Profile**；
- App Store Connect API Key 管理 **读取 TestFlight、上传构建、上传元数据、提交审核** 的权限。

一个解决“我能不能签出这个 IPA”，另一个解决“我能不能把 IPA 传到 App Store Connect”。两者都需要，但职责不能互相替代。

## 十二、上传 TestFlight：先上传，不自动外发

HabitPulse 的 `beta` lane 做的顺序是：计算下一个构建号 → 生成发布说明 → 安装 App Store 签名材料 → Release Archive → 验证 IPA/dSYM/Bundle ID → 上传 TestFlight。

核心上传部分如下：

```ruby
upload_to_testflight(
  ipa: ipa_path,
  api_key: lane_context[SharedValues::APP_STORE_CONNECT_API_KEY],
  app_identifier: APP_IDENTIFIER,
  changelog: File.read(notes_path),
  skip_submission: true,
  distribute_external: false,
  notify_external_testers: false,
  skip_waiting_for_build_processing: false,
  wait_processing_timeout_duration: 600
)
```

执行：

```sh
bundle exec fastlane ios beta
```

几个参数的含义：

- `skip_submission: true`：不自动把构建提交给 Beta Review；
- `distribute_external: false`：默认不外部分发；
- `notify_external_testers: false`：不自动向外部测试者发通知；
- `wait_processing_timeout_duration: 600`：最多等待十分钟处理状态，避免无限挂起。

这种默认值更保守。第一次跑通 Fastlane 或新换 API Key 时，最适合先上传到 TestFlight、确认构建存在，再决定是否分发给外部测试人员。

### 上传前为什么要验证产物

归档成功不等于导出的东西完全正确。HabitPulse 在上传前会检查：

- IPA 文件是否存在；
- archive 是否存在；
- archive 中主 App 与 Widget 的 dSYM 是否齐全；
- 导出的 IPA 是否为 distribution 签名；
- Team ID 和 Bundle ID 是否符合预期。

这个额外步骤能把“上传后才发现签错包”的问题提前到本机解决。

## 十三、商店元数据、预检和截图

Fastlane 的发布能力不只有传 IPA。App Store 名称、副标题、描述、关键词、支持链接、隐私链接、版权、年龄分级、截图等也可以作为版本化文件管理。

### 1. 多语言元数据

HabitPulse 将三种语言放在：

```text
fastlane/metadata/zh-Hans/
fastlane/metadata/en-US/
fastlane/metadata/fr-FR/
```

每个目录包含类似：

```text
name.txt
subtitle.txt
promotional_text.txt
description.txt
keywords.txt
support_url.txt
marketing_url.txt
privacy_url.txt
copyright.txt
```

本地校验 lane 检查文件是否完整、内容是否为空、URL 是否为 HTTPS、字符数是否超出 App Store 限制：

```sh
bundle exec fastlane ios metadata_validate
```

这个命令不需要上传，也不需要读取私钥，适合放进 PR 检查。

### 2. precheck 与 metadata_upload

两者职责不同：

```sh
# 与 App Store Connect 做预检查：不上传、不提交审核
bundle exec fastlane ios precheck

# 只上传商店文案：不上传二进制、不传截图、不提交审核
bundle exec fastlane ios metadata_upload
```

这样可以把“修改文案”和“上传一个新的 App 二进制”分开。真实实践中，App Store Connect 的首个版本审核详情接口偶尔可能返回空数据，导致元数据上传后出现非关键的清理异常。HabitPulse 的 lane 对已知、可确认的 `No data` 情况做了窄范围处理：确认元数据已经上传后跳过非必要的审核附件清理；其他错误仍然抛出，不能一概吞掉。

### 3. snapshot 自动截图

商店截图最适合由 UI Test 自动化：稳定、可重复、多语言和多尺寸不会漏。HabitPulse 的 `screenshots` lane：

```sh
# 默认生成简体中文、英语、法语截图
bundle exec fastlane ios screenshots

# 只生成英文和法语，便于调试
SCREENSHOT_LANGUAGES=en-US,fr-FR bundle exec fastlane ios screenshots
```

它指定了 UI Test Target、设备、语言和启动参数：

```ruby
snapshot(
  scheme: SCHEME,
  test_target_name: "HabitPulseUITests",
  devices: ["iPhone 17 Pro", "iPad Pro 13-inch (M5)"],
  languages: (ENV["SCREENSHOT_LANGUAGES"] || "zh-Hans,en-US,fr-FR").split(","),
  launch_arguments: ["--ui-testing", "--skip-onboarding"],
  headless: true,
  concurrent_simulators: false
)
```

截图自动化的重点不在 Fastlane 本身，而在你的 App 必须有可预测的 UI Test 状态：不要读取真实 iCloud 数据，不要在截图过程中弹不可控权限框，不要让首次引导随机出现。必要时给 App 增加仅限 UI Test 的启动参数和种子数据。

## 十四、正式审核提交：必须有安全闸门

正式提交审核是外部状态变更，不能和普通 Archive 混在一起。HabitPulse 的 `release` lane 采取了两层保护：

1. 当前提交必须精确对应 `v<version>-build.<build>` tag；
2. 只有 `CONFIRM_RELEASE=true` 才真的上传 IPA、上传元数据、提交审核。

```sh
# 默认 dry-run：测试、归档、precheck，但不上传、不提交
bundle exec fastlane ios release

# 显式确认后才允许外部动作
CONFIRM_RELEASE=true bundle exec fastlane ios release
```

dry-run 不是摆设。HabitPulse 实测过：在没有正确发布 tag 的情况下，即使设置了 `CONFIRM_RELEASE=true`，lane 会在任何测试、归档、网络上传之前以退出码 1 阻断。

真正调用 `deliver` 时还会保持：

```ruby
submit_for_review: true,
automatic_release: false
```

这表示提交审核可以自动化，但审核通过后仍需你在 App Store Connect 手动选择公开发布。对于大多数独立开发者和小团队，这个默认值既高效又稳妥。

## 十五、这次接入中遇到的真实问题与解决方案

下面这些不是“理论上可能发生”的问题，而是 HabitPulse 在接入 Fastlane 与后续 CI 时实际验证过或遇到过的情况。

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

```ruby
SWIFT_SANDBOX_XCARGS = 'OTHER_SWIFT_FLAGS="-disable-sandbox"'
```

然后将它放进 `run_tests`、开发 Archive、App Store Archive、截图等所有需要 xcodebuild 的 lane。关键不是这一个参数本身，而是不要只在某一条命令里临时补上；否则本机测试通过、CI 或 release 又失败，问题会反复出现。

### 问题 2：构建工具链版本漂移

**现象**

本地能跑，GitHub Actions 上找不到模拟器、Ruby 依赖不兼容，或者 Xcode 版本不满足项目要求。

**原因**

没有锁定 Xcode、Ruby、Fastlane 以及测试 Runtime；GitHub-hosted Runner 会更新，系统全局 Ruby/Gem 也会变化。

**解决方案**

- 用 `Gemfile` 与 `Gemfile.lock` 固定 Ruby/Fastlane/Bundler；
- 本机和 CI 都使用 `bundle exec`；
- Fastlane 固定测试设备和 Runtime；
- GitHub Actions 使用 `setup-xcode` 显式选择 Xcode 版本；
- 升级 Xcode 前先在分支或 PR 流程验证。

### 问题 3：Widget 导致签名失败

**现象**

主 App 看起来已经有证书和 Profile，但 Archive 报 Extension 没有合适的 Provisioning Profile，或导出时签名失败。

**原因**

只把主 App Bundle ID 配给了 match，漏了 Widget 的 Bundle ID；或者 Apple Developer 后台没有为 Extension 正确配置 App ID/Capability。

**解决方案**

把所有需要签名的 Target 列成常量：

```ruby
APP_IDENTIFIERS = [main_app_bundle_id, widget_bundle_id].freeze
```

并让 development/appstore 两条 match lane 都使用同一列表。新增任何 Extension 时，把 Apple Developer、match、Fastfile、CI 签名检查这四处一起更新。

### 问题 4：TestFlight 提示构建号重复

**现象**

IPA 签名正确，但上传被 App Store Connect 拒绝，提示 build number 已存在。

**原因**

只从工程本地读取 `CURRENT_PROJECT_VERSION`，忽略了已经上传到 TestFlight 的构建号；或者并行构建时重复使用同一个号。

**解决方案**

在上传前查询 App Store Connect 的最新 build number，并与本地号比较取更大值的下一个数。对于频繁并行发布的团队，还应在发布流程中串行化或用外部构建号策略协调，避免两个 Job 同时算到相同结果。

### 问题 5：把 Fastlane CLI 的 `--dry-run` 当作业务 dry-run

**现象**

尝试运行：

```sh
bundle exec fastlane ios release --dry-run
```

结果失败或没有达到预期。

**原因**

“是否产生上传/提交审核等外部副作用”是你的 release lane 的业务规则，不是 Fastlane CLI 自动知道的概念。

**解决方案**

在 Fastfile 内设计明确的确认变量，例如：

```ruby
unless ENV["CONFIRM_RELEASE"] == "true"
  UI.important("dry-run：未上传 IPA、未上传元数据、未提交审核")
  next
end
```

默认执行 `bundle exec fastlane ios release` 时只做测试、归档与预检；只有显式设置 `CONFIRM_RELEASE=true` 才进入 `deliver`。这才是可审计、可测试的 dry-run。

### 问题 6：Fastlane 把 README 自动重写

**现象**

运行某些 Fastlane 命令或文档生成动作后，已跟踪的 `fastlane/README.md` 被生成内容覆盖，导致工作树出现不属于当前任务的改动。

**原因**

Fastlane 的默认文档生成行为会产出 lane 列表；如果你把 README 当成手写操作手册，两者会冲突。

**解决方案**

明确 README 的所有权：要么接受自动生成、不要手写；要么将手写使用说明放到独立文件并避免生成覆盖。HabitPulse 的实践是先检查 diff，确认它只是生成副作用后恢复到提交版本，避免把无关改动混进功能提交。

### 问题 7：商店预检出现版权字段警告

**现象**

本地三语 `copyright.txt` 已包含年份，App Store Connect 的 `precheck` 仍给出版权相关警告。

**原因**

预检有时检查的是远端当前商店元数据或特定 App Store Connect 规则，未必等同于本地文本文件格式。

**解决方案**

先区分 warning 与 error：本地元数据校验是否已经通过？远端 warning 是否阻断提交？如果不阻断，不要为了消除一条远端提示盲目来回修改已经正确的本地文本。将它记录为非阻断项，等真正上传并同步远端元数据后再复查。

### 问题 8：审核联系信息在本机有，CI 却没有

**现象**

本机 `release` 能找到 `review_information.yml`，GitHub Actions 的正式 release 因缺少审核联系人、电话或审核备注失败。

**原因**

审核信息应该不提交到仓库，所以 CI Runner 天然没有这个文件。

**解决方案**

将审核信息作为受保护 Environment Secret 注入到 Runner 临时文件，或为 CI 单独配置受保护的安全文件路径；并让 lane 在缺失时明确报错。不要把真实电话、邮箱、登录说明直接提交进 `fastlane/`。

### 问题 9：Fastlane 的 lane 名与内置工具重名

**现象**

运行时出现类似提示：`Lane name 'precheck' should not be used because it is the name of a fastlane tool`。

**原因**

lane 叫 `precheck`，同时 Fastlane 也有同名 action/tool。多数情况下仍可运行，但可读性和未来兼容性都不好。

**解决方案**

更推荐使用语义明确且不冲突的命名，例如：

```ruby
lane :store_precheck do
  check_app_store_metadata(...)
end
```

然后同步更新 README、CI 和团队命令。若历史项目暂时保留旧 lane，至少把这条 warning 记录下来，避免误认为是构建失败。

### 问题 10：证书仓库拉取失败或 GitHub 走错代理

**现象**

`match` 无法 clone 私有证书仓库，或 GitHub 请求报 502、超时、SSH permission denied。

**原因**

可能是 SSH 私钥没有读取权限、Deploy Key 没加到正确仓库、`known_hosts` 缺 GitHub 指纹，或者本机透明代理配置错误。

**解决方案**

- 先用 `ssh -T git@github.com` 或对目标仓库做只读连接诊断；
- 确保私钥权限是 `600`；
- CI 里显式设置 `GIT_SSH_COMMAND`，指定唯一的 Deploy Key；
- 本机代理异常时，使用已验证可用的代理做一次性 Git 覆盖，不要永久污染全局配置；
- 不要为了跳过问题把证书仓库改为公开。

## 十六、把 Fastlane 接进 GitHub Actions 后，职责如何划分

Fastlane 接入完成后，GitHub Actions 不必重复实现构建逻辑，只需调用 lane。HabitPulse 最终采用三段式：

```text
PR workflow
  └── bundle exec fastlane ios metadata_validate
  └── bundle exec fastlane ios test

main archive workflow
  └── 设置临时 SSH / Keychain / match 密码
  └── bundle exec fastlane ios habitpulse_archive_appstore

manual release workflow
  └── 设置临时 SSH / Keychain / App Store Connect API Key
  └── bundle exec fastlane ios beta 或 release
```

这样 Fastlane 是能力层，GitHub Actions 是调度层。你仍然可以在本机执行同一条 lane；CI 的差别只是它从 Secret 注入凭据，并把产物上传成 Artifact。

关于 GitHub Actions 的完整配置、Environment Secret 和临时 Keychain，可参阅同目录文章：[如何将 iOS 项目接入 GitHub Action](如何将iOS项目接入GitHub%20Action.md)。

## 十七、一份可照着执行的日常清单

### 日常开发

```sh
bundle install
bundle exec fastlane ios test
```

测试通过后再提交代码。不要等到发布日才第一次运行 Fastlane。

### 需要给测试人员一个包

```sh
# 仅本地归档，先检查产物
bundle exec fastlane ios habitpulse_archive_appstore

# 需要上传 TestFlight 时
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

### 准备正式审核

```sh
# 预览下一个构建号与发布说明
bundle exec fastlane ios release_prepare

# 显式准备版本和 tag（会修改工程与 Git）
CONFIRM_RELEASE_PREPARE=true bundle exec fastlane ios release_prepare

# 默认 dry-run：不上传、不提交审核
bundle exec fastlane ios release

# 最后确认后才真正提交审核
CONFIRM_RELEASE=true bundle exec fastlane ios release
```

正式提交前，还应确认：当前工作树干净、当前提交具有正确 tag、TestFlight 已验证、审核联系信息已准备好、隐私链接和商店元数据正确。

## 结语

Fastlane 最好的使用方式，不是追求“一条命令把所有事情都做掉”，而是把 iOS 交付里的每个高风险环节做成边界明确、可单独验证的 lane：

- `test` 只负责可靠测试和报告；
- `match` 只负责把受控签名材料安全地带到当前机器；
- `build_app` 只负责稳定归档和导出；
- `beta` 只负责 TestFlight；
- `release` 在 tag 与显式确认之后才允许提交审核；
- GitHub Actions 只负责合适的时机调用这些能力，并不重写业务规则。

当这些规则被写进仓库、被本机和 CI 共同执行后，发版就不再依赖某位同事记得住一套手工步骤。它变成了一条可阅读、可复现、可审计的工程流程——这才是 Fastlane 在 iOS 项目里真正长期有价值的地方。

