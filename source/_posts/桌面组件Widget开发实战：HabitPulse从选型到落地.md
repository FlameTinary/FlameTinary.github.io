---
title: 桌面组件 Widget 开发实战：HabitPulse 从选型到落地
tags: [iOS, WidgetKit, SwiftData, CloudKit, App Group, fastlane, match, 踩坑]
date: 2026-09-01
---

# 桌面组件 Widget 开发实战：HabitPulse 从选型到落地

> 以习惯打卡 App **HabitPulse**（Swift 6 + SwiftUI + SwiftData + CloudKit，手动 `.xcodeproj`）为例，系统复盘「今日打卡进度」桌面组件从**方案选型 → 工程接入 → 开发者中心配置 → 真机踩坑**的完整过程。
>
> 如果你只想看结论：本项目最终采用 **「App Group + JSON 快照 + 深链」** 方案，主 app 写文件、Widget 读文件、点击 Widget 深链回主 app，全程不碰 Widget 内的写入与 SwiftData 直连。

---

## 0. 背景与目标

HabitPulse 是一个习惯打卡应用，核心数据模型：

- `Habit`：习惯（名称、图标、颜色、频率规则、是否归档）
- `HabitCompletion`：打卡记录（日期、状态 done/skip、关联 habitID）

我们想做一个**桌面组件（Widget）**，在手机主屏直接展示「今日待打卡习惯 + 完成进度」，并且点一下能跳回 App 的「今日」页面去打卡。

技术栈有一个关键约束：**数据层是 SwiftData + CloudKit 同步**。

- SwiftData 底层是 SQLite；开启 CloudKit 后，本地还有一个 **CloudKit 镜像库（`.sqlite` + `.sqlite-wal`）**。
- Widget 是**独立进程（App Extension）**，与主 app 不在同一个进程空间。

这个约束直接决定了 Widget 能不能「直连数据库」——下面第一节展开。

---

## 一、Widget 的几种接入方式对比

WidgetKit 扩展要拿到主 app 的数据，业界主流有三条路：

### 方案 A：App Group + 共享 SwiftData 直连

让 Widget Extension 直接 `ModelContainer(for: schema, ...)` 打开**主 app 同一个 SwiftData store**（store 放到 App Group 容器里，两个进程共享同一份 `.sqlite`）。

```swift
// Widget 侧（设想）
let config = ModelConfiguration(
    schema: schema,
    url: AppGroupContainer.appGroupSQLITEURL   // 与主 app 同一路径
)
let container = try ModelContainer(for: schema, configurations: [config])
```

### 方案 B：App Group + JSON 快照 + 深链（本项目采用）

主 app 是唯一写入者：在合适的时机把「今日进度」序列化成 `widget_snapshot.json` 写到 App Group 容器。Widget 只**读**这个 JSON 渲染，并通过 `widgetURL` 深链把点击转给主 app。

```
主 app (进程1)  --写-->  App Group/widget_snapshot.json  <--读--  Widget (进程2)
                                  │
                          WidgetCenter.reloadAllTimelines()
                                  │
                    widgetURL(habitpulse://today) 点击 ──> 主 app 今日 Tab
```

### 方案 C：App Intents + 可交互 Widget（Widget 内直接打卡）

Widget 上放一个按钮，用户点了直接在 Widget 进程里完成打卡（通过 `AppIntent`/`TimelineProvider` 的 `intent` 配置）。写入仍需落到共享存储（方案 A 或 B 的存储之上再叠一套写回机制）。

### 三者对比

| 维度 | A. 共享 SwiftData 直连 | B. JSON 快照 + 深链（采用） | C. Widget 内可交互打卡 |
|---|---|---|---|
| **多进程并发安全** | ⚠️ 差：两进程同时开同一 SQLite，CloudKit 镜像库易锁/崩溃 | ✅ 好：单一写入者，Widget 只读文件 | ⚠️ 中：Widget 也变写入者，需额外同步 |
| **是否需要迁移 store** | ⚠️ 需把默认 store 迁到 App Group 容器，可能清空/复杂 | ✅ 不需动现有 SwiftData | ⚠️ 同 A 的存储问题 |
| **数据实时性** | ✅ 最高（直接读库） | ✅ 高（写入即 `reloadAllTimelines`） | ✅ 高 |
| **CloudKit 冲突风险** | ❌ 高：扩展误改镜像库会污染同步 | ✅ 无：扩展从不碰 SwiftData | ❌ 高 |
| **实现复杂度** | 中（但后期坑多） | 中（契约要双方对齐） | 高（Intent + 写回 + 冲突） |
| **打卡交互位置** | 取决于是否在 Widget 内加按钮 | 点击深链回主 app 打卡 | Widget 内直接打卡 |
| **适合本项目的程度** | 不适合（CloudKit 多进程锁） | ✅ 最适合 | 过度设计（v1 不需要 Widget 内打卡） |

---

## 二、为什么选方案 B（JSON 快照 + 深链）

一句话：**在「SwiftData + CloudKit」项目里，让 Widget 直连数据库是高风险动作；让主 app 独占写入、Widget 只读快照，把并发冲突面降到最低。**

具体原因：

### 1. 规避多进程 SQLite 锁（最核心）

CloudKit + SwiftData 的本地镜像库，本质是多个 SQLite 文件。主 app 运行时会持续持有并写入它（本地改动 → 触发 CloudKit 上传）。如果 Widget 进程也用同一份 store 打开：

- 两个进程各自持锁，轻则 `database is locked`，重则镜像库损坏、CloudKit 同步异常。
- Widget 进程生命周期由系统管理（主屏 daemon 拉起/挂起），无法与主 app 协调事务。

> 业界竞品（如 Kado 等习惯类 App）的成熟做法也是：**扩展不直开 SwiftData，改走 JSON 快照**。我们用同样的思路。

### 2. 不迁移 store，避免清空用户数据

方案 A 要把 SwiftData 默认 store 迁到 App Group 容器才能共享。迁移过程一旦出错（路径、版本、CloudKit 重连），**可能清空本地数据**。对已经上线、有真实用户数据的 App 这是不可接受的。方案 B 完全不碰 store。

### 3. 单一写入者原则

只有主 app 持有 `ModelContext` 并负责写入。Widget 是纯消费者，读一份 JSON 即可。并发模型从「双写者竞争」降级为「单写者 + 多读者」，简单且可靠。

### 4. 打卡放在主 app，而非 Widget 内

打卡是**写操作**。如果放 Widget 里（方案 C），Widget 既要读又要写，又回到多进程写入的坑；而且「今日进度」这种聚合数据在 Widget 里重算也很重。本项目 v1 的策略是：**Widget 只展示 + 引导**，点击 → 深链回主 app 今日 Tab 完成打卡。职责清晰。

### 5. JSON 契约解耦

主 app 内部模型随便演进，只要持续输出一份稳定结构的 `widget_snapshot.json`，Widget 就不需要跟着改。两侧通过 JSON 字段契约解耦。

---

## 三、方案 B 的具体接入步骤

### 3.1 主 app 侧：快照写入服务 `WidgetSnapshotService`

核心职责：取今日数据 → 算进度 → 写 App Group JSON → **通知 WidgetKit 刷新**。

```swift
import Foundation
import SwiftData
import os
import WidgetKit

enum WidgetSnapshotService {
    static let appGroupID = "group.com.sheldon.HabitPulse"
    static let fileName = "widget_snapshot.json"

    private static let logger = Logger(
        subsystem: "com.sheldon.HabitPulse",
        category: "widget-snapshot"
    )

    /// 刷新并写入今日快照。须在持有主 app ModelContext 的主线程调用。
    @MainActor
    static func refresh(using context: ModelContext) {
        let snapshot = buildSnapshot(using: context)
        write(snapshot)
        // ⚠️ 关键：跨进程同步完成后必须显式 reload，
        // 否则 WidgetKit 不会主动重读 App Group 文件。
        WidgetCenter.shared.reloadAllTimelines()
    }

    private static func buildSnapshot(using context: ModelContext) -> WidgetSnapshot {
        let habits = (try? context.fetch(FetchDescriptor<Habit>(
            predicate: #Predicate { !$0.isArchived }
        ))) ?? []
        let completions = (try? context.fetch(FetchDescriptor<HabitCompletion>())) ?? []
        let today = Calendar.current.startOfDay(for: Date())
        let todayDone = completions.filter {
            $0.status == .done && Calendar.current.isDate($0.date, inSameDayAs: today)
        }

        var overallDone = 0, overallTarget = 0
        var items: [WidgetHabitItem] = []

        for habit in habits {
            guard FrequencyEvaluator.isDue(on: Date(), habit: habit) else { continue }
            let target = FrequencyEvaluator.targetPerDay(habit: habit)
            let done = todayDone.filter { $0.habitID == habit.id }.count
            let fully = done >= target
            if fully { overallDone += 1 }
            overallTarget += 1
            items.append(WidgetHabitItem(
                id: habit.id.uuidString, name: habit.name, icon: habit.icon,
                colorHex: habit.colorHex, targetPerDay: target,
                doneCount: done, isFullyDone: fully
            ))
        }
        // 未完成排前、已完成排后，名称字典序稳定排序。
        items.sort {
            if $0.isFullyDone == $1.isFullyDone { return $0.name < $1.name }
            return !$0.isFullyDone
        }
        return WidgetSnapshot(
            date: ISO8601DateFormatter().string(from: Date()),
            overallDone: overallDone, overallTarget: overallTarget, habits: items
        )
    }

    private static func write(_ snapshot: WidgetSnapshot) {
        guard let container = FileManager.default
            .containerURL(forSecurityApplicationGroupIdentifier: appGroupID) else {
            // 真机 containerURL 为 nil = entitlements 声明了 App Group
            // 但描述文件(profile)没该能力——模拟器不校验所以测不出。
            logger.error("App Group 容器不可用：profile 缺能力或 ID 不一致")
            return
        }
        let fileURL = container.appendingPathComponent(fileName)
        do {
            let data = try JSONEncoder().encode(snapshot)
            try data.write(to: fileURL, options: [.atomic])
            logger.debug("快照写入成功：habits=\(snapshot.habits.count)")
        } catch {
            logger.error("快照写入失败：\(String(describing: error))")
        }
    }

    /// 供 Widget 复用的 JSON 契约（主 app 编码、Widget 解码，字段须完全一致）。
    struct WidgetSnapshot: Codable {
        let date: String
        let overallDone: Int
        let overallTarget: Int
        let habits: [WidgetHabitItem]
    }
    struct WidgetHabitItem: Codable {
        let id: String
        let name: String
        let icon: String
        let colorHex: String
        let targetPerDay: Int
        let doneCount: Int
        let isFullyDone: Bool
    }
}
```

> 注意：`WidgetSnapshot` / `WidgetHabitItem` 在**主 app 和 Widget 两侧各定义一份相同字段的 `Codable`**（Widget 侧文件 `WidgetSnapshot.swift` 里也有一份）。这是 JSON 契约的两端，改动任一侧都要同步另一侧。

### 3.2 主 app 触发刷新的时机

只在「写完文件」还不够，必须在用户会看到变化的地方触发 `refresh`：

- **App 启动时**写首屏数据（否则首次添加 Widget 永远空）：

```swift
// HabitPulseApp.swift
.task {
    WidgetSnapshotService.refresh(using: modelContainer.mainContext)
}
.onChange(of: scenePhase) { _ in
    // 回到前台：跨天 / 后台数据变更后，Widget 重新可见即更新。
    WidgetSnapshotService.refresh(using: modelContainer.mainContext)
}
```

- **打卡 / 跳过 / 新建习惯后**立即刷新（保证「动一下就同步」）：

```swift
// TodayViewModel.swift 的 persist() 末尾
WidgetSnapshotService.refresh(using: modelContext)
```

### 3.3 深链：点击 Widget 回「今日」Tab

Widget 本身不打卡，点击 → `habitpulse://today` → 主 app 切到今日 Tab。

**(1) 主 app `Info.plist` 注册 URL Scheme：**

```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>habitpulse</string>
    </array>
    <key>CFBundleURLName</key>
    <string>com.sheldon.HabitPulse</string>
  </dict>
</array>
```

**(2) `HabitPulseApp` 处理 URL：**

```swift
.onOpenURL { handleDeepLink($0) }

private func handleDeepLink(_ url: URL) {
    guard url.scheme == "habitpulse" else { return }
    NotificationCenter.default.post(name: .widgetDeepLinkToday, object: nil)
}

// 通知名
extension Notification.Name {
    static let widgetDeepLinkToday =
        Notification.Name("com.sheldon.HabitPulse.deepLink.today")
}
```

**(3) `RootTabView` 用 `selection` 接收并切 Tab：**

```swift
struct RootTabView: View {
    @State private var selectedTab: MainTab = .today
    var body: some View {
        TabView(selection: $selectedTab) {
            TodayView().tag(MainTab.today)
                .tabItem { Label { LText("tab.today") } icon: { Image(systemName: "sun.max.fill") } }
            // ... 其余 3 个 Tab
        }
        .onReceive(NotificationCenter.default.publisher(for: .widgetDeepLinkToday)) { _ in
            selectedTab = .today
        }
    }
}

enum MainTab: Hashable { case today, calendar, stats, me }
```

### 3.4 Widget Extension Target 的工程接入

本项目是**手动维护的 `.xcodeproj`**，没有用 Xcode 的「Editor → Add Target → Widget Extension」向导，所以 target 是**手写进 `project.pbxproj`** 的（UUID 前缀统一用 `2B` 区隔主 app 的 `1A`）。需要补齐以下段落：

- `PBXBuildFile`：每个源文件一条 `in Sources`，外加一条 `Embed App Extensions` 的 copy
- `PBXFileReference`：5 个 swift 源 + `Info.plist` + `.entitlements` + 最终的 `.appex`
- `PBXGroup`：新建 `HabitPulseWidget` 分组，挂到主 `mainGroup`
- `PBXNativeTarget`：`productType = com.apple.product-type.app-extension`
- `PBXSourcesBuildPhase` / `PBXFrameworksBuildPhase` / `PBXResourcesBuildPhase`
- `PBXCopyFilesBuildPhase`：Embed App Extensions（`dstSubfolderSpec = 13`），把 `.appex` 拷进主 app 的 `PlugIns/`
- `PBXTargetDependency` + `PBXContainerItemProxy`：主 app 依赖 Widget
- `XCConfigurationList` + 两个 `XCBuildConfiguration`（Debug/Release）

关键构建设置（`Debug` 为例）：

```ruby
CODE_SIGN_STYLE = Manual
CODE_SIGN_IDENTITY = Apple Development
DEVELOPMENT_TEAM = CJ5L2XW65B
PRODUCT_BUNDLE_IDENTIFIER = com.sheldon.HabitPulse.Widget
INFOPLIST_FILE = HabitPulseWidget/Info.plist
CODE_SIGN_ENTITLEMENTS = HabitPulseWidget/HabitPulseWidget.entitlements
SWIFT_VERSION = 6.0
SWIFT_STRICT_CONCURRENCY = complete
IPHONEOS_DEPLOYMENT_TARGET = 17.6
GENERATE_INFOPLIST_FILE = NO
```

主 app 的 `PBXCopyFilesBuildPhase` 里加入 `Embed App Extensions`，`dependencies` 加入对 Widget 的 `PBXTargetDependency`——这样 `.appex` 才会被打进主 app 包。

> ⚠️ 本项目用 Swift 6 严格并发（`SWIFT_STRICT_CONCURRENCY = complete`），Widget 源码也要继承同样的并发约束，否则主 app 能过、Widget target 单独编不过。

### 3.5 Widget 侧代码（5 个文件 + Bundle）

- **`WidgetSnapshot.swift`**：与主 app 同字段的 `WidgetSnapshot`/`WidgetHabitItem` `Codable` + `WidgetSnapshotStore`（读 App Group JSON 解码，失败回退 `.placeholder`）。
- **`HabitWidgetProvider.swift`**：`TimelineProvider`
  - `placeholder` / `getSnapshot`：直接读 JSON（预览/首屏）
  - `getTimeline`：每小时生成一个刷新点作为**兜底**（即使主 app 没主动 reload，最多 1 小时也会自己刷新一次）
- **`WidgetViews.swift`**：独立 `Color(widgetHex:)` + 14 色调色板（复制自主 app）、`WidgetProgressRing` 进度环、`HabitWidgetEntryView`（small/medium/large 三尺寸；medium 显示前 3 条、large 全量）
- **`HabitPulseWidget.swift`**：

```swift
@main   // 注意：Widget 的 @main 在 Bundle 文件，这里不要重复
struct HabitPulseWidget: Widget {
    let kind = "HabitPulseWidget"
    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: HabitWidgetProvider()) { entry in
            HabitWidgetEntryView(entry: entry)
        }
        .configurationDisplayName("今日打卡")
        .description("查看今日习惯完成进度")
        .widgetURL(URL(string: "habitpulse://today"))   // 点击深链
        .supportedFamilies([.systemSmall, .systemMedium, .systemLarge])
        .containerBackground(Color(.systemBackground), for: .widget)
    }
}
```

- **`HabitPulseWidgetBundle.swift`**：

```swift
@main
struct HabitPulseWidgetBundle: WidgetBundle {
    var body: some Widget { HabitPulseWidget() }
}
```

- **`Info.plist`**：必须含 `NSExtensionPointIdentifier = com.apple.widgetkit-extension`
- **`HabitPulseWidget.entitlements`**：声明 `com.apple.security.application-groups` 含 `group.com.sheldon.HabitPulse`

### 3.6 App Group 容器共享

主 app 与 Widget 的 entitlements 都声明同一个 App Group：

```
group.com.sheldon.HabitPulse
```

- 主 app 写：`FileManager.default.containerURL(forSecurityApplicationGroupIdentifier: "group.com.sheldon.HabitPulse")`
- Widget 读：同一句，拼 `widget_snapshot.json`

---

## 四、开发者中心（Apple Developer Portal）配置要点

这一块是真机才能暴露的坑——**模拟器不强制校验描述文件 / entitlements**，所以本地编译全过、模拟器能跑，但真机装完「找不到 Widget」十有八九是这里没配。

### 4.1 注册 Widget 的 App ID

1. 开发者中心 → Identifiers → **新建 App ID**
2. **类型选「App」（不是 App Clip）**：Widget Extension 复用宿主 App 的签名证书与 App Group，本质是普通 App ID。
3. **Bundle ID 类型选「Explicit」（显式）**，填 `com.sheldon.HabitPulse.Widget`。**不要用 Wildcard（`*`）**——通配 App ID 不能启用 App Groups，而 Widget 强依赖它。
4. **Capabilities 勾选 App Groups**，并在其配置里关联 `group.com.sheldon.HabitPulse`（主 app 已注册过的那个 group）。

### 4.2 App Groups 注册与关联（若未注册）

左侧 **App Groups** 里确认 `group.com.sheldon.HabitPulse` 存在，并把它关联到**主 app 和 Widget 两个 App ID**。

### 4.3 Fastfile 改造（纳入 Widget identifier）

本项目的签名走 `fastlane match`。原 lane 的 `app_identifier` 只写了主 app，必须把 Widget 一起纳进来，否则 `match` 不会生成 Widget 的 profile：

```ruby
desc "habitpulse 开发证书（含 Widget 扩展）"
lane :habitpulse_dev do
  match(
    type: "development",
    app_identifier: ["com.sheldon.HabitPulse", "com.sheldon.HabitPulse.Widget"],
    git_url: MATCH_GIT_URL,
    git_branch: MATCH_GIT_BRANCH,
  )
end

desc "habitpulse AppStore证书（含 Widget 扩展）"
lane :habitpulse_appstore do
  match(
    type: "appstore",
    app_identifier: ["com.sheldon.HabitPulse", "com.sheldon.HabitPulse.Widget"],
    git_url: MATCH_GIT_URL,
    git_branch: MATCH_GIT_BRANCH,
  )
end
```

### 4.4 match 重签（不需要 `--force`）

```bash
fastlane habitpulse_dev        # 生成 match Development com.sheldon.HabitPulse.Widget
fastlane habitpulse_appstore   # 仅打 Release/IPA 时需要
```

> **为什么不用 `--force`**：你这次是给**全新 App ID** 出 profile，第一次创建就是干净的，会直接把刚在门户启用的 App Groups 能力打进描述文件。只有「**已存在 App ID 后来才加能力**」时才需 `--force` 强制重签。先配能力再出 profile，省掉 `--force`。

### 4.5 Xcode 签名对齐

Widget target → Build Settings → Signing：

- `Code Signing Style` → **Manual**
- Debug → `Provisioning Profile` = `match Development com.sheldon.HabitPulse.Widget`
- Release → `Provisioning Profile` = `match AppStore com.sheldon.HabitPulse.Widget`

**关键**：主 app 与 Widget 的签名方式必须一致（本项目都用 `match` Manual）。如果 Widget 是 Automatic、主 app 是 Manual，真机上 Widget 的 App Group 能力可能没正确预置，扩展在启动访问 `containerURL` 时被系统拒绝，从而被**悄悄排除在组件库之外**（表现就是「app 能跑、Widget 找不到」）。

### 4.6 真机安装后

装完**重启一次手机**——Widget daemon 偶发不刷新，重启能强制重新索引组件。

---

## 五、开发过程中遇到的实际问题 & 解决方案

### 问题 1：`Cannot find 'WidgetSnapshotService' in scope`

**现象**：编译失败，报错找不到主 app 新建的快照服务。
**根因**：`WidgetSnapshotService.swift` 文件建了，却**漏加进主 app target**（没在 `project.pbxproj` 里登记 `PBXBuildFile`/`PBXFileReference`/`Sources build phase`）。
**解决**：补齐 4 处 pbxproj 引用（`PBXBuildFile` + `PBXFileReference` + `Core` group + 主 app `Sources` build phase），用新 UUID（`1A…00C2`/`00C3`）避免冲突。

### 问题 2：元组 `(Bool, String)` 不能用 `<` 比较

**现象**：原排序 `items.sort { (!$0.isFullyDone, $0.name) < (!$1.isFullyDone, $1.name) }` 编译报错。
**根因**：Swift 不允许对 `(Bool, String)` 这类异质元组用 `<` 直接比较。
**解决**：改成显式比较器：

```swift
items.sort {
    if $0.isFullyDone == $1.isFullyDone { return $0.name < $1.name }
    return !$0.isFullyDone   // 未完成排前
}
```

### 问题 3（最隐蔽）：Widget 真机显示「今日 0/0 暂无打卡计划」，但 app 里明明有数据

**现象**：主屏 Widget 一直空，app 内新建/打卡后 Widget 也不更新。
**根因**：主 app 把 JSON 写到 App Group 后，**没有调用 `WidgetCenter.shared.reloadAllTimelines()`**。WidgetKit 不会主动监测 App Group 文件变化，唯一的跨进程刷新手段就是 `reloadAllTimelines`。Widget 里的「每小时兜底刷新」对「刚加完 Widget」的用户完全无感。
**为什么模拟器没暴露**：模拟器测试时我直接去 App Group 容器里读 `widget_snapshot.json` 验证写入成功，根本没验证 Widget 进程实际有没有拿到——写入成功 ≠ Widget 收到了。
**解决**：
- `WidgetSnapshotService.refresh` 末尾加 `WidgetCenter.shared.reloadAllTimelines()`（`@MainActor`，写在已是 MainActor 的方法末尾无并发问题）。
- 顺手给 `write` 加 `os.Logger`（subsystem `com.sheldon.HabitPulse`，category `widget-snapshot`），把「containerURL 为 nil / 写入失败」从**静默吞掉**改成可观测——以后在 Mac 的 **Console.app** 过滤该 subsystem 就能定位。

### 问题 4：签名不一致（Widget Automatic vs 主 app match Manual）

**现象**：真机装了 app 却找不到 Widget。
**根因**：主 app 用 `match` 手动描述文件，Widget 一开始是 Automatic 签名，两者签名链不同；Widget 的 App Group 能力在真机可能没正确预置 → 扩展加载被拒 → 不进组件库。
**解决**：Widget target 切到 `match` Manual，Debug/Release 对齐主 app（见第四节 4.5）。本机确认两份 profile 已生成：
- `match Development com.sheldon.HabitPulse.Widget`
- `match AppStore com.sheldon.HabitPulse.Widget`

### 问题 5：模拟器「能编能嵌」≠ 真机「能显示」

**教训**：模拟器**不强制校验描述文件 / entitlements / App Group 能力**。任何签名或能力缺失在模拟器上都「看起来正常」，只有真机才暴露。所以 **Widget 类功能一定要上真机验证**，不能只信模拟器编译通过。

### 问题 6（附带）：`git push` 一直 `502 CONNECT tunnel failed`

**现象**：push 到 GitHub 反复失败，报代理 502；直连又 `Couldn't connect to github.com:443`。
**根因排查**：
- 用户网络**必须走代理**才能上 GitHub（直连 443 不通）。
- 但 shell 环境被注入了全局代理 `HTTP_PROXY/HTTPS_PROXY/http_proxy/https_proxy = http://127.0.0.1:52383`（WorkBuddy/CodeBuddy 的透明代理），它当前对 GitHub 上游返回 502。
- 第一次我显式用 `HTTP_PROXY=http://127.0.0.1:7897` 覆盖了**大写**变量，但**小写** `http_proxy/https_proxy` 仍指向 52383，git 实际走了那个坏的代理。
- 实测 `127.0.0.1:7897` 才是可用的本地代理（curl 经它访问 GitHub 返回 200）。
**解决**：把大小写四个变量**全部统一**指向 7897 再 push：

```bash
HTTP_PROXY=http://127.0.0.1:7897 HTTPS_PROXY=http://127.0.0.1:7897 \
http_proxy=http://127.0.0.1:7897 https_proxy=http://127.0.0.1:7897 \
git push origin main
```

> 经验：**覆盖代理环境变量时一定要大小写一起覆盖**，否则 git/curl 可能仍读到你没清掉的那个。

---

## 六、验证清单（交付前自检）

| 项 | 命令 / 操作 | 期望 |
|---|---|---|
| 双 target 编译 | `xcodebuild -scheme HabitPulse build`（加 `OTHER_SWIFT_FLAGS="-Xfrontend -disable-sandbox"` 绕过宏沙盒） | `BUILD SUCCEEDED` |
| 快照写入 | 模拟器 `simctl install/launch` 后查 App Group 容器 | `widget_snapshot.json` 存在且 `habits` 数组正确 |
| 深链 | `xcrun simctl openurl <UDID> "habitpulse://today"` | app 不崩、切到今日 Tab |
| 真机显示 | 主屏长按空白 → 「+」→ 搜 HabitPulse | Widget 出现在组件库 |
| 真机同步 | app 内打卡/新建 → 回主屏 | Widget 数据立即更新 |
| 真机点击 | 点 Widget | 深链回 app 今日 Tab |

---

## 七、总结 & 最佳实践

1. **CloudKit + Widget = 别直连数据库**。用「App Group JSON 快照 + 主 app 独占写入」把并发冲突面降到最低。
2. **写完文件 ≠ Widget 刷新**。必须 `WidgetCenter.shared.reloadAllTimelines()`，否则用户看到的是旧数据。
3. **Widget 内只做展示与引导**，写操作（打卡）交给主 app，用深链回跳。
4. **真机才是照妖镜**：签名、App Group 能力、描述文件这些问题模拟器全不校验，务必上真机。
5. **主 app 与 Extension 签名方式要一致**（本项目统一 `match` Manual），否则扩展会被悄悄排除。
6. **日志别静默吞**：`containerURL` 为 nil / 写入失败这类，用 `os.Logger` 留线索，真机排查全靠 Console.app。
7. **代理环境变量大小写一起管**：CI / 自动化里覆盖代理时，四个变量（HTTP/HTTPS × 大写/小写）一起设，否则容易踩到没清掉的坏代理。

---

*相关代码位置（HabitPulse 工程）：*
- 主 app：`HabitPulse/Core/WidgetSnapshotService.swift`、`HabitPulse/HabitPulseApp.swift`、`HabitPulse/Views/RootTabView.swift`、`HabitPulse/ViewModels/TodayViewModel.swift`、`HabitPulse/Info.plist`
- Widget：`HabitPulseWidget/` 整个目录（5 swift + Info.plist + entitlements）
- 工程配置：`HabitPulse.xcodeproj/project.pbxproj`（Widget target，UUID 前缀 `2B`）
- 签名：`fastlane/Fastfile`（`habitpulse_dev` / `habitpulse_appstore` 含 Widget identifier）
