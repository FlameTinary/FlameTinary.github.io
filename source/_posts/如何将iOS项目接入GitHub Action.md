---
title: 如何将 iOS 项目接入 GitHub Action
date: 2026-09-07
tags:
  - iOS
  - GitHub-Actions
  - Fastlane
  - CI-CD
  - TestFlight
---

# 如何将 iOS 项目接入 GitHub Action

很多人第一次给 iOS 项目接 GitHub Actions，会以为就是“写一个 YAML，跑一下 `xcodebuild test`”。真开始做才会发现，真正麻烦的不是 YAML，而是签名证书、Provisioning Profile、App Store Connect API Key、TestFlight 上传权限，以及“怎样确保一次误操作不会直接把包提交审核”。

这篇文章以 HabitPulse 的真实落地过程为主线，讲清楚一个可发布 iOS 项目是怎样接入 GitHub Actions 的。它是一个 SwiftUI + SwiftData 项目，最低支持 iOS 17.6，包含主 App 和 Widget 扩展；构建和发布能力由 Fastlane 统一封装。文中所有仓库地址、Bundle ID、Team ID 和密钥均已脱敏。

最终得到的不是一个大而全、什么都自动跑的工作流，而是三条职责明确的流水线：PR 做质量检查，`main` 做受保护归档，发布只能人工发起。

```text
创建或更新 PR
    ↓
PR 检查：工程校验 + 单元测试 + 测试报告
    ↓
合并到 main
    ↓
主干归档：获取签名材料 + Release Archive + 保存构建产物
    ↓
在 Actions 页面手动选择 beta / release
    ↓
上传 TestFlight 或提交 App Store 审核（不会自动公开发布）
```

## 一、先定边界：GitHub Actions 不该替代 Fastlane

一个非常关键的分层是：**Fastlane 负责“如何构建和发布”，GitHub Actions 负责“什么时候、在哪台机器上调用 Fastlane”。**

如果把复杂的 `xcodebuild` 参数、签名逻辑、TestFlight 上传逻辑都塞进 YAML，后面你想在本机复现 CI 失败，往往会很痛苦。反过来，把这些动作都写成 Fastlane lane 后，本地和 CI 只需要执行同一条命令。

例如 HabitPulse 里的核心命令是：

```sh
# 固定模拟器运行测试，并输出 JUnit、xcresult 与日志
bundle exec fastlane ios test

# 使用 App Store 签名归档并导出 IPA、dSYM、archive；不上传
bundle exec fastlane ios habitpulse_archive_appstore

# 上传 TestFlight
bundle exec fastlane ios beta

# 正式提交审核；需要额外安全确认
CONFIRM_RELEASE=true bundle exec fastlane ios release
```

这样做有三个直接好处：

- 本地先跑通的命令，放到 CI 中通常也能跑通；
- workflow 文件更短，重点落在运行环境、权限和产物保存；
- 发版规则不会只藏在 GitHub YAML 中，开发者在本机也能看见、验证和使用。

本文后面的 YAML 都假设你已经有一个可在本机运行的 Fastlane 基础。至少应当锁定 Ruby 与 Bundler 版本，并将构建参数收进 `Fastfile`。

## 二、先把本地发布链路跑通

不要一上来就把问题交给 GitHub Actions。先在自己电脑上解决“能测试”“能归档”“能签名”这三件事，之后才值得把它搬到云端 Runner。

### 1. 锁定工具版本

GitHub 的 macOS Runner 会更新，Xcode 也会更新。没有版本约束时，今天能过的构建下周可能因为 SDK、Swift 或 Ruby 变化突然失败。

HabitPulse 实际固定了：

| 工具      |            CI 使用的版本 | 用途                    |
| ------- | ------------------: | --------------------- |
| Runner  |          `macos-15` | 提供 macOS 构建环境         |
| Xcode   |              `26.3` | 编译、测试、归档              |
| Ruby    |               `3.3` | 运行 Bundler 和 Fastlane |
| Bundler | 由 `Gemfile.lock` 锁定 | 保证 Fastlane 依赖一致      |

请按自己的工程调整 Xcode 与模拟器型号。尤其要注意：Xcode 版本必须包含你的 deployment target 所需的 SDK，以及 Fastlane 测试 lane 中指定的 iOS Simulator Runtime。

### 2. 把测试、构建和归档变成 lane

下面是简化后的思路。真实项目的 lane 通常还会包含日志、产物目录、参数校验和错误提示，但职责应该保持单一。

```ruby
platform :ios do
  lane :test do
    run_tests(
      project: "HabitPulse.xcodeproj",
      scheme: "HabitPulse",
      device: "iPhone 17 Pro (26.3)",
      result_bundle: true,
      result_bundle_path: "fastlane/test_output/HabitPulseTests.xcresult",
      output_types: "junit",
      output_files: "HabitPulseTests.junit"
    )
  end
end
```

HabitPulse 还有一个环境相关的细节：命令行执行 `xcodebuild` 时，Swift 宏插件在沙箱里可能启动失败，表现为 `could not be found for macro 'Model()'` 或 `swift-plugin-server produced malformed response`。项目将下面的参数统一收进 Fastlane，避免本机和 CI 行为不一致：

```ruby
SWIFT_SANDBOX_XCARGS = 'OTHER_SWIFT_FLAGS="-disable-sandbox"'
```

这不是所有项目都需要照抄的配置。正确做法是：先确认自己的命令行构建问题和原因，再把必要参数固定在构建入口，而不是散落在各个 workflow 里。

### 3. 使用 match 管理签名材料

iOS CI 最大的门槛通常是签名。开发证书、发布证书和 Profile 不能凭空出现在 GitHub Runner 上；而直接把 `.p12`、profile 文件放进业务仓库，既不安全也难维护。

HabitPulse 使用 Fastlane match：

- 签名证书和 Profile 存在一个独立、加密的私有 Git 仓库；
- 业务项目只记录 match 仓库地址和两个 Bundle ID；
- CI 使用 `readonly: true`，只下载、解密和安装，绝不在云端新建或吊销证书；
- 主 App 和 Widget 都必须写进 `app_identifier`，否则 Archive 时很容易出现“扩展没有 Profile”的错误。

示例：

```ruby
APP_IDENTIFIERS = [
  "com.example.HabitPulse",
  "com.example.HabitPulse.Widget"
].freeze

lane :appstore_signing do
  match(
    type: "appstore",
    app_identifier: APP_IDENTIFIERS,
    git_url: ENV.fetch("MATCH_GIT_URL"),
    readonly: true
  )
end
```

这里的 `readonly: true` 很重要。它让 CI 失败时只会失败，不会因为一次环境异常去重建或撤销团队的证书。

## 三、为什么要拆成三个 workflow

HabitPulse 没有用一个“万能 workflow”。它在 `.github/workflows/` 下有三个文件：

| 文件 | 触发条件 | 做什么 | 不做什么 |
|---|---|---|---|
| `pr.yml` | PR 的目标分支是 `main` | 工程完整性检查、元数据校验、测试、上传报告 | 不读取发布密钥，不签名，不上传 |
| `main-archive.yml` | 推送到 `main` | 取回签名材料，生成 App Store archive，保存 IPA/dSYM | 不上传 TestFlight，不提交审核 |
| `manual-release.yml` | 在 Actions 页面手动运行 | 选择 TestFlight 或正式审核动作 | 不会在普通 push 时自动运行 |

这个划分不是形式主义，而是权限分层。

PR 来自开发分支，最需要的是快速反馈和低权限；`main` 代表已经合并的代码，可以生成一份可追溯的正式归档；真正会改变外部状态的动作，例如上传 TestFlight、提交审核，必须由人明确点击触发。

### 命令行直接合并会触发什么？

这是一个很常见的误会。下面这种操作：

```sh
git checkout main
git merge --ff-only feature/my-change
git push origin main
```

触发的是 `push` 事件，因此会执行 `main-archive.yml`，**不会**执行只监听 `pull_request` 的 `pr.yml`。

如果你希望每次合并前一定经过 PR 检查，应在 GitHub 仓库里给 `main` 设置 Branch protection rule，把对应的 PR check 设为必需检查。否则，拥有直接推送权限的人仍然可以绕过 PR 流程。

## 四、PR 工作流：只做验证，不碰发布凭据

PR workflow 最好足够“干净”：即使来自 fork 的 PR，或者有人误改了分支，也不应接触发布证书、match 密码、App Store Connect 私钥。

HabitPulse 的核心版本如下：

```yaml
name: iOS PR

on:
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  test:
    runs-on: macos-15
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4

      - uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: '26.3'

      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true

      - name: Check project integrity
        run: |
          ruby -c fastlane/Fastfile
          xcodebuild -list -project HabitPulse.xcodeproj
          bundle exec fastlane ios metadata_validate

      - name: Run tests
        run: bundle exec fastlane ios test

      - name: Upload test reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: ios-pr-test-reports
          if-no-files-found: warn
          path: fastlane/test_output/
```

有几个小点很值得保留：

- `permissions: contents: read` 是最小权限。没有写入仓库、创建 Release 或读取 Secret 的权限；
- `timeout-minutes` 防止模拟器或依赖安装卡死，持续消耗 macOS Runner 时间；
- `if: always()` 保证测试失败后仍上传 `.xcresult`、JUnit 和日志。否则最需要排查材料的时候，反而什么都拿不到；
- `bundler-cache: true` 会缓存 Ruby gems，能明显减少重复安装 Fastlane 的时间。

除了测试，HabitPulse 还执行 `ruby -c fastlane/Fastfile`、`xcodebuild -list` 和商店元数据校验。它们都很轻量，但能及早发现 Fastfile 写错、Xcode Scheme 丢失、上架文案文件缺失这些“测试未必能覆盖”的问题。

## 五、主干归档：有签名，但不发布

一旦代码进入 `main`，HabitPulse 会自动生成 App Store 归档。这里的目标不是马上发布，而是确保主干在一个干净的 macOS 环境中**真的能被签名并导出**。

```yaml
name: iOS Main Archive

on:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  archive:
    runs-on: macos-15
    timeout-minutes: 45
    environment: appstore
    steps:
      - uses: actions/checkout@v4
      # 此处省略 Xcode、Ruby 安装步骤
      - name: Configure SSH for match
        # 将 MATCH_DEPLOY_KEY 写入 Runner 的临时文件
      - name: Configure temporary keychain
        # 创建、解锁临时钥匙串，并写入 MATCH_PASSWORD
      - name: Archive App Store build
        run: bundle exec fastlane ios habitpulse_archive_appstore
      - name: Upload archive artifacts
        if: always()
        uses: actions/upload-artifact@v4
```

### 1. 为什么要用 GitHub Environment

workflow 中的：

```yaml
environment: appstore
```

表示 Job 使用 GitHub 仓库的 `appstore` Environment。建议把发布相关 Secret 放在这里，而不是放在仓库级 Secrets：

- 能把发布凭据和普通 CI 配置隔离开；
- 可为 Environment 配置 required reviewers，使发布 Job 必须经过审批；
- 后续有 staging、production 等不同环境时，扩展更自然；
- GitHub Actions 页面能清楚看到某次 Job 使用了哪个环境。

在 GitHub 仓库中依次进入：`Settings → Environments → New environment`，创建 `appstore`；再进入这个 Environment 的 Secrets 页面添加后文的五个 Secret。

### 2. 让 match 仓库可被 Runner 读取

match 的证书仓库通常是私有仓库，Runner 需要一把专用 SSH Key 才能拉取它。workflow 会把密钥仅写入当前 Runner：

```yaml
- name: Configure SSH for match
  env:
    MATCH_DEPLOY_KEY: ${{ secrets.MATCH_DEPLOY_KEY }}
  run: |
    install -d -m 700 ~/.ssh
    printf '%s\n' "$MATCH_DEPLOY_KEY" > ~/.ssh/match_deploy_key
    chmod 600 ~/.ssh/match_deploy_key
    ssh-keyscan github.com >> ~/.ssh/known_hosts
    echo 'GIT_SSH_COMMAND=ssh -i ~/.ssh/match_deploy_key -o IdentitiesOnly=yes' >> "$GITHUB_ENV"
```

其中 `IdentitiesOnly=yes` 的作用是让 SSH 只尝试这一把部署密钥，避免 Runner 上其他身份造成不确定性。`ssh-keyscan` 是为了预先写入 GitHub 主机指纹，避免非交互式执行卡在“是否信任主机”。

### 3. 为什么还要临时 Keychain

下载到 match 仓库的证书不是放到文件夹里就能用于 codesign。macOS 的签名工具需要从 Keychain 中找到证书和私钥。因此每个 Runner Job 建一个临时 Keychain：

```yaml
- name: Configure temporary keychain
  env:
    MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
  run: |
    security create-keychain -p "$MATCH_PASSWORD" ci.keychain-db
    security set-keychain-settings -lut 21600 ci.keychain-db
    security unlock-keychain -p "$MATCH_PASSWORD" ci.keychain-db
    security list-keychains -d user -s ci.keychain-db login.keychain-db
    echo "MATCH_PASSWORD=$MATCH_PASSWORD" >> "$GITHUB_ENV"
```

这里把 Keychain 密码直接复用为 `MATCH_PASSWORD` 只是实践中方便的一种做法，不是硬性要求。更重要的原则是：它只在当前临时 Runner 存在，结束时删除。

最后无论构建成功还是失败，都应清理：

```yaml
- name: Remove temporary credentials
  if: always()
  run: |
    security delete-keychain ci.keychain-db || true
    rm -f ~/.ssh/match_deploy_key
```

GitHub-hosted Runner 本身是临时的，但显式清理仍然值得保留：代码的意图更清楚，也便于未来迁移到 self-hosted Runner。

### 4. 最隐蔽的 Keychain 坑：创建了临时钥匙串，不代表 match 会用它

这是一次真实踩坑：workflow 已经创建并解锁了 `ci.keychain-db`，但 Fastlane `match` 没有被显式告知该钥匙串名称和密码。于是 match 退回使用 `login.keychain`。证书和 Profile 看起来都“安装成功”，但 match 无法为登录钥匙串中的私钥设置 partition list，日志会出现类似内容：

```text
Keychain password for .../login.keychain-db was not specified
Could not configure imported keychain item ... to prevent UI permission popup
```

随后 `codesign` 在签名 App 或 Extension（本例是 Widget）时等待钥匙串权限弹窗；GitHub-hosted Runner 没有桌面可点击，进程会一直挂起，最后只留下误导性的 `The operation was canceled`，实际是 job 达到超时后被 Actions 终止。

修复的关键是把 Keychain 从 workflow 明确传递给 Fastlane，并由 `match` 显式使用它：

```yaml
- name: Configure temporary keychain
  run: |
    security create-keychain -p "$MATCH_PASSWORD" ci.keychain-db
    security unlock-keychain -p "$MATCH_PASSWORD" ci.keychain-db
    security list-keychains -d user -s ci.keychain-db login.keychain-db
    echo "MATCH_KEYCHAIN_NAME=ci.keychain-db" >> "$GITHUB_ENV"
    echo "MATCH_KEYCHAIN_PASSWORD=$MATCH_PASSWORD" >> "$GITHUB_ENV"
```

```ruby
match(
  type: "appstore",
  app_identifier: APP_IDENTIFIERS,
  readonly: true,
  keychain_name: ENV.fetch("MATCH_KEYCHAIN_NAME"),
  keychain_password: ENV.fetch("MATCH_KEYCHAIN_PASSWORD")
)
```

本机运行时不必强行复用 CI 临时钥匙串。一个实用做法是：只有环境变量存在时才传这两个参数；本机保留 match 默认的登录钥匙串，CI 则固定使用 `ci.keychain-db`。排查签名卡死时，优先看 match 最终摘要里的 `keychain_name`，并确认它与 workflow 创建的名称完全一致。

## 六、手动发布：把“可能产生外部影响”的动作锁起来

HabitPulse 的手动工作流只接受两个输入：选择 `beta` 或 `release`，再决定是否确认正式审核。

```yaml
on:
  workflow_dispatch:
    inputs:
      lane:
        description: '选择发布动作'
        required: true
        type: choice
        options: [beta, release]
      confirm_release:
        description: '仅正式审核选择 true；会上传并提交审核，不会自动公开发布'
        required: true
        default: false
        type: boolean
```

这里分两层保护：

1. GitHub Actions UI 要求你明确选择；
2. Fastlane 的 `release` lane 还会检查 `CONFIRM_RELEASE=true`。

也就是说，不能因为某次提交推到了 `main`，就自动把版本提交给 App Review。对于面向用户的应用，这种显式确认非常有价值。

### beta 与 release 的区别

| 选择 | 最终动作 | 使用时机 |
|---|---|---|
| `beta` | Archive、验证、上传到 TestFlight；默认不分发外部测试 | 给内部人员或 TestFlight 测试 |
| `release` + `confirm_release=true` | 校验 tag、测试、Archive、precheck、上传二进制和元数据、提交审核 | TestFlight 验证完成后 |

HabitPulse 的正式 release 还固定了 `automatic_release: false`。这意味着审核通过也不会自动公开到 App Store，仍由开发者在 App Store Connect 中手动发布。这是我很推荐的默认值：少一次意外，就少一次深夜救火。

### 把 App Store Connect 的 `.p8` 写到临时目录

上传 TestFlight 与提交审核需要 App Store Connect API Key。`.p8` 是私钥，不能提交到仓库，也不应该放在固定工作路径。正确方式是从 GitHub Secret 恢复到 `$RUNNER_TEMP`：

```yaml
- name: Configure App Store Connect credentials
  env:
    ASC_KEY: ${{ secrets.ASC_KEY_P8 }}
    ASC_KEY_ID: ${{ secrets.ASC_KEY_ID }}
    ASC_ISSUER_ID: ${{ secrets.ASC_ISSUER_ID }}
  run: |
    install -d -m 700 "$RUNNER_TEMP/appstore"
    printf '%s\n' "$ASC_KEY" > "$RUNNER_TEMP/appstore/AuthKey_${ASC_KEY_ID}.p8"
    chmod 600 "$RUNNER_TEMP/appstore/AuthKey_${ASC_KEY_ID}.p8"
    echo "ASC_KEY_FILEPATH=$RUNNER_TEMP/appstore/AuthKey_${ASC_KEY_ID}.p8" >> "$GITHUB_ENV"
    echo "ASC_KEY_ID=$ASC_KEY_ID" >> "$GITHUB_ENV"
    echo "ASC_ISSUER_ID=$ASC_ISSUER_ID" >> "$GITHUB_ENV"
```

Fastlane 则只从环境变量读取这三个值，并检查 Key ID 格式、Issuer ID 格式、文件是否存在以及文件权限是否过宽。换句话说：YAML 负责“把 Secret 安全地交给当前 Job”，Fastlane 负责“确认这些凭据看起来可用”。

真实项目里临时目录叫 `dayvu` 还是 `appstore` 并不重要；重要的是它在 `$RUNNER_TEMP` 下，不被提交、上传或复用。

## 七、五个 Secret 到底从哪里来

HabitPulse 的 `appstore` Environment 使用下面五个 Secret：

| Secret | 内容 | 获取方式 |
|---|---|---|
| `MATCH_DEPLOY_KEY` | 读取 match 证书仓库的 SSH 私钥全文 | 为 CI 单独生成 SSH key；把公钥作为证书仓库的 Deploy key（只读）或机器账户的 SSH key |
| `MATCH_PASSWORD` | match 加密密码 | 初始化 match 时设定的密码；不是 Apple ID 密码 |
| `ASC_KEY_ID` | App Store Connect API Key 的 Key ID | App Store Connect → Users and Access → Integrations → Keys |
| `ASC_ISSUER_ID` | App Store Connect API 的 Issuer ID | 同一 Keys 页面顶部 |
| `ASC_KEY_P8` | `.p8` 私钥完整文本 | 创建 API Key 时下载；通常只能下载一次，务必安全保存 |

### `MATCH_DEPLOY_KEY` 的推荐创建方式

不要把自己的个人 SSH 私钥直接塞进 CI。单独生成一把只给 CI 使用的 key：

```sh
ssh-keygen -t ed25519 -C "github-actions-match" -f ./github-actions-match
```

随后：

1. 将 `github-actions-match.pub` 添加到**证书仓库**的 Deploy keys；勾选只读即可；
2. 将没有 `.pub` 后缀的私钥文件完整内容添加为 `MATCH_DEPLOY_KEY`；
3. 不要把这两个文件提交到任何仓库；用完从当前目录删除或移到密码管理器。

私钥内容应包含开头和结尾，例如：

```text
-----BEGIN OPENSSH PRIVATE KEY-----
...多行密钥内容...
-----END OPENSSH PRIVATE KEY-----
```

### App Store Connect API Key 的权限

在 App Store Connect 创建 API Key 时，按实际操作授予最小权限。若 CI 需要上传 TestFlight 和提交审核，通常需要 App Manager 或能覆盖发布流程的适当角色；具体权限名称会随 Apple 后台演进。创建后立刻记录 Key ID、Issuer ID，并把 `.p8` 放进密码管理器。

有两个坑特别常见：

- `.p8` 一般只能下载一次。丢了不能重新下载同一把，需要撤销并新建 Key；
- GitHub Secret 必须保存 `.p8` 的**完整多行内容**，不是文件路径，也不是 Base64（除非你同时修改 workflow 进行解码）。

### 不要把 Secret 放错位置

如果 workflow 写的是：

```yaml
environment: appstore
```

那么最直观、也最符合本项目权限模型的做法，是把 Secret 放在 `Settings → Environments → appstore → Environment secrets`。如果你只放在仓库级 Secrets，可能也能通过 `secrets.xxx` 读取，但会失去 Environment 审批和隔离的意义。

## 八、Artifact 是 CI 的“事故现场”

CI 构建不应该只给你一个绿色或红色的小图标。发生失败时，最重要的是能拿到足够材料定位问题。

HabitPulse 会上传：

```yaml
path: |
  fastlane/build_output/ios/habitpulse/appstore/
  fastlane/test_output/
  build/*.log
```

其中通常包含：

- `.ipa`：可安装或提交的打包产物；
- `.xcarchive`：用于检查签名、dSYM 和归档结构；
- `.dSYM`：将来符号化崩溃日志需要；
- `.xcresult`：Xcode 测试报告，可在 Xcode 中打开；
- JUnit 与 xcodebuild 日志：适合快速查 CI 报错。

注意，Artifact 不等于长期制品仓库。它适合排错和短期下载；如果你有保留周期、访问控制或合规要求，应在仓库设置中明确配置 retention，或使用专门的制品存储方案。

## 九、从零配置到第一次成功的建议顺序

下面是一个比较稳的实施顺序。每一步只增加一点复杂度，出问题时也更容易定位。

1. 本机执行 `bundle exec fastlane ios test`，直到稳定通过；
2. 本机执行 `bundle exec fastlane ios habitpulse_archive_appstore`，确认主 App 与所有 Extension 都能签名；
3. 提交 `pr.yml`，创建一个指向 `main` 的 PR，确认无 Secret 的测试工作流通过；
4. 创建 GitHub `appstore` Environment，添加 `MATCH_DEPLOY_KEY` 和 `MATCH_PASSWORD`；
5. 合并一个小改动到 `main`，确认 `main-archive.yml` 能完成 archive，并能下载 Artifact；
6. 再添加 `ASC_KEY_ID`、`ASC_ISSUER_ID`、`ASC_KEY_P8`；
7. 在 Actions 中手动运行 `iOS Manual Release`，选择 `beta`，确认 TestFlight 出现新构建；
8. TestFlight 验证完成后，准备正确的版本 tag，再人工运行 `release` 并勾选确认；
9. 到 App Store Connect 检查构建、元数据、审核信息，审核通过后按自己的发布节奏手动公开。

如果你刚开始搭建，强烈建议先跑 `beta`。不要把“第一次验证 API Key、match 和 CI 环境”的风险，和“提交 App Review”绑在同一次执行中。

## 十、几个真实踩过的坑

### 1. 以为 push 到 main 会跑 PR 流程

不会。`pull_request` 与 `push` 是不同事件。直接命令行合并并推送到 `main`，只会触发监听 `push` 的主干归档 workflow。要保证 PR 检查不被绕过，使用 GitHub 分支保护规则。

### 2. 把所有动作塞进 main 的自动工作流

这是最危险的设计。主干自动归档很合理，自动上传 TestFlight 已经需要更谨慎，自动提交审核通常不值得。把发布留给 `workflow_dispatch`，并额外要求 `confirm_release`，能避免许多可预防的问题。

### 3. CI 里运行了 `match`，却没有临时 Keychain

这样可能下载证书成功，但 `codesign` 找不到私钥或无法访问证书。match、Keychain 和 Profile 是一条链，缺一个都可能在 Archive 阶段失败。

### 4. 忘记 Extension 的 Bundle ID

iOS App 只要有 Widget、Share Extension、Notification Service Extension 等目标，就不能只配置主 App 的 Bundle ID。match、Apple Developer 后台的 App ID 和 Profile 都要覆盖所有需要签名的 Target。

### 5. 让 CI 以写模式运行 match

CI 不该承担创建证书或 Profile 的责任。应使用 `readonly: true`。如果签名材料过期或缺失，让构建明确失败，然后由有权限的开发者在受控的本机环境处理。

### 6. 测试失败后没有报告

没有 `if: always()` 时，失败步骤会跳过 Artifact 上传；最终你只能看到一句“Process completed with exit code 65”。保留 xcresult 和完整日志，会让排查效率有非常明显的差别。

### 7. 把 `.p8`、`.p12` 或 SSH 私钥提交进仓库

这不是“之后再删掉就好”的问题。Git 历史里出现过的私钥应视为泄露，即使后来删除也应立即撤销并换新。正确位置是 GitHub Secret、密码管理器或专门的密钥管理系统。

### 8. 没有检查 Runner 的 Xcode 与模拟器 Runtime

`macos-15` 不代表一定存在你随手写下的每个 Xcode 或 iOS Runtime。把 Xcode 版本、测试设备和 Runtime 当作一个整体维护；如果升级 Xcode，先在 PR workflow 验证。

## 十一、可以直接复用的最小工作流组合

如果你暂时不做 App Store 发布，可以只从 PR workflow 开始；如果需要正式签名归档和 TestFlight，则按本文的三段式结构扩展。

最小但完整的职责图是：

```text
Fastlane
├── test                       ← 模拟器测试、报告
├── archive_appstore           ← match readonly、Archive、导出 IPA
├── beta                       ← 调用 archive 后上传 TestFlight
└── release                    ← 额外检查与人工确认后提交审核

GitHub Actions
├── pr.yml                     ← 只调用 test
├── main-archive.yml           ← 只调用 archive_appstore
└── manual-release.yml         ← 只允许人工调用 beta / release
```

它的核心不是“自动化得越多越好”，而是把自动化放到合适的位置：高频、低风险的检查自动做；低频、会对外产生影响的发布动作由人明确确认。

## 结语

给 iOS 项目接 GitHub Actions，真正的完成标准不是“看到一次绿色勾”，而是下面几件事都成立：

- PR 能在不接触发布凭据的前提下稳定验证代码；
- `main` 能在干净环境中稳定签名、归档并留下可下载的证据；
- 发布凭据被隔离在 Environment Secret 中，且只在需要它们的 Job 出现；
- TestFlight 和 App Store 审核不会因为一次普通 push 被误触发；
- 开发者能在本地用同一条 Fastlane 命令复现 CI 的关键动作。

做到这些之后，GitHub Actions 才不只是“替你点了构建按钮”，而是成为一条可靠、可审计、能让团队放心交付的 iOS 工程流水线。
