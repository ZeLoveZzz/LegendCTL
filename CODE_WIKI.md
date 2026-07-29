# LegendCTL Code Wiki

> **项目版本**: v2.6.2  
> **最后更新**: 2026-07-29  
> **项目类型**: Windows 桌面应用 (Python + Dear PyGui)  
> **许可证**: MIT

---

## 目录

1. [项目概述](#1-项目概述)
2. [技术架构总览](#2-技术架构总览)
3. [目录结构与模块划分](#3-目录结构与模块划分)
4. [核心数据模型](#4-核心数据模型)
5. [Protocol 层详解](#5-protocol-层详解)
6. [Services 层详解](#6-services-层详解)
7. [Storage 层详解](#7-storage-层详解)
8. [UI 层详解](#8-ui-层详解)
9. [i18n 国际化系统](#9-i18n-国际化系统)
10. [约束架构与测试防护](#10-约束架构与测试防护)
11. [构建与运行](#11-构建与运行)
12. [CI/CD 流水线](#12-cicd-流水线)
13. [数据目录结构](#13-数据目录结构)
14. [依赖关系图谱](#14-依赖关系图谱)

---

## 1. 项目概述

### 1.1 项目简介

**LegendCTL**（又名 *ZD Ultimate Legend Wrapper*）是一款**开源、免费的 Windows 桌面应用**，用于为 **ZD Ultimate Legend** 游戏手柄读取和写入固件设置。它通过标准 **USB HID Feature Report** 协议直接与手柄通信，无需官方 ZD 应用即可运行。

### 1.2 核心特性

| 类别 | 功能 |
|------|------|
| **手柄设置** | USB 轮询率 (250-8000Hz)、16×16 按键映射矩阵、摇杆死区/灵敏度曲线 (3锚点/8锚点)、轴反转、摇杆步长、扳机范围/模式/震动、灯效 (RGB)、震动强度、背键绑定 |
| **生命周期管理** | 还原点 (Restore Points)、设备 vs 配置三方差异对比、健康报告、就绪检查、磨损台账 (Wear Ledger)、模块护照 (Module Passport)、诊断包导出 |
| **实时验证** | 实时摇杆可视化、XInput 圆形度测量、死区内联调优、每字段写入结果诚实报告 |
| **信任机制** | 信任矩阵、首次连接信任卡、信任自检、导入安全检查 (Safe Import) |
| **多语言** | English、简体中文、한국어 (三语完整对等，测试门禁) |

### 1.3 设计约束（强制，非口号）

项目有十大**强制架构约束**，由测试套件门禁保证：

1. ✅ **仅 HID Feature Report 写入** (MI_02)
2. ✅ **无驱动安装**
3. ✅ **无虚拟设备**
4. ✅ **无输入注入**
5. ✅ **无游戏进程钩子**
6. ✅ **无后台服务**（关闭即退出，无残留进程）
7. ✅ **无自动化**（无宏/连发/脚本）
8. ✅ **无网络调用**（零遥测、零自动更新、零分析）
9. ✅ **诚实写入报告**（每次写入报告实际结果 + 可读字段回读验证）
10. ✅ **纯本地数据**（`%APPDATA%\ZDUltimateLegend\`，明文 JSON/JSONL）

---

## 2. 技术架构总览

### 2.1 分层架构图

```
┌─────────────────────────────────────────────────────────┐
│                     UI Layer (Dear PyGui)               │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │ AppShell  │  │  Screens/*   │  │  Components/UI   │   │
│  │ Coordinator│  │  (14个页面) │  │  Widgets/Themes  │   │
│  └────┬─────┘  └──────┬───────┘  └────────┬─────────┘   │
├───────┼───────────────┼──────────────────┼──────────────┤
│       │  Services Layer (零 UI 导入)      │              │
│  ┌────▼─────┐  ┌──────▼───────┐  ┌──────▼─────────┐    │
│  │ Settings  │  │ Restore/     │  │ Health/ Wear/  │    │
│  │ Service   │  │ Profiles/    │  │ Module/Diag    │    │
│  │ + Apply   │  │ Diff/Verify  │  │ Bundle         │    │
│  │ Coord.    │  │              │  │                │    │
│  └────┬─────┘  └──────┬───────┘  └──────┬─────────┘    │
├───────┼───────────────┼──────────────────┼──────────────┤
│       │  Storage Layer (原子写入)         │              │
│  ┌────▼─────┐  ┌──────▼───────┐  ┌──────▼─────────┐    │
│  │ Settings  │  │ Restore/     │  │ Profile/       │    │
│  │ Store     │  │ Snapshot/    │  │ Last Applied/  │    │
│  │           │  │ Wear Ledger  │  │ Module Passport│    │
│  └───────────┘  └──────────────┘  └────────────────┘    │
├─────────────────────────────────────────────────────────┤
│              Protocol Layer (HID Transport)             │
│  ┌──────────────────────────────────────────────────┐   │
│  │ hid_transport (SetupAPI + Win32 CreateFile)      │   │
│  │ preflight_visibility / trigger_interface         │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### 2.2 两个关键接缝（Seams）

架构中有两个承载性的设计接缝：

#### (1) 线程化 HID 作业接缝
- 长耗时设备操作（读取、配置应用、还原、重试脉冲）运行在**工作线程**
- `_run_hid_job` 持有 busy 标志，完成队列在渲染线程 `_tick` 顶部排空
- 任何设备触碰 UI 入口在作业进行中会拒绝（带状态提示）
- 测试可构造无 executor 的同步模式，保持字节级遗留行为

#### (2) 延迟 UI / 模态框交换接缝
- DearPyGui 的实证规律：**同一渲染帧内销毁并创建模态框，新模态不会渲染**
- 所有模态框链路由 `_defer_modal_swap` 路由：销毁帧 → 渲染一帧 → 创建帧
- 支持 coalescing keys，两个模态从不堆叠（下层隐藏后再上层）

### 2.3 应用启动流程

```
main_zd.py
  ├─ 1. logging 基础配置 + 崩溃处理器安装 (crash_reporter)
  ├─ 2. initialize_user_data_dir() → 配置 AppData 目录 + 遗留迁移
  ├─ 3. 文件日志 (5MB 轮转 × 3) + 路径清洗格式化器
  ├─ 4. SettingsStore 加载 (AppSettings)
  ├─ 5. SettingsService.start() → 尝试连接手柄
  │    ├─ 成功 → 传给 AppShell
  │    └─ 失败 → 启动 watchdog 线程每 2s 重试
  ├─ 6. VMware USB 重定向警告
  ├─ 7. 构造服务依赖图
  │    WearLedger → ModulePassport → DiagnosticBundle
  └─ 8. AppShell.run() → DPG 主循环
```

**入口文件**: [main_zd.py](file:///workspace/main_zd.py#L79-L192)

---

## 3. 目录结构与模块划分

### 3.1 顶级目录

```
/workspace
├── main_zd.py                    # 应用入口
├── pyinstaller_main_zd.spec      # PyInstaller 打包配置
├── requirements.txt              # 运行时依赖
├── requirements-build.txt        # 构建依赖
├── version_info.txt              # Windows EXE 版本资源
├── zd_app/                       # 核心包（所有生产代码）
│   ├── models.py                 # 共享数据模型（无 UI、无服务导入）
│   ├── version.py                # 版本常量
│   ├── protocol/                 # HID 协议层
│   ├── services/                 # 业务逻辑层（零 UI 导入）
│   ├── storage/                  # 持久化层（JSON/JSONL 原子写入）
│   ├── ui/                       # Dear PyGui UI 层
│   └── i18n/                     # 国际化
├── tests/                        # ~3134 个 unittest 测试
├── tools/                        # 构建/开发脚本
├── packaging/                    # winget 包配置
├── assets/                       # 字体、许可证、截图
├── docs/                         # 用户文档、架构文档
└── .github/                      # CI 工作流、Issue/PR 模板
```

### 3.2 zd_app/ 模块职责矩阵

| 模块 | 职责 | 生产代码 | 零 UI 导入 |
|------|------|:--------:|:----------:|
| `zd_app.protocol` | HID 设备枚举、接口路径发现、隐藏传输探测 | ✅ | ✅ |
| `zd_app.services` | 所有业务逻辑：设置读写、应用协调、还原点、健康报告、磨损台账、模块护照、诊断包 | ✅ | ✅ |
| `zd_app.storage` | JSON/JSONL 持久化：设置、配置文件、还原点、上次应用记录、快照编解码 | ✅ | ✅ |
| `zd_app.ui` | Dear PyGui 屏幕渲染、AppShell 协调器、主题/字体/组件 | ✅ | ❌ |
| `zd_app.i18n` | 本地化加载器、键值查找、歧义守卫 | ✅ | ✅ |
| `zd_app.models` | 共享 dataclass 模型：连接状态、Profile、设备状态、AppSettings | ✅ | ✅ |

---

## 4. 核心数据模型

所有模型定义在：[zd_app/models.py](file:///workspace/zd_app/models.py)

### 4.1 枚举类型（状态机）

| 枚举 | 值 | 用途 |
|------|----|------|
| `ConnectionState` | `no_device`, `connecting`, `connected`, `unsupported_firmware`, `wrong_mode`, `device_error` | 设备连接状态 |
| `DataFreshness` | `never_read`, `reading`, `fresh`, `stale`, `write_pending`, `write_success`, `write_failed` | 数据新鲜度 |
| `SyncStatus` | `Disconnected`, `Connected`, `Reading`, `Ready`, `Unsaved Changes`, `Applying`, `Apply Failed` | UI 页脚同步状态 |
| `DeviceClass` | `zd_ultimate_legend`, `generic_xinput`, `none` | 设备能力分级（白名单门禁） |

**设备分级门禁逻辑**（`DeviceState` 属性）：
```python
_WRITE_SUPPORTED_DEVICE_CLASSES = {"zd_ultimate_legend"}   # 仅白名单设备可 HID 写入
_LIVE_VERIFY_DEVICE_CLASSES = {"zd_ultimate_legend", "generic_xinput"}  # 任何 XInput 设备可实时测试
```

### 4.2 核心数据类

#### (1) `DeviceState` — 设备连接态

位置: [models.py#L298-L363](file:///workspace/zd_app/models.py#L298-L363)

```python
@dataclass
class DeviceState:
    product_name: str                    # 产品名称
    device_class: DeviceClass            # 能力分级
    stable_identifier: str               # 稳定标识
    firmware_version: str                # 固件版本
    active_onboard_profile: int          # 当前板载配置
    sync_status: SyncStatus              # UI 同步状态
    connection_state: ConnectionState    # 连接状态
    data_freshness: DataFreshness        # 数据新鲜度
    supported_capabilities: dict         # 支持能力映射
    summary_sources: dict                # 摘要字段来源（信任矩阵用）
    xinput_slot: int | None              # XInput 槽位
    # 属性:
    #   write_supported → bool  (写入门禁)
    #   live_verify_supported → bool (实时测试门禁)
```

#### (2) `Profile` — 配置文件（旧/遗留）

位置: [models.py#L191-L245](file:///workspace/zd_app/models.py#L191-L245)

包含：按钮映射、左右摇杆、左右扳机、修改时间、脏标记、激活标记。

#### (3) `WrapperProfile` — 包装器配置（现行）

位置: [models.py#L249-L284](file:///workspace/zd_app/models.py#L249-L284)

**命名的控制器全状态快照**，通过 `snapshot_codec` 序列化。

#### (4) `StickSettings` / `TriggerSettings` / `ButtonMapping`

- 摇杆：中心死区、外周死区、补偿、曲线预设、轴反转
- 扳机：模式、阈值、触发点、发丝扳机
- 按键映射：物理输入 ID → 动作 + 预设动作

#### (5) `AppSettings` — 应用设置

位置: [models.py#L380-L425](file:///workspace/zd_app/models.py#L380-L425)

```python
@dataclass
class AppSettings:
    language: str                        # 语言 (en/zh-CN/ko)
    auto_read_on_connect: bool           # 连接时自动读取
    developer_panels_visible: bool       # 开发者面板开关
    first_run_acknowledged: bool         # 首次运行确认
    last_reviewed_crash_timestamp: str   # 上次审阅崩溃时间
```

### 4.3 `ControllerSnapshot` — 设置服务快照

定义于: [zd_app/services/settings_service.py](file:///workspace/zd_app/services/settings_service.py)

这是**所有手柄设置的单一真值源**（Single Source of Truth）。测试门禁 `test_field_registry_drift.py` 确保所有镜像注册表与它键集对等。

包含字段：
- 标量：`polling_rate`, `step_size`, `vibration`, `deadzones`, `axis_inversion_left/right`, `sensitivity_left/right`, `trigger_left/right`, `motion_settings`
- 集合：`button_bindings` (16 槽), `back_paddle_bindings` (8 槽), `lighting_zones` (3 区)

---

## 5. Protocol 层详解

位置: [zd_app/protocol/](file:///workspace/zd_app/protocol/)

### 5.1 模块职责

| 文件 | 职责 |
|------|------|
| [hid_transport.py](file:///workspace/zd_app/protocol/hid_transport.py) | 通过 SetupAPI 枚举 HID 接口路径，提供隐藏接口发现/探测/监视能力 |
| [preflight_visibility.py](file:///workspace/zd_app/protocol/preflight_visibility.py) | 预检 MI_02 接口可见性，确认设置通道可用 |
| [trigger_interface.py](file:///workspace/zd_app/protocol/trigger_interface.py) | 双模式触发接口协调（VID_20BC 路径） |

### 5.2 核心常量

```python
PUBLIC_VENDOR_ID  = 0x413D   # ZD 手柄 VID
PUBLIC_PRODUCT_ID = 0x2104   # ZD 手柄 PID
MI02_DEVICE_INTERFACE_GUID = "{4d1e55b2-f16f-11cf-88cb-001111000030}"  # HID 标准接口 GUID
```

### 5.3 打开配置文件（HidOpenProfile）

Protocol 层定义了 4 种 Win32 CreateFile 打开模式：

| 配置名 | 用途 | DesiredAccess | Flags |
|--------|------|:-------------:|-------|
| `PUBLIC_IDENTIFY_OPEN_PROFILE` | 公开识别读写 | R\|W | Normal |
| `HIDDEN_BOOTSTRAP_OPEN_PROFILE` | 引导写入（仅写） | W | 0 |
| `HIDDEN_READBACK_OPEN_PROFILE` | 回读（重叠IO） | R\|W | OVERLAPPED |
| `HIDDEN_METADATA_PROBE_PROFILE` | 元数据探测（弱句柄） | 0 | OVERLAPPED |

元数据探测用 `DesiredAccess = 0` 打开句柄，避免共享冲突，可用于 TLC 身份确认。

### 5.4 重试机制

`RETRYABLE_OPEN_ERRORS` 在后续 PnP ARRIVAL 事件上重试：
- `ERROR_SHARING_VIOLATION` (32) — 共享模式冲突
- `ERROR_FILE_NOT_FOUND` (2) — 瞬态接口注销
- `ERROR_PATH_NOT_FOUND` (3) — 瞬态路径消失

---

## 6. Services 层详解

位置: [zd_app/services/](file:///workspace/zd_app/services/)

**硬性规则**: Services 层零 UI 导入（由 `test_import_boundary.py` 门禁）。

### 6.1 核心服务地图

```
services/
├── settings_service.py              # HID 协议原始传输 + 字段编解码
├── settings_apply_coordinator.py    # Apply 流水线：字段级 trailer + settle + 重试
├── write_verification.py            # 写入后回读验证 + 结果分类
├── restore_point_service.py         # 还原点捕获/恢复（含新鲜度规则）
├── profile_service.py               # 配置文件 CRUD
├── device_service.py                # 设备存在轮询 + 检测
├── diagnostics_service.py           # 诊断日志
├── xinput_poll_service.py           # XInput 实时轮询（Live Verify）
├── preflight_service.py             # 传输预检
├── snapshot_diff.py                 # 三方快照差异（设备/配置/上次应用）
├── trust_matrix.py                  # 信任矩阵（值来源标注）
├── trust_self_check.py              # 信任自检
├── crash_reporter.py                # 崩溃处理 + 报告
├── locale_router.py                 # 语言切换路由
├── title_manager.py                 # 窗口标题管理
├── circularity.py                   # 摇杆圆形度算法
├── button_binding_formatting.py     # 按键绑定格式化
├── restore_field_formatting.py      # 还原字段显示格式化
├── share_card.py                    # 分享卡模型
├── markdown_safety.py               # Markdown 安全渲染
├── path_scrub.py                    # 日志路径清洗（隐私）
├── import_classifier.py             # 导入分类器（Safe Import）
├── official_app_summary_service.py  # 官方应用摘要读取
├── model_fingerprint.py             # 模块指纹
├── host_environment.py              # 宿主机环境检测（VMware 等）
├── compatibility_report.py          # 兼容性报告
├── standalone_trigger_service.py    # 独立触发服务（可选，守卫保护）
├── _subprocess_helpers.py           # 子进程辅助
├── _log_entry.py                    # 日志条目组合
│
├── health_report/                   # 健康报告子模块
│   ├── service.py                   # 主服务
│   ├── measurements.py              # 测量步骤
│   ├── sample_capture.py            # 采样捕获
│   ├── quick_check.py               # 快速检查 (20s)
│   ├── json_export.py / markdown_export.py  # 导出
│   ├── models.py / boundary.py
│
├── wear_ledger/                     # 磨损台账（仅追加审计日志）
│   ├── service.py / models.py
│
├── module_passport/                 # 模块护照（摇杆模块指纹 + 趋势）
│   ├── service.py / characterize.py / trend_analysis.py
│
├── diagnostic_bundle/               # 诊断包（路径清洗后导出）
│   ├── service.py / boundary.py
│
└── restore_points/                  # 还原点子模块
    └── boundary.py
```

### 6.2 SettingsService — HID 设置协议

位置: [zd_app/services/settings_service.py](file:///workspace/zd_app/services/settings_service.py)

这是**最底层的协议服务**，所有对手柄的读写都通过它。

#### (1) 协议魔术字节

```python
HID_FEATURE_REPORT_SIZE = 65
MAGIC_WRITE_PREFIX         = b"\x10\x55\xaa\x51"  # 写入请求前缀
MAGIC_READ_QUERY_PREFIX    = b"\x10\x55\xaa\x50"  # 读取查询前缀
MAGIC_READ_RESPONSE_PREFIX = b"\x30\x55\xaa\xd0"  # 读取响应前缀
MAGIC_WRITE_ACK_PREFIX     = b"\x30\x55\xaa\xd1"  # 写入确认前缀
```

#### (2) 分类目录（Category Codes）

| 类别 | 代码 | 字段 |
|------|:----:|------|
| 轮询率 | `0x11` | PollingRate 枚举 (250-8000Hz) |
| 步长 | `0x0D` | 1-255 (默认 73, fw 1.24) |
| 按键绑定 | `0x02` | 16 槽 × ButtonSlot |
| 摇杆死区 | `0x09` | 4 区死区值 |
| 灵敏度 (3锚点) | `0x06` | 3 锚点曲线 |
| 灵敏度 (8锚点) | `0x86` | fw 1.24+ 能力探测 |
| 轴反转 | `0x07` | 左/右 |
| 扳机设置 | `0x0A` | 模式/范围 |
| 震动 | `0x0C` | 强度 0-100 |
| 体感 | `0x0B` | 陀螺仪映射 |
| 灯效 | `0x10` | 3 区 × RGB+模式+亮度 |
| 背键主通道 | `0x03` | |
| 背键绑定 | `0x05` | |
| 背键批量 | `0x12` | 8 背键 (M1-M4, LM, RM, LK, RK) |

#### (3) 验证/重试写入器（针对静默拒绝）

固件已知 quirk：多字段突发写入中，**最后一个同类写入**常被静默拒绝（WriteFile 返回 OK 但固件不提交）。

- `set_step_size_verified()`：写入 → settle 100ms → 回读 → 不匹配重试，最多 3 次
- `set_zone_lighting_verified()`：同上模式，针对灯效 3 区
- 按钮绑定读取：单次读取偶发丢槽，重试 3 次

### 6.3 SettingsApplyCoordinator — Apply 流水线

位置: [zd_app/services/settings_apply_coordinator.py](file:///workspace/zd_app/services/settings_apply_coordinator.py)

**每字段写入流水线**：

```
字段写入请求
   │
   ▼
┌────────────────────┐
│ 1. Trailer 写入     │  (固件 quirk：多字段突发会静默拒绝，
│    + Settle 延迟    │   trailer + settle + 单次重试缓解)
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ 2. WriteOutcome     │  (同步 WriteFile 调用结果: OK/FAIL)
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ 3. Post-Apply 回读  │  (每可读字段对比)
│    Sweep            │  matched / mismatched / could-not-verify
└────────────────────┘
```

### 6.4 RestorePointService — 还原点服务

位置: [zd_app/services/restore_point_service.py](file:///workspace/zd_app/services/restore_point_service.py)

#### (1) 捕获流程

```
1. 调用 settings_service.get_*() 逐字段新鲜读取
   └─ 每调用 None = 读失败（覆盖率地图需要）
2. 字段足够 → 以 CaptureSource.FRESH_READ 构建快照
3. 否则若有最近缓存快照 (<30s) → FRESH_READ 降级为 CACHED_SNAPSHOT
4. 否则 → 返回 None (无法捕获)
```

#### (2) 恢复流程

```
恢复 RP N
  ├─ 1. 先捕获 before_restore RP (当前状态)
  ├─ 2. 过滤快照到可写字段 + 可选字段子集
  ├─ 3. 通过 SettingsApplyCoordinator 应用
  ├─ 4. 新鲜回读控制器
  ├─ 5. 每尝试字段: expected vs read-back 对比
  └─ 6. 构建 RestoreResult + 持久化 RestoreAttemptRecord
```

#### (3) 字段分类模型

| 分类 | 包含 | 可恢复 |
|------|------|:------:|
| `DEVICE` | polling_rate, step_size | ✅ |
| `FEEL` | deadzones, axis_inversion, sensitivity, triggers | ✅ |
| `LAYOUT` | button_bindings, back_paddle_bindings | ✅ |
| `COSMETIC` | vibration, lighting_zones | ✅ |
| `UNSUPPORTED` | motion_settings | ❌ |

### 6.5 SnapshotDiff — 三方差异引擎

位置: [zd_app/services/snapshot_diff.py](file:///workspace/zd_app/services/snapshot_diff.py)

支持三方对比：
- **设备实时** vs **选中配置文件** vs **上次应用记录**
- 漂移检测（drift detection）
- 8 点灵敏度曲线 riders 折叠
- 不可读 vs 缺失的诚实区分（由 provenance maps 驱动）

### 6.6 HealthReportService — 健康报告

引导式多步骤测量工作流，可导出 JSON/Markdown。子模块：
- `measurements.py`：步骤定义
- `sample_capture.py`：采样捕获逻辑
- `quick_check.py`：20 秒赛前快速检查
- `json_export.py` / `markdown_export.py`：导出器

### 6.7 WearLedgerService — 磨损台账

位置: [zd_app/services/wear_ledger/](file:///workspace/zd_app/services/wear_ledger/)

**仅追加（append-only）审计日志**，记录包装器事件：
- `SESSION_START` / `SESSION_END` — 会话起止
- `PROFILE_APPLY` — 配置应用
- `SLIDER_WRITE` — 滑块写入
- `RP_CAPTURE` / `RP_RESTORE` / `RP_DELETE` — 还原点操作

### 6.8 ModulePassportService — 模块护照

位置: [zd_app/services/module_passport/](file:///workspace/zd_app/services/module_passport/)

左右摇杆模块指纹 + 纵向趋势分析：
- `characterize.py`：模块特征提取
- `trend_analysis.py`：趋势分析算法

### 6.9 DiagnosticBundleService — 诊断包

位置: [zd_app/services/diagnostic_bundle/](file:///workspace/zd_app/services/diagnostic_bundle/)

操作者触发的**路径消毒可分享证据导出**：
- 从多个子系统收集证据
- 路径绝对路径清洗（隐私）
- 打包为可分享压缩包

### 6.10 CrashReporter — 崩溃捕获

位置: [zd_app/services/crash_reporter.py](file:///workspace/zd_app/services/crash_reporter.py)

`install_crash_handlers(user_data_dir)` 在启动最早期安装（`main_zd.py#L89`），确保任何后续阶段崩溃（DPG 初始化、服务启动、线程派生）都被捕获。

### 6.11 PathScrub — 日志隐私清洗

位置: [zd_app/services/path_scrub.py](file:///workspace/zd_app/services/path_scrub.py)

`PathScrubbingFormatter` 安装在文件日志处理器上（`main_zd.py#L110-L112`），**从每行日志中剥离绝对路径（含用户名）**，因为 SUPPORT.md 要求用户附加此文件提交 bug 报告。

---

## 7. Storage 层详解

位置: [zd_app/storage/](file:///workspace/zd_app/storage/)

### 7.1 核心原则

- **全部 JSON/JSONL 明文**
- **全部原子写入**：temp-file → flush+fsync → os.replace()
- **腐败降级**：解析错误不崩溃，以披露卡片或无操作日志降级
- **读守卫**：`read_guarded_json()` 限制文件大小/深度，防止递归崩溃

### 7.2 各存储类

| 类 | 文件 | 存储位置 | 格式 |
|----|------|----------|------|
| `SettingsStore` | [settings_store.py](file:///workspace/zd_app/storage/settings_store.py) | `<data>/settings.json` | JSON |
| `ProfileStore` | [profile_store.py](file:///workspace/zd_app/storage/profile_store.py) | `<data>/profiles/` | 每文件 JSON |
| `WrapperProfileStore` | [wrapper_profile_store.py](file:///workspace/zd_app/storage/wrapper_profile_store.py) | `<data>/wrapper_profiles/` | 每文件 JSON |
| `RestorePointStore` | [restore_point_store.py](file:///workspace/zd_app/storage/restore_point_store.py) | `<data>/restore_points/` | 每文件 JSON + 保留规则 |
| `LastAppliedStore` | [last_applied_store.py](file:///workspace/zd_app/storage/last_applied_store.py) | `<data>/last_applied.json` | JSON |
| `SnapshotCodec` | [snapshot_codec.py](file:///workspace/zd_app/storage/snapshot_codec.py) | (编解码函数) | `snapshot_from_dict`/`snapshot_to_dict` |

### 7.3 用户数据目录路由

`settings_store.initialize_user_data_dir()`:

```python
if ZDUL_DATA_DIR 环境变量 → 使用该路径
elif frozen (打包 EXE)    → %APPDATA%\ZDUltimateLegend\
else (源码运行)           → ./zd_data/  (gitignore)
```

**遗留迁移机制**：打包 EXE 首次启动时，若 `./zd_data` 存在而目标目录为空，自动一次性迁移到 AppData（带状态标记 + 中断恢复 + 防重复）。

### 7.4 RestorePointStore 保留规则

位置: [restore_point_store.py](file:///workspace/zd_app/storage/restore_point_store.py)

- 最大数量：**50** 个
- 最大磁盘：**25 MB**
- 修剪顺序：最旧自动创建优先
- 保留规则：每设备身份保留最新 `first_readable_connect`
- 永不修剪：`protect` 集合（用户保护的还原点）
- 文件名格式：`YYYYMMDD-HHMMSS_<trigger>_<6hex>.json`

---

## 8. UI 层详解

位置: [zd_app/ui/](file:///workspace/zd_app/ui/)

UI 框架: **Dear PyGui 2.3.1** (ImGui 风格的 Python 绑定)

### 8.1 AppShell — 协调器

位置: [zd_app/ui/app_shell.py](file:///workspace/zd_app/ui/app_shell.py)

**这是 UI 的心脏**，负责：
- 所有服务注入（构造时可替换 fakes 用于测试）
- DPG 上下文创建、主题/字体/组件注册
- 侧边栏路由（14 个页面）
- HID 作业调度（`threaded_hid_executor`）+ 忙标志管理
- 模态框延迟交换机制（`_defer_modal_swap`）
- 页脚状态（Read/Apply/Delete 按钮）
- 设备变化监听 + 自动刷新
- 信任前门、首次运行确认、崩溃审阅模态

**生产构造参数** (`main_zd.py#L151-L170`):
```python
AppShell(
    device_service=DeviceService(),
    profile_service=ProfileService(ProfileStore()),
    diagnostics_service=DiagnosticsService(),
    settings_store=settings_store,
    settings_service=settings_service_for_ui,
    wrapper_profile_store=WrapperProfileStore(),
    last_applied_store=LastAppliedStore(),
    wear_ledger_service=wear_ledger_service,
    module_passport_service=module_passport_service,
    diagnostic_bundle_service=diagnostic_bundle_service,
    hid_executor=threaded_hid_executor,  # ← 生产用多线程
)
```

### 8.2 侧边栏屏幕（14 + 3 遗留）

| 屏幕 | 文件 | 说明 |
|------|------|------|
| **Home** | [screens/home.py](file:///workspace/zd_app/ui/screens/home.py) | 首页：设备/配置状态 + 信任卡 + 最近活动 |
| **Controller** | [screens/controller.py](file:///workspace/zd_app/ui/screens/controller.py) | 控制器设置主界面（所有设置分类） |
| **Device vs Profile** | [screens/device_vs_profile.py](file:///workspace/zd_app/ui/screens/device_vs_profile.py) | 三方差异：实时/配置/上次应用 |
| **Live Verify** | [screens/live_verify.py](file:///workspace/zd_app/ui/screens/live_verify.py) | 实时摇杆可视化 + 圆形度测量 |
| **Restore Points** | [screens/restore_points.py](file:///workspace/zd_app/ui/screens/restore_points.py) | 还原点列表/预览/恢复/删除 |
| **Health Report** | [screens/health_report.py](file:///workspace/zd_app/ui/screens/health_report.py) | 引导式健康报告流程 |
| **Readiness Check** | [screens/readiness_check.py](file:///workspace/zd_app/ui/screens/readiness_check.py) | 20 秒赛前检查 |
| **Modules** | [screens/modules.py](file:///workspace/zd_app/ui/screens/modules.py) | 模块护照 + 导出 |
| **Wear Ledger** | [screens/wear_ledger.py](file:///workspace/zd_app/ui/screens/wear_ledger.py) | 仅追加操作日志 |
| **Diagnostics** | [screens/diagnostics.py](file:///workspace/zd_app/ui/screens/diagnostics.py) | 诊断日志 + 诊断包导出 |
| **Preferences** | [screens/preferences.py](file:///workspace/zd_app/ui/screens/preferences.py) | 应用设置（语言等） |
| **Safe Import** | [screens/safe_import.py](file:///workspace/zd_app/ui/screens/safe_import.py) | 配置文件安全导入（开发者门控） |
| **About** | [screens/about.py](file:///workspace/zd_app/ui/screens/about.py) | 版本/许可证/链接 |
| 遗留 Buttons | [screens/legacy/buttons.py](file:///workspace/zd_app/ui/screens/legacy/buttons.py) | 旧版按键界面（默认隐藏） |
| 遗留 Sticks | [screens/legacy/sticks.py](file:///workspace/zd_app/ui/screens/legacy/sticks.py) | 旧版摇杆界面 |
| 遗留 Triggers | [screens/legacy/triggers.py](file:///workspace/zd_app/ui/screens/legacy/triggers.py) | 旧版扳机界面 |

### 8.3 UI 子模块

| 文件 | 职责 |
|------|------|
| [components.py](file:///workspace/zd_app/ui/components.py) | 通用卡片、表格、破坏性主题 |
| [themes.py](file:///workspace/zd_app/ui/themes.py) | 全局颜色、间距、主题注册 (COLORS, SPACE_SM/MD/LG) |
| [fonts.py](file:///workspace/zd_app/ui/fonts.py) | 字体注册：Inter/JetBrainsMono/NotoSansSC/NotoSansKR |
| [typography.py](file:///workspace/zd_app/ui/typography.py) | 排版辅助（屏幕标题等） |
| [choice_labels.py](file:///workspace/zd_app/ui/choice_labels.py) | 选择项 ↔ 显示值转换（canonical/display） |
| [localized_dpg.py](file:///workspace/zd_app/ui/localized_dpg.py) | Dear PyGui i18n 包装 |
| [onboarding.py](file:///workspace/zd_app/ui/onboarding.py) | 首次运行引导 |
| [right_rail.py](file:///workspace/zd_app/ui/right_rail.py) | 右侧辅助面板 |
| [trust_front_door.py](file:///workspace/zd_app/ui/trust_front_door.py) | 信任前门（首次连接） |
| [trust_labels.py](file:///workspace/zd_app/ui/trust_labels.py) | 信任矩阵标签 |
| [safe_import_model.py](file:///workspace/zd_app/ui/safe_import_model.py) | Safe Import 风险分类 + 过滤 |
| [safe_import_badges.py](file:///workspace/zd_app/ui/safe_import_badges.py) | Safe Import 徽标渲染 |
| [diagnostic_bundle_preview.py](file:///workspace/zd_app/ui/diagnostic_bundle_preview.py) | 诊断包预览模态 |
| [support_reference.py](file:///workspace/zd_app/ui/support_reference.py) | 支持指南引用 |
| [controller_diagram_layout.py](file:///workspace/zd_app/ui/controller_diagram_layout.py) | 控制器图布局（Live Verify 用） |
| [widgets/trust_ritual.py](file:///workspace/zd_app/ui/widgets/trust_ritual.py) | 信任仪式（Trust Ritual）小组件 |

### 8.4 关键节流/防抖常量

| 常量 | 值 | 用途 |
|------|:--:|------|
| `MANUAL_DEVICE_WRITE_RP_WINDOW_S` | 7s | 手动设置写前还原点防抖（拖拽风暴合并） |
| `SLIDER_LIVE_WRITE_THROTTLE_S` | 0.15s | 滑块实时写入节流（约 6-7 writes/s） |
| `READ_TIMEOUT_RETRY_SETTLE_S` | 0.3s | 首次读超时重试前等待 |
| `POST_APPLY_READ_SETTLE_S` | 0.25s | Apply 后自动回读前固件安静时间 |
| `HID_JOB_STALL_WARN_S` | 30s | HID 作业卡住警告阈值 |

---

## 9. i18n 国际化系统

位置: [zd_app/i18n/](file:///workspace/zd_app/i18n/)

### 9.1 支持语言

| 语言代码 | 文件 | 测试门禁 |
|----------|------|:--------:|
| `en` | [locales/en.json](file:///workspace/zd_app/i18n/locales/en.json) | ✓ (默认) |
| `zh-CN` | [locales/zh-CN.json](file:///workspace/zd_app/i18n/locales/zh-CN.json) | ✓ |
| `ko` | [locales/ko.json](file:///workspace/zd_app/i18n/locales/ko.json) | ✓ |

**测试保障** (`test_i18n.py`)：
- `set(en) == set(zh-CN) == set(ko)`：键集完全对等
- 无空值（空 tombstone 检测 `_is_tombstone_text`）

### 9.2 API

```python
from zd_app.i18n import set_locale, get_locale, t, translate_literal

set_locale("zh-CN")                     # 切换语言（不支持回退到 en）
value = t("nav.home")                   # 按键查找: "首页"
value = translate_literal("Controller") # 按英语字面量反查键
```

### 9.3 歧义守卫

`_REVIEWED_AMBIGUOUS` 白名单（`i18n/__init__.py#L33-L165`）：

**问题**：同一个英语字面量（如 "Home"、"Left"、"Right"、"Back"、"Controller"）在不同上下文下有不同的非英语翻译。直接传递英语字面量给 `translate_literal()` 会取第一个 JSON 顺序的键，可能错译。

**守卫机制**：
- 加载英语时构建反向映射 `_reverse_en`
- 扫描所有非默认语言，找出「多键共享英语字面量但非英语翻译实际不同」的条目
- 若不在审查白名单中 → 启动时 WARNING 日志警告
- 白名单是「精确键集子集」守卫：新增同名新键若不在审查集中 → 重新触发警告

### 9.4 语言代码规范化

`_normalize_language_code()` 将历史遗留代码（如 "简体中文"、"zh"、"한국어" 等）统一到标准 `en/zh-CN/ko`。

---

## 10. 约束架构与测试防护

### 10.1 约束门禁测试

项目声明的十大约束不是口号，由 `tests/` 中的测试门禁强制：

| 测试文件 | 防护内容 |
|----------|----------|
| `test_forbidden_phrases.py` (多处 - health_report, restore_point) | 用户文案禁止过度声明（如 "calibrated"、"ban-safe"）；仅 `boundary.py` 中声明段为白名单 |
| `test_field_registry_drift.py` | `ControllerSnapshot` 是唯一真值源；所有镜像注册表键集对等 + 分类值钉死 |
| `test_i18n.py` | 三语键集完全对等、无空值 |
| `test_import_boundary.py` | 发布应用只导入 `zd_app/`、`main_zd.py`、构建工具；Services 零 UI 导入 |
| `test_public_vocab_hygiene.py` | 公开词汇卫生（不泄露维护者用户名等） |
| `test_shipped_source_hygiene.py` | 发布源代码卫生 |
| `test_xinput.py` / `test_any_xinput_readonly.py` | 禁止在非 ZD 设备上尝试 HID 写入（只读 XInput 测试独立） |

### 10.2 测试套件概览

- 总数：约 **3,134** 个 unittest 测试
- 服务测试：无头运行（不创建真实 DPG 上下文）
- 屏幕测试：使用 patched Dear PyGui 记录部件调用（不真实渲染）
- 真实 DPG 行为（如模态框规律）由 `tools/` 中的手动台架工具额外钉死

### 10.3 测试约定

```powershell
# Windows · Python 3.12 · dearpygui 安装
python -m unittest discover tests -p "test_*.py"

# 注意：teardown 时 exit code 139 = 已知 DPG 段错误假象
# CI 决定 pass/fail 依据 unittest 摘要行 (OK/FAILED)，不是原始退出码
```

---

## 11. 构建与运行

### 11.1 环境要求

| 项 | 要求 |
|----|------|
| 操作系统 | Windows 10 / 11 |
| Python | **64-bit Python 3.12** (开发测试版本)；CI 打包使用 Python 3.13 |
| 依赖 | `dearpygui==2.3.1`（仅；`hid` 可选，守卫保护） |
| 构建 | `pyinstaller==6.21.0` |
| 可选安装器 | [Inno Setup 6](https://jrsoftware.org/isdl.php)（`C:\Program Files (x86)\Inno Setup 6\ISCC.exe`） |

### 11.2 源码运行（开发模式）

```powershell
# 1. 创建虚拟环境
python -m venv .venv-zd

# 2. 安装运行时依赖
.venv-zd\Scripts\pip install -r requirements.txt

# 3. 运行（推荐 pythonw，无控制台窗口）
.venv-zd\Scripts\pythonw main_zd.py

# 3a. 若窗口不显示，用 python 查看启动错误
.venv-zd\Scripts\python main_zd.py
```

### 11.3 一键环境搭建

```powershell
tools\setup_dev_env.ps1
# 自动: 定位 Python 3.12 → 创建 .venv-zd → 安装 requirements-build.txt
```

脚本: [tools/setup_dev_env.ps1](file:///workspace/tools/setup_dev_env.ps1)

### 11.4 运行测试套件

```powershell
.venv-zd\Scripts\python -m unittest discover tests -p "test_*.py"
```

### 11.5 构建发布

```powershell
# 构建: PyInstaller 便携目录 + ZIP + (Inno 存在时) 安装器 EXE + SHA256SUMS.txt
.\tools\build_release.ps1

# 冒烟测试: 启动 EXE 运行指定秒数后关闭
.\tools\smoke_release.ps1 -DurationSeconds 5

# 本地安装（镜像最新构建到 local-install\ + 刷新快捷方式）
.\tools\install_local.ps1
```

### 11.6 PyInstaller 打包配置

文件: [pyinstaller_main_zd.spec](file:///workspace/pyinstaller_main_zd.spec)

**打包数据**:
- 字体 (`assets/fonts/*.ttf`, `*.otf`)
- 许可证 (`assets/licenses/*.txt`)
- 语言文件 (`zd_app/i18n/locales/*.json`)
- PowerShell 探测脚本
- LICENSE / NOTICE

**排除模块**（减少打包体积）:
`tkinter`, `unittest`, `pytest`, `frida`, `IPython`, `jupyter`

**输出**:
```
dist/ZDUltimateLegend/
  └── ZD Ultimate Legend.exe      # console=False（无控制台窗口）
      + version_info.txt（Windows 版本资源）
```

### 11.7 发行物

| 分发形式 | 说明 |
|----------|------|
| **便携 ZIP** (推荐) | `ZDUltimateLegend-v<version>-windows.zip` — 免安装、免管理员 |
| **安装器 EXE** | `ZDUltimateLegend-v<version>-Setup.exe` — Inno Setup，需管理员，开始菜单 + 卸载 |
| **winget** | `winget install EvilHumphrey.LegendCTL`（等待 Microsoft 审核） |

---

## 12. CI/CD 流水线

文件: [.github/workflows/ci.yml](file:///workspace/.github/workflows/ci.yml)

### 12.1 CI 工作流程

```
Push/PR to main
   │
   ├─ test (Windows · Python 3.12)
   │   ├─ 安装 requirements.txt
   │   ├─ unittest discover tests
   │   └─ 判定规则:
   │       ├─ 输出含 "FAILED (" → 失败
   │       ├─ 输出含 "OK " → 通过（忽略 exit code 139 = DPG 段错误假象）
   │       └─ 其他 → 崩溃/异常失败
   │
   └─ audit (Windows · Python 3.12)
       ├─ 安装 requirements.txt + requirements-build.txt + pip-audit
       └─ pip-audit 扫描已知 CVE
          （不适用项可 --ignore-vuln 并注明理由）
```

### 12.2 权限

- `permissions: contents: read` — 最小权限令牌
- 使用 `pull_request`（非 `pull_request_target`），Fork PR 代码永远拿不到仓库 Secrets

### 12.3 并发

`concurrency: ci-${{ github.ref }}` — 同分支新推送取消进行中的旧构建。

---

## 13. 数据目录结构

### 13.1 生产模式（打包 EXE）

```
%APPDATA%\ZDUltimateLegend\
├── settings.json                      # AppSettings
├── last_applied.json                  # 上次应用快照记录
├── .zd_migration_state                # 遗留 zd_data → AppData 迁移标记
│
├── logs/                              # 应用日志（5 MB 轮转，保留 3 份）
│   └── zd_wrapper.log
│
├── crash_reports/                     # 崩溃报告（crash_reporter 生成）
│
├── wrapper_profiles/                  # 用户保存的包装器配置
│   └── <name>.json                    # WrapperProfile (含 ControllerSnapshot)
│
├── restore_points/                    # 还原点
│   └── YYYYMMDD-HHMMSS_<trigger>_<suffix>.json
│
├── wear_ledger/                       # 磨损台账（仅追加 JSONL）
│
├── module_passport/                   # 模块护照数据
│
├── health_reports/                    # 健康报告导出
│
└── diagnostic_bundles/                # 诊断包生成目录
```

### 13.2 源码运行模式

```
./zd_data/   (gitignored)
└── 结构同上
```

---

## 14. 依赖关系图谱

### 14.1 外部依赖

文件: [requirements.txt](file:///workspace/requirements.txt)

| 包 | 版本 | 用途 | 必填 |
|----|:----:|------|:----:|
| `dearpygui` | **2.3.1** | UI 框架（ImGui 风格） | ✅ |
| `hid` | - | 独立触发 HID 路径（守卫保护，不默认导入） | ❌ |

文件: [requirements-build.txt](file:///workspace/requirements-build.txt)

| 包 | 版本 | 用途 |
|----|:----:|------|
| （上述全部） | - | 运行时 |
| `pyinstaller` | **6.21.0** | 打包 EXE |

### 14.2 内部模块依赖方向

```
main_zd.py (入口)
  │
  ├─→ zd_app.version                   (无依赖)
  ├─→ zd_app.models                    (仅依赖 i18n)
  │     └──→ zd_app.i18n
  │
  ├─→ zd_app.protocol/*                (仅依赖 Python stdlib)
  │
  ├─→ zd_app.services/*                (分层依赖，零 UI)
  │     ├──→ zd_app.models
  │     ├──→ zd_app.protocol
  │     ├──→ zd_app.storage
  │     └──→ zd_app.i18n
  │
  ├─→ zd_app.storage/*                 (分层依赖，零 UI)
  │     ├──→ zd_app.models
  │     └──→ zd_app.services (path_scrub 等小工具)
  │
  └─→ zd_app.ui/*                      (全栈依赖)
        ├──→ zd_app.models
        ├──→ zd_app.services
        ├──→ zd_app.storage
        ├──→ zd_app.i18n
        └──→ dearpygui (外部)
```

**核心约束**：
- `services/` **不得**导入 `ui/` 任何内容（测试门禁）
- `storage/` **不得**导入 `ui/` 任何内容
- `protocol/` 零上层依赖（仅 stdlib）
- 只有 `main_zd.py` 和 `ui/` 可以导入 `dearpygui`

### 14.3 服务依赖注入图（生产构造）

```
SettingsStore
    │
    ▼
SettingsService ──→ SettingsApplyCoordinator ──→ RestorePointService
    │                                                    │
    │                                                    ▼
    │                                            RestorePointStore
    │
    ├──────────────────────────────────────────────────────────┐
    │                                                          │
    ▼                                                          ▼
DeviceService  ProfileService(ProfileStore)  DiagnosticsService
    │              │
    └──────┬───────┘
           │
           ▼
        AppShell ◄── WrapperProfileStore
           │        ◄── LastAppliedStore
           │        ◄── WearLedgerService
           │        ◄── ModulePassportService (依赖 WearLedger)
           │        ◄── DiagnosticBundleService (依赖 ModulePassport + WearLedger)
           │        ◄── threaded_hid_executor
           │
           ▼
        Dear PyGui Render Loop
```

---

## 附录 A：文件快速索引表

| 查找内容 | 跳转位置 |
|----------|----------|
| 应用入口 | [main_zd.py](file:///workspace/main_zd.py) |
| 版本号 | [zd_app/version.py](file:///workspace/zd_app/version.py) |
| 核心模型 (dataclass) | [zd_app/models.py](file:///workspace/zd_app/models.py) |
| HID 协议常量 + 编解码 | [zd_app/services/settings_service.py](file:///workspace/zd_app/services/settings_service.py) |
| 设置应用流水线 | [zd_app/services/settings_apply_coordinator.py](file:///workspace/zd_app/services/settings_apply_coordinator.py) |
| 写入验证逻辑 | [zd_app/services/write_verification.py](file:///workspace/zd_app/services/write_verification.py) |
| 还原点服务 | [zd_app/services/restore_point_service.py](file:///workspace/zd_app/services/restore_point_service.py) |
| 还原点存储 | [zd_app/storage/restore_point_store.py](file:///workspace/zd_app/storage/restore_point_store.py) |
| UI 协调器 (AppShell) | [zd_app/ui/app_shell.py](file:///workspace/zd_app/ui/app_shell.py) |
| HID 传输枚举 | [zd_app/protocol/hid_transport.py](file:///workspace/zd_app/protocol/hid_transport.py) |
| i18n 加载 + 歧义守卫 | [zd_app/i18n/__init__.py](file:///workspace/zd_app/i18n/__init__.py) |
| 数据目录初始化 | [zd_app/storage/settings_store.py](file:///workspace/zd_app/storage/settings_store.py) |
| 快照三方差异 | [zd_app/services/snapshot_diff.py](file:///workspace/zd_app/services/snapshot_diff.py) |
| 快照编解码 | [zd_app/storage/snapshot_codec.py](file:///workspace/zd_app/storage/snapshot_codec.py) |
| 圆形度算法 | [zd_app/services/circularity.py](file:///workspace/zd_app/services/circularity.py) |
| 路径隐私清洗 | [zd_app/services/path_scrub.py](file:///workspace/zd_app/services/path_scrub.py) |
| 崩溃处理器 | [zd_app/services/crash_reporter.py](file:///workspace/zd_app/services/crash_reporter.py) |
| 健康报告 | [zd_app/services/health_report/](file:///workspace/zd_app/services/health_report/) |
| 磨损台账 | [zd_app/services/wear_ledger/](file:///workspace/zd_app/services/wear_ledger/) |
| 模块护照 | [zd_app/services/module_passport/](file:///workspace/zd_app/services/module_passport/) |
| 诊断包 | [zd_app/services/diagnostic_bundle/](file:///workspace/zd_app/services/diagnostic_bundle/) |
| 技术架构文档（用户向） | [docs/ARCHITECTURE.md](file:///workspace/docs/ARCHITECTURE.md) |
| 构建脚本 | [tools/build_release.ps1](file:///workspace/tools/build_release.ps1) |
| CI 配置 | [.github/workflows/ci.yml](file:///workspace/.github/workflows/ci.yml) |
| PyInstaller 配置 | [pyinstaller_main_zd.spec](file:///workspace/pyinstaller_main_zd.spec) |
| 全部测试 | [tests/](file:///workspace/tests/) |

---

*本文档基于 LegendCTL v2.6.2 代码库自动生成。如需更新，请在代码变更后重新运行 Code Wiki 生成流程。*
