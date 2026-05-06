---
title: Bluedroid 一个月学习计划
tags:
  - bluedroid
  - bluetooth
  - study
created: 2026-05-06
status: active
---

# Bluedroid 一个月学习计划

> [!tip] 核心学习思路
> 不要从头到尾读代码，要**以问题驱动、以数据流为线索**。
> 每个模块都围绕一个核心问题：
> **"一个蓝牙操作从触发到完成，数据和控制流经过哪些地方？"**

## 架构速查

```
Android 框架
    ↑
BTIF 层 (btif/)          — HAL 接口，连接 Android 框架与协议栈
    ↑
BTA 层  (bta/)           — 各蓝牙 Profile 的业务逻辑
    ↑
Stack 层 (stack/)        — L2CAP、RFCOMM、SDP、GATT、SMP 等核心协议
    ↑
HCI 层  (hci/)           — 与蓝牙控制器通信，支持 H4/UART 和 MCT/USB
    ↑
蓝牙控制器硬件
```

---

## 第一周：基础 + 全局架构

**目标：建立整体心智模型，能画出完整架构图**

### Day 1-2：环境与工具

- 阅读 [[btsnoop_net_guide]]、`doc/properties.md`
- 搭建 Android 编译环境，能成功编译 `bluetooth.default.so`
- 工具准备：cscope/ctags 或 clangd 建立代码索引

### Day 3-4：启动流程

- 入口：`main/bte_main.c` → `bte_main_boot_entry()`
- 追踪初始化链：GKI 初始化 → HCI 打开 → 各层 enable
- 重点读：`btif/src/btif_core.c`

### Day 5-7：协议分层精读

- 从上到下通读各层头文件（不读实现，只看接口）
- 重点：`include/` 下的 `bt_types.h`、`hcimsgs.h`、`l2cdefs.h`

> [!todo] 产出
> 手绘一张完整的模块依赖图

---

## 第二周：核心协议栈

**目标：深入理解 HCI、L2CAP、RFCOMM 三层**

### Day 8-10：HCI 层

- `hci/src/bt_hci_bdroid.c` — 初始化与生命周期
- `hci/src/hci_h4.c` — H4 串口协议帧解析
- `hci/src/btsnoop.c` — 抓包格式

> [!example] 实验
> 用 Wireshark 抓 BTSnoop，对照代码理解包格式
> 参考 [[btsnoop_net_guide]]

### Day 11-13：L2CAP

- `stack/l2cap/` — 重点读 `l2c_api.c`、`l2c_csm.c`（状态机）
- 理解 Channel 建立、流控、分片重组

> [!todo] 产出
> 画出 L2CAP 连接建立的状态机图

### Day 14：RFCOMM

- `stack/rfcomm/` — `rfc_mx_fsm.c`（多路复用状态机）
- 理解 RFCOMM 如何在 L2CAP 之上模拟串口

---

## 第三周：Profile 层 + BLE

**目标：理解一个完整 Profile 的实现，掌握 BLE GATT**

### Day 15-17：A2DP 音频流（推荐起点）

自底向上追踪数据流：

```mermaid
graph LR
    A[stack/a2dp] --> B[bta/av]
    B --> C[btif/btif_av.c]
    C --> D[audio_a2dp_hw.c]
    D --> E[Unix Socket / HAL]
```

- SBC 编码：`embdrv/sbc/`
- IPC 传输：`udrv/ulinux/uipc.c`

> [!question] 追踪目标
> 手机播放音乐时，PCM 数据如何一步步变成蓝牙包？

### Day 18-19：SMP 配对与安全

- `stack/smp/` — BLE 配对协议实现
- `btif/src/btif_dm.c` — 设备管理，`pairing_cb` 结构体
- 理解 Legacy Pairing vs ==LE Secure Connections== 的区别

### Day 20-21：GATT / BLE

- `stack/gatt/` — ATT 协议实现
- `bta/gatt/` — GATT 客户端/服务端

> [!question] 追踪目标
> BLE 设备扫描 → 连接 → 服务发现 → 读写 Characteristic 的完整流程

---

## 第四周：系统机制 + 综合

**目标：理解 OS 抽象、内存管理、并发模型，形成完整认知**

### Day 22-23：GKI vs OSI

- `gki/common/` — 老式 buffer pool、timer、task 模型
- `osi/src/` — 现代 reactor + thread 模型

> [!info] 历史背景
> GKI 是 Broadcom 遗留设计，OSI 是 Google 后来重写的现代替代。两套并存是历史债务。

### Day 24-25：并发与消息传递

- 理解 `fixed_queue` + `reactor` + `thread` 的协作模式
- 追踪一个 HCI 事件如何通过消息队列逐层传递到 BTIF

### Day 26-27：安全与边界审查

- 查阅 3-5 个历史 CVE（如 BlueBorne 系列），对照代码找漏洞位置
- 重点审查 `stack/smp/`、`stack/bnep/` 的输入校验逻辑

> [!warning] 注意
> 此阶段以防御性学习为目的，理解漏洞成因有助于写出更健壮的代码

### Day 28-30：综合复盘

- [ ] 写一篇技术文档：选择最感兴趣的 Profile，从 HAL 到 HCI 完整描述数据流
- [ ] 尝试修改一个小功能并编译验证（如修改 BTSnoop 格式、调整 A2DP buffer 大小）

---

## 关键学习方法

| 方法 | 说明 |
|------|------|
| **追数据流** | 每个协议都追一条完整路径，不要孤立读函数 |
| **画状态机** | BTM、L2CAP、SMP 都是状态机驱动，画图是关键 |
| **配合抓包** | BTSnoop + Wireshark 对照代码，理解效率提升数倍 |
| **写注释** | 读懂的代码立即写中文注释，防止遗忘 |
| **对照规范** | 遇到协议细节查 [Bluetooth Core Spec](https://www.bluetooth.com/specifications/specs/) |

---

> [!success] 推荐起点
> 从 **第三周 Day 15 的 A2DP 流程**开始，先建立信心，再往底层走。
> A2DP 音频链路是整个栈中最直观、最容易验证的路径。
