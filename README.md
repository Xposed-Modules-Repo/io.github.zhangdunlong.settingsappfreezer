# 设置内冻结 · Settings App Freezer

为系统「设置 → 应用信息」页注入 **冻结 / 解冻** 按钮，一键停用任意用户应用。
为 **LSPosed + ColorOS / RealmeUI / 现代 Android** 重新实现。

> 本仓库是 Xposed-Modules-Repo 的**模块分发仓库**，只存放元数据与发布产物。
> **源码**请见 <https://github.com/zhangdunlong/settings-app-freezer>

---

## 这是什么

Android 8 之后官方设置里的「停用」入口被移除，冻结应用通常得靠 `pm disable-user`
或 Titanium Backup 这类外部工具。本模块把冻结能力直接放回你每天都会经过的
「应用信息」页面 —— 打开页面，点一下，应用即冻结；从桌面消失，数据保留。

## 核心功能

- **一键冻结 / 解冻** —— 应用信息页浮动按钮，冻结后应用从桌面消失，数据完整保留
- **状态一目了然** —— 未冻结青色，已冻结薄荷绿；颜色 + 文案双通道表达，不单靠颜色编码
- **已冻结应用列表** —— 底部「❄ 已冻结 N」入口带实时计数角标，弹窗列出全部已冻结应用
  （图标 + 应用名 + 包名），点击任意行即解冻
- **仅统计用户主动冻结** —— 只计 `DISABLED_USER`，排除系统出厂禁用项
- **长按诊断** —— 显示目标包名、启用状态、是否系统应用、是否被保护

## 深层安全保护

误冻系统组件会导致设备无法使用。本模块**动态识别**（非硬编码）并拒绝冻结：

- 系统关键包 —— 设置、SystemUI、权限控制器、应用商店、GMS 等
- 当前桌面启动器 —— 冻结它会让你没有桌面可用
- 当前启用的输入法 —— 冻结它会让全局无法输入
- 正在前台运行的应用

被拦截时会明确告知原因，不会静默失败。

## 分身（App Twin / MultiApp）支持

- 冻结 / 解冻**精确到 Android user**，可在分身内单独冻结而不影响本机原版
- 分身环境下按钮显示 `[分身]` 前缀，读屏播报「分身应用」

## 无障碍

- 全量 `contentDescription`（含状态与拦截原因），读屏朗读整行而非割裂碎片
- 按钮具备 `focused` 态，键盘 / 导航设备可达
- 列表行触控目标 ≥56dp，文字对比度全部达 WCAG AA（≥4.5:1）

## 技术要点

| | |
|---|---|
| **ROM 无关** | 不 hook 任何厂商内部类名，改用 Intent 判定 + 视图内容特征识别 |
| **多层兜底识别** | 多级包名解析路径 + 归一化（兼容 `//com.example.app` 这类带斜杠的值），最后用 `getApplicationInfo()` 硬验证 |
| **六层生命周期 hook** | `onCreate` / `onResume` / `onPostResume` / `onNewIntent` / `onWindowFocusChanged` / `ViewRootImpl.performTraversals` |
| **跨 Activity 交接** | 宿主销毁时把注入转交给持焦点窗口，独立 12 秒重试预算 |
| **零 Gradle 构建** | 纯命令行链路，无需 Android Studio |

### 关于 ColorOS 的两段式页面

旧版冻结类模块在现代 ROM 上失效，主因是 ColorOS 应用信息页的真实结构：

```
com.android.settings.applications.InstalledAppDetailsTop        ← 瞬时 Activity，启动即销毁
                    ↓ 交接
com.android.settings.Settings / SubSettings                     ← 真正渲染，单 Activity 内换页
```

在 `onCreate` 注入时布局尚未测量（`w=0 h=0`），而百余毫秒后的首次重试时宿主已经销毁 ——
**重试窗口与页面存活窗口完全错开**，按钮因此永不出现。

本模块通过跨 Activity 交接 + 可重试的内容特征守卫解决该问题。

## 运行要求

- Android 8.0 (API 26) 及以上
- **LSPosed** 框架（Zygisk / NeoZygisk 环境）
- Root 权限
- 已验证：OnePlus PJZ110 / Android 16 (API 36) / ColorOS

## 安装

1. 下载 Releases 中的 `freezer-module.apk` 安装到手机
2. LSPosed Manager → 模块 → 勾选 **settings-app-freezer**
3. 进入模块 → 作用域 → **同时勾选**以下两项：

   | 作用域 | 缺失后果 |
   |---|---|
   | **系统框架**（`system`） | 模块开机时不注入，任何页面都不会有按钮 |
   | **设置**（`com.android.settings`） | 模块能加载，但不会 hook 任何设置进程 |

   > ⚠️ **两项都必须勾**。缺任意一项都不会生效，且**没有任何界面提示**。

4. **重启手机**（LSPosed 在开机时读入作用域，仅强停设置进程不够）
5. 打开任意应用信息页，即出现冻结按钮

## 排查：装好了但没反应

```bash
adb shell su -c 'grep Freezer /data/adb/lspd/log/modules_*.log | tail -50'
```

| 日志现象 | 含义 | 处理 |
|---|---|---|
| 完全没有 `FreezerModule` 日志 | 作用域未生效 | 检查 `system` 与 `com.android.settings` 两项是否都勾 |
| 有 hook 日志，无 `button bar injected` | 包名未解析到 | 该机型页面结构未适配，请附日志提 Issue |

## 已知限制

- **需要 Root + LSPosed**，作用域改动后必须重启手机
- **系统应用通常无法冻结** —— `DISABLED_USER` 对系统应用需更高权限，会被拒绝
- **加固应用可能自我拉活** —— 部分应用带自保护机制
- **冻结不等于删除** —— 采用框架级「停用」语义，数据完整保留
- **ColorOS 分身** —— 分身详情页由主用户 settings 进程渲染，若页面 Intent 未携带 user 标识，
  分身作用域可能需手动指定
- **其他 ROM 首次适配** —— 需提供 LSPosed 日志协助排查

## 反馈

遇到问题请到源码仓库提交 Issue，并附上：

```bash
adb shell su -c 'grep Freezer /data/adb/lspd/log/modules_*.log | tail -50'
```

同时请说明设备型号、Android 版本、ROM 与 LSPosed 版本。

---

**许可**：[GPL-3.0](https://github.com/zhangdunlong/settings-app-freezer/blob/main/LICENSE)
· **源码**：<https://github.com/zhangdunlong/settings-app-freezer>
