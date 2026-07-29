# ZZ-ZD

*ZZ-ZD — 开源、本地、可审计的 ZD Ultimate Legend 手柄配置工具。*

![Windows 10/11](https://img.shields.io/badge/Windows-10%2F11-0078D6)
![License: MIT](https://img.shields.io/github/license/ZeLoveZzz/ZZ-ZD)
![Python 3.12+](https://img.shields.io/badge/Python-3.12%2B-blue)

ZZ-ZD 是一款免费、开源的 Windows 应用，用于读取和应用 ZD Ultimate Legend 手柄的配置。它**独立运行**——通过 USB-HID 直接与手柄通信，无需安装官方 ZD 应用。轻量、完全本地化、跨会话保留你的配置。

> **独立第三方项目，非 ZD Gaming 开发、关联或认可。** "ZD Gaming" 和 "ZD Ultimate Legend" 仅用于标识本工具兼容的手柄；所有商标归各自所有者所有。

---

## 核心特性

### 可验证的信任

- **开源、MIT 协议** — 每行代码都在 GitHub 上，可阅读、自行构建、fork
- **完全本地 — 无网络、无遥测** — 进程发起**零**网络调用，无遥测、无分析、无自动更新
- **无驱动、无虚拟设备、无后台服务** — 关闭应用后不留任何进程或服务
- **诚实的写入报告** — 每次 Apply 报告每个字段的真实写入结果，并从设备刷新屏幕状态；配置 Apply / Restore / Safe Import / 内联死区流程会读回并验证可读字段
- **无宏、无连发、无自动化** — 只配置手柄，绝不代玩（由测试强制约束）
- **默认安全网** — 涉及设备的变更会先捕获还原点，事件写入仅追加的本地账本

### 手柄配置（完整功能）

所有设置通过标准 HID feature report 写入：

- **USB 回报率** — 250–8000 Hz（8K 需固件 v1.18+）
- **16×16 按键绑定矩阵** — 含从手柄读取的当前绑定显示
- **摇杆** — 死区（4 区）、灵敏度曲线（3 锚点，固件 v1.24+ 支持 8 点曲线）、轴反转、摇杆步长
- **扳机** — 行程范围、模式、振动模式
- **灯光** — 分区控制（Home / 左 / 右）— 开关、模式、亮度、RGB
- **振动** — 独立马达 + 扳机振动模式
- **背键绑定** — 1 步控制器按键映射（8 个背键：M1–M4, LM, RM, LK, RK）
- **配置档案** — 保存 / 应用 / 删除完整手柄状态

### 交互式可视化

- **手柄热点图** — 正面/背面双视图，点击按钮热点即可跳转到对应配置；热点坐标基于 SVG 路径精确提取，与背景图像对齐
- **灵敏度曲线编辑器** — Catmull-Rom 样条插值，5 种预设（线性/指数/S 曲线/阶梯/自定义），可拖拽锚点
- **摇杆轨迹追踪器** — 双圆形画布，实时轨迹渐隐（2 秒滑动窗口），死区可视化，60fps 帧回调
- **PlayStation 风格 UI** — 纯黑 OLED 背景 + PS Blue 强调色，卡片式布局，8px 间距网格

### 生命周期与信任面

- **还原点** — 风险操作前捕获全状态快照，支持按条目验证还原
- **设备 vs 配置档案** — 只读三方差异对比（实时设备 / 已保存配置 / 上次应用），逐字段漂移高亮
- **实时手柄可视化** — 按下按键时高亮，诚实标注实时视图反映 XInput 输出
- **健康报告** — 引导式多步测量工作流，可导出
- **就绪检查** — 20 秒赛前体检
- **磨损账本** — 仅追加的包装器事件审计日志
- **模块护照** — 分侧摇杆模块指纹与纵向趋势
- **Live Verify** — 实时 XInput 摇杆 + 逐摇杆圆度读数，支持内联固件死区调节

### 多语言支持

界面支持**简体中文、英文、韩文**三种语言，运行时切换。

---

## 架构

混合架构 — Dear PyGui (DPG) 管理表单控件，WebView (HTML+SVG+JS) 管理交互可视化。

```
UI 层 (DPG)          → 只能调用 →  状态层 (AppState)
交互层 (WebView)      → 只能调用 →  状态层 (AppState) [通过 JS bridge]
状态层 (AppState)     → 只能调用 →  驱动层 (HID Service)
驱动层 (HID Service)  → 不依赖 →   任何 UI 层
```

### 目录结构

| 目录 | 职责 |
|------|------|
| `zd_app/protocol/` | 稳定的 HID 协议层（接口枚举、双句柄会话、预检） |
| `zd_app/services/` | 业务逻辑，零 UI 依赖：设置传输、应用协调器、还原点、快照差异、健康报告、磨损账本、模块护照 |
| `zd_app/storage/` | JSON/JSONL 存储，原子写入：配置档案、应用设置、还原点、上次应用记录 |
| `zd_app/ui/` | Dear PyGui 屏幕 + AppShell 协调器，含线程化 HID 作业接缝 |
| `zd_app/ui/components/` | 新 UI 组件（侧栏/顶栏/标签栏/手柄图/配置面板/曲线编辑器/摇杆追踪器） |
| `zd_app/ui/webview/` | WebView 资源（HTML+SVG+JS 交互可视化） |
| `zd_app/i18n/` | 本地化加载器 + `en.json` / `zh-CN.json` / `ko.json` |
| `tests/` | 单元测试 + 集成测试 |
| `tools/` | 构建脚本、SVG 坐标提取工具 |

### UI 组件（新架构）

- **GamepadDiagram** — 手柄正面/背面交互热点图，点击选中按钮
- **ConfigPanel** — 根据选中按钮动态渲染参数字段
- **CurveEditor** — 灵敏度曲线编辑器，Catmull-Rom 样条插值，5 种预设
- **JoystickTracker** — 摇杆轨迹追踪器，双圆形画布，实时轨迹渐隐，死区可视化

---

## 快速开始

### 从源码运行（推荐开发者）

需要 **64 位 Python 3.12**。

```powershell
python -m venv .venv-zd
.venv-zd\Scripts\pip install -r requirements.txt
.venv-zd\Scripts\pythonw main_zd.py
```

`pythonw`（无控制台窗口）优先于 `python`。如果窗口未出现，用 `python` 重新运行以查看启动错误。

### 使用流程

1. 将 ZD Ultimate Legend 手柄插入 USB 端口
2. 启动 `main_zd.py`
3. 应用自动在连接时读取当前手柄状态
4. 通过侧栏标签调整设置，点击 Apply 写入

---

## 从源码构建

**构建发布版本：**

```powershell
.\tools\build_release.ps1
.\tools\smoke_release.ps1 -DurationSeconds 5
```

**运行测试套件：**

```powershell
.venv-zd\Scripts\python -m pytest tests/ -p "test_*.py"
```

> 注：部分测试（`test_hidden_transport`、`test_preflight_*`、`test_trigger_hidden_interface` 等）依赖 Windows 的 `ctypes.windll`，只能在 Windows 上运行。

### 首次运行的 SmartScreen 提示

构建当前**未签名**（无 Authenticode 证书），首次运行时 Windows SmartScreen 可能显示 "Windows 保护了你的电脑" 提示。这是正常的——验证下载哈希后选择 **更多信息 → 仍要运行**。

---

## 兼容性

**如果你有 ZD Ultimate Legend，核心设置大概率可用。**

应用在**单台 ZD Ultimate Legend 设备**上开发和测试，已知工作固件：**v1.18**（含 8K 回报率）和 **v1.24**（含 8 点灵敏度曲线）。

ZD Ultimate Legend 有**六种手柄变体**，配不同摇杆模块和固件版本。其他固件版本、其他变体、不同摇杆模块为**尽力而为**——HID 协议可能不同，部分设置可能读取或写入异常。应用会**明确报告不支持的路径和只写字段**，而非假装写入成功。

---

## 恢复 — 如果手柄感觉异常

1. **官方 ZD 应用 → "恢复默认"** — 打开官方 ZD Gaming 应用，使用其 *恢复默认* 功能
2. **还原点** — 在侧栏打开 **还原点**，恢复较早的快照回滚应用可写的设置

---

## 技术约束（强制）

以下约束由架构和测试强制执行，非缺失功能：

- 无驱动、无虚拟设备、无输入注入
- 无宏/连发/自动化
- 无后台服务、无网络调用、无遥测、无自动更新

---

## 许可证

Project license: MIT (c) 2026 EvilHumphrey；详见 `LICENSE` 文件。捆绑的第三方组件保留原始许可证。

---

## 致谢

本项目在人类指导和审查下，借助大量 AI 协助构建。逆向工程、实现、测试编写和对抗性代码审查大部分由 AI 代理完成——所有变更经人工审查，面向硬件的变更在真实手柄上测试。

> **历史注记** — 项目原名 LegendCTL，现更名为 ZZ-ZD。winget 包标识保留原名以兼容已发布的安装包。