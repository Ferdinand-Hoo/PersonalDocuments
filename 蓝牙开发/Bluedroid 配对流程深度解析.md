---
title: Bluedroid 配对流程深度解析：BR/EDR SSP 与 BLE SMP
tags:
  - bluedroid
  - bluetooth
  - 协议栈
  - 安全
  - 源码分析
  - 配对
created: 2026-05-07
status: active
---

# Bluedroid 配对流程深度解析：BR/EDR SSP 与 BLE SMP

> [!info] 阅读背景
> 本文基于 Android AOSP Bluedroid 源码，结合 Bluetooth Core Spec 5.x，深入分析经典蓝牙（BR/EDR）**安全简单配对（SSP）** 与低功耗蓝牙（BLE）**安全管理协议（SMP）** 两条配对路径的完整实现，包括模块层次、状态机、关键函数调用链及安全考量。

---

## 一、配对机制概述

蓝牙配对的本质是让两台设备在没有预共享密钥的前提下，通过公开信道安全地协商出一个共同的密钥，并据此建立加密链路。

Bluedroid 实现了两套独立的配对协议：

| 配对类型 | 适用制式 | 核心协议 | 承载信道 | 密钥产物 |
|---------|---------|---------|---------|---------|
| **SSP（安全简单配对）** | BR/EDR | ECDH + 数字确认 | HCI 事件驱动 | Link Key（128 bit）|
| **SMP（安全管理协议）** | BLE | AES-128 + Confirm | L2CAP 固定信道 CID=0x0006 | LTK / IRK / CSRK |

> [!tip] 配对 vs 绑定
> **配对（Pairing）** 指一次性协商密钥并建立加密；**绑定（Bonding）** 指将密钥持久化保存，供后续重连使用。配对不一定导致绑定（见 `BOND_TYPE_TEMPORARY`），但绑定必然经历配对。详见 [[Bluedroid绑定机制深度解析]]。

---

## 二、模块层次与代码分布

### 2.1 整体架构

```
应用层 / Android Framework
    ↓  createBond() / JNI
btif/src/btif_dm.c          ← BTIF DM：HAL 桥接层，维护 pairing_cb
    ↓  BTA_DmBond()
bta/dm/bta_dm_api.c         ← BTA DM：应用适配层
    ↓  btm_sec_bond_by_transport()
stack/btm/btm_sec.c         ← BTM SEC：BR/EDR 安全核心，配对状态机
stack/btm/btm_ble.c         ← BTM BLE：BLE 安全管理，Key 存储
    ↓  SMP_Pair()（BLE 路径）
stack/smp/smp_main.c        ← SMP：BLE 配对状态机
stack/smp/smp_act.c         ← SMP 动作函数
stack/smp/smp_keys.c        ← 密钥生成工具
    ↓
stack/l2cap/                ← L2CAP：SMP 固定信道承载
HCI / Controller
```

### 2.2 关键文件职责速查

| 文件路径 | 职责 |
|---------|------|
| `btif/src/btif_dm.c` | 维护 `pairing_cb`，处理 BTA 上报的配对事件，决定 `bond_type` |
| `bta/dm/bta_dm_api.c` | 提供 `BTA_DmBond()`，根据 transport 分发到 BTM |
| `stack/btm/btm_sec.c` | BR/EDR 配对状态机、IO Capability 处理、认证完成回调 |
| `stack/btm/btm_ble.c` | BLE 配对入口、加密发起、Key 分发存储 |
| `stack/smp/smp_main.c` | SMP FSM 主表（Master/Slave 分别一张状态转移表） |
| `stack/smp/smp_act.c` | SMP 每个状态的动作函数，含关联模型选择 |
| `stack/smp/smp_api.c` | SMP 对外接口：`SMP_Pair()`、`SMP_PasskeyReply()` 等 |
| `stack/smp/smp_keys.c` | Confirm 值、STK、LTK 的 AES-128 计算 |

---

## 三、BR/EDR 安全简单配对（SSP）

### 3.1 SSP 关联模型

SSP 根据双方 IO Capability 和 OOB 标志，选择以下关联模型：

| 本地 IO Cap | 对端 IO Cap | 关联模型 | MITM 保护 |
|------------|------------|---------|:--------:|
| DisplayOnly | DisplayOnly | Just Works | ✗ |
| DisplayOnly | KeyboardOnly | Passkey Entry（显示方） | ✓ |
| KeyboardOnly | DisplayOnly | Passkey Entry（输入方） | ✓ |
| DisplayYesNo | DisplayYesNo | Numeric Comparison | ✓ |
| NoInputNoOutput | 任意 | Just Works | ✗ |

IO Capability 常量定义：

```c
// stack/btm/btm_api.h
#define BTM_IO_CAP_OUT      0   // DisplayOnly
#define BTM_IO_CAP_IO       1   // DisplayYesNo
#define BTM_IO_CAP_IN       2   // KeyboardOnly
#define BTM_IO_CAP_NONE     3   // NoInputNoOutput
#define BTM_IO_CAP_KBDISP   4   // KeyboardDisplay
```

### 3.2 BTM 配对状态机

```c
// stack/btm/btm_int.h:725
enum {
    BTM_PAIR_STATE_IDLE,                 // 空闲
    BTM_PAIR_STATE_GET_REM_NAME,         // 查询远端名称（检查 SM4）
    BTM_PAIR_STATE_WAIT_PIN_REQ,         // 等待 PIN 请求（Legacy Pairing）
    BTM_PAIR_STATE_WAIT_LOCAL_PIN,       // 等待本地 PIN 输入
    BTM_PAIR_STATE_WAIT_NUMERIC_CONFIRM, // SSP 数字确认
    BTM_PAIR_STATE_KEY_ENTRY,            // Passkey 输入
    BTM_PAIR_STATE_WAIT_LOCAL_OOB_RSP,  // OOB 响应
    BTM_PAIR_STATE_WAIT_LOCAL_IOCAPS,   // IO Capability 交换
    BTM_PAIR_STATE_INCOMING_SSP,         // 被动接受 SSP
    BTM_PAIR_STATE_WAIT_AUTH_COMPLETE,   // 等待认证完成
    BTM_PAIR_STATE_WAIT_DISCONNECT       // 等待断开（专有绑定）
};
```

> [!example]- BR/EDR SSP 状态转移图（点击展开）
> ```mermaid
> stateDiagram-v2
>     [*] --> IDLE
>     IDLE --> GET_REM_NAME : BTM_SecBond() 发起
>     GET_REM_NAME --> WAIT_LOCAL_IOCAPS : 名称获取完成（SSP）
>     GET_REM_NAME --> WAIT_PIN_REQ : Legacy Pairing 路径
>     WAIT_LOCAL_IOCAPS --> INCOMING_SSP : IO Caps 交换完成
>     INCOMING_SSP --> WAIT_NUMERIC_CONFIRM : Numeric Comparison
>     INCOMING_SSP --> KEY_ENTRY : Passkey Entry
>     WAIT_NUMERIC_CONFIRM --> WAIT_AUTH_COMPLETE : 用户确认
>     KEY_ENTRY --> WAIT_AUTH_COMPLETE : Passkey 输入完成
>     WAIT_LOCAL_PIN --> WAIT_AUTH_COMPLETE : PIN 输入完成
>     WAIT_AUTH_COMPLETE --> IDLE : 认证成功
>     WAIT_AUTH_COMPLETE --> WAIT_DISCONNECT : 认证失败（DD 绑定）
>     WAIT_DISCONNECT --> IDLE : 断开完成
> ```

### 3.3 配对标志位

```c
// stack/btm/btm_int.h
#define BTM_PAIR_FLAGS_WE_STARTED_DD    0x01  // 我方发起专有绑定
#define BTM_PAIR_FLAGS_PEER_STARTED_DD  0x02  // 对端发起专有绑定
#define BTM_PAIR_FLAGS_DISC_WHEN_DONE   0x04  // 完成后立即断连
#define BTM_PAIR_FLAGS_PIN_REQD         0x08  // 已发出 PIN 请求
#define BTM_PAIR_FLAGS_LE_ACTIVE        0x20  // SMP 正在进行
#define BTM_PAIR_FLAGS_WE_CANCEL_DD     0x40  // 正在取消绑定
```

### 3.4 SSP 完整调用链

> [!example]- BR/EDR SSP 时序图（点击展开）
> ```mermaid
> sequenceDiagram
>     participant App as 应用层
>     participant BTIF as btif_dm.c
>     participant BTM as btm_sec.c
>     participant Peer as 对端设备
>
>     App->>BTIF: createBond()
>     BTIF->>BTM: btm_sec_bond_by_transport(BR_EDR)
>     Note over BTM: pairing_state = GET_REM_NAME
>     BTM->>Peer: HCI Remote Name Request
>     Peer-->>BTM: Remote Name Complete
>
>     BTM->>Peer: HCI IO Capability Request Reply
>     Note over BTM: pairing_state = WAIT_LOCAL_IOCAPS
>     Peer-->>BTM: IO Capability Response
>     Note over BTM: 确定关联模型
>
>     alt Numeric Comparison
>         BTM-->>BTIF: BTA_DM_SP_CFM_REQ_EVT（显示数字）
>         BTIF-->>App: pairing_request(PASSKEY_CONFIRMATION)
>         App->>BTIF: 用户确认
>         BTIF->>BTM: btm_sec_bond_by_transport confirm
>     else Passkey Entry
>         BTM-->>BTIF: BTA_DM_SP_KEY_NOTIF_EVT
>         BTIF-->>App: ssp_request(PASSKEY_ENTRY)
>         App->>BTIF: passkey_reply()
>     end
>
>     Peer-->>BTM: HCI Authentication Complete
>     Note over BTM: Link Key 写入 tBTM_SEC_DEV_REC.link_key
>     BTM-->>BTIF: BTA_DM_AUTH_CMPL_EVT
>     BTIF-->>App: bond_state_changed(BONDED)
> ```

**逐步说明：**

1. **`btm_sec_bond_by_transport()`** — `stack/btm/btm_sec.c`
   判断 transport：BR/EDR 进入状态机，BLE 调用 `SMP_Pair()`。

2. **`btm_io_capabilities_req()`** — `stack/btm/btm_sec.c`
   收到 HCI `IO_CAPABILITY_REQUEST` 事件，状态进入 `WAIT_LOCAL_IOCAPS`，回调 BTIF 请求本地 IO 能力。

3. **`btif_dm_proc_io_req()`** — `btif/src/btif_dm.c`
   根据 `pairing_cb.is_local_initiated` 决定 `auth_req`，注入绑定位（DD_BOND 或 GEN_BOND）。

4. **`btm_io_capabilities_rsp()`** — `stack/btm/btm_sec.c`
   收到对端 IO Capability，确定 SSP 方式；若对端设置 `DD_BOND` 位则置 `BTM_PAIR_FLAGS_PEER_STARTED_DD`。

5. **`btm_proc_sp_req_evt()`** — `stack/btm/btm_sec.c`
   状态进入 `WAIT_NUMERIC_CONFIRM`，向上层抛出 `BTA_DM_SP_CFM_REQ_EVT` 等待用户确认。

6. **`btm_sec_auth_complete()`** — `stack/btm/btm_sec.c`
   HCI Authentication Complete 触发，Link Key 写入设备记录，通知上层。

7. **`btm_sec_encrypt_change()`** — `stack/btm/btm_sec.c`
   加密状态变更，配对流程结束，发出 `BTM_AUTH_CMPL_EVT`。

---

## 四、BLE 安全管理协议（SMP）

### 4.1 SMP 关联模型

`smp_decide_asso_model()` — `stack/smp/smp_act.c:31`

| 本地 IO 能力 | 对端 IO 能力 | OOB | 选择模型 | MITM 保护 |
|------------|------------|-----|---------|:--------:|
| DisplayOnly | DisplayOnly | 无 | Just Works（`SMP_MODEL_ENC_ONLY`） | ✗ |
| DisplayOnly | KeyboardOnly | 无 | Passkey Entry（`SMP_MODEL_PASSKEY`） | ✓ |
| KeyboardOnly | DisplayOnly | 无 | Key Notification（`SMP_MODEL_KEY_NOTIF`） | ✓ |
| KeyboardOnly | KeyboardOnly | 无 | Passkey Entry（`SMP_MODEL_PASSKEY`） | ✓ |
| DisplayYesNo | DisplayYesNo | 无 | Numeric Comparison | ✓ |
| NoInputNoOutput | 任意 | 无 | Just Works（`SMP_MODEL_ENC_ONLY`） | ✗ |
| 任意 | 任意 | 有 | OOB（`SMP_MODEL_OOB`） | ✓ |

### 4.2 SMP 状态机

```c
// stack/smp/smp_int.h:92
enum {
    SMP_ST_IDLE,              // 空闲
    SMP_ST_WAIT_APP_RSP,      // 等待应用层响应（IO Caps / TK）
    SMP_ST_SEC_REQ_PENDING,   // Security Request 已发出，等待
    SMP_ST_PAIR_REQ_RSP,      // Pairing Request/Response 交换中
    SMP_ST_WAIT_CONFIRM,      // 等待 Confirm 值
    SMP_ST_CONFIRM,           // 发送 Confirm 值
    SMP_ST_RAND,              // Random 值交换
    SMP_ST_ENC_PENDING,       // 等待链路加密完成
    SMP_ST_BOND_PENDING,      // 等待 Key 分发完成
    SMP_ST_RELEASE_DELAY,     // 配对完成延迟释放
    SMP_ST_MAX
};
```

> [!example]- SMP 状态转移图（点击展开）
> ```mermaid
> stateDiagram-v2
>     [*] --> ST_IDLE
>     ST_IDLE --> ST_WAIT_APP_RSP : L2CAP 连接建立 / 收到 Pairing Req
>     ST_WAIT_APP_RSP --> ST_PAIR_REQ_RSP : App 提供 IO Caps
>     ST_PAIR_REQ_RSP --> ST_WAIT_CONFIRM : Pairing Req/Rsp 交换完成
>     ST_WAIT_CONFIRM --> ST_CONFIRM : 收到对端 Confirm 值
>     ST_CONFIRM --> ST_RAND : 发送本端 Confirm 值
>     ST_RAND --> ST_ENC_PENDING : Random 交换完成，生成 STK
>     ST_ENC_PENDING --> ST_BOND_PENDING : 链路加密建立（STK）
>     ST_BOND_PENDING --> ST_RELEASE_DELAY : Key 分发完成
>     ST_RELEASE_DELAY --> ST_IDLE : 配对完成
> ```

### 4.3 关键数据结构

SMP 控制块 `tSMP_CB` 为全局单例，意味着 **BLE 同一时刻只能有一对设备处于配对流程中**：

```c
// stack/smp/smp_int.h:157
typedef struct {
    tSMP_CALLBACK   *p_callback;      // 上层回调（BTM 注册）
    TIMER_LIST_ENT   rsp_timer_ent;   // 响应超时定时器
    BD_ADDR          pairing_bda;     // 当前配对目标地址
    tSMP_STATE       state;           // 当前 FSM 状态
    tSMP_IO_CAP      peer_io_caps;    // 对端 IO 能力
    tSMP_IO_CAP      loc_io_caps;     // 本地 IO 能力
    tSMP_OOB_FLAG    peer_oob_flag;   // 对端 OOB 标志
    tSMP_AUTH_REQ    peer_auth_req;   // 对端 AuthReq
    BT_OCTET16       tk;              // Temporary Key（由关联模型决定）
    BT_OCTET16       ltk;             // Long Term Key
    BT_OCTET16       csrk;            // Connection Signature Resolving Key
    UINT16           div;             // Diversifier（EDIV 的生成基础）
    UINT8            role;            // HCI_ROLE_MASTER / SLAVE
    UINT8            flags;           // 配对标志位
} tSMP_CB;

extern tSMP_CB smp_cb;               // 全局单例
```

> [!warning] 并发限制
> `tSMP_CB` 是全局单例，BLE 同时只能进行一对设备的配对。并发场景（如多个 BLE 设备同时发起连接）需要在 BTM 层进行串行化处理，否则会导致状态机错乱。

### 4.4 BLE 配对完整调用链

> [!example]- BLE SMP 时序图（点击展开）
> ```mermaid
> sequenceDiagram
>     participant App as 应用层
>     participant BTIF as btif_dm.c
>     participant BTM as btm_sec.c / btm_ble.c
>     participant SMP as smp_api.c / smp_main.c
>     participant L2C as L2CAP
>     participant HCI as HCI/Controller
>
>     App->>BTIF: createBond()
>     BTIF->>BTM: btm_sec_bond_by_transport(BT_TRANSPORT_LE)
>     BTM->>SMP: SMP_Pair(bd_addr)
>     SMP->>L2C: L2CA_ConnectFixedChnl(0x0006, bd_addr)
>     L2C-->>SMP: SMP_L2CAP_CONN_EVT
>     SMP->>SMP: smp_sm_event → 发送 Pairing Request PDU
>
>     SMP-->>BTM: tSMP_CALLBACK(SMP_IO_CAP_REQ_EVT)
>     BTM-->>BTIF: IO Caps 请求
>     BTIF-->>App: pairing_request 回调
>     App->>BTIF: 用户响应（Just Works / Passkey 等）
>     BTIF->>BTM: btm_ble_io_capabilities_req_reply()
>     BTM->>SMP: SMP_IO_RSP_EVT
>
>     SMP->>SMP: smp_decide_asso_model() → 选择关联模型
>     SMP->>SMP: smp_send_confirm() → Confirm 值交换
>     SMP->>SMP: smp_send_init() → Random 值交换
>     Note over SMP: 双方验证 Confirm，生成 STK
>
>     SMP->>HCI: btm_ble_start_encrypt(STK)
>     HCI-->>SMP: SMP_ENCRYPTED_EVT
>     Note over SMP,HCI: 链路加密建立（使用 STK）
>
>     SMP->>SMP: Key 分发（LTK / IRK / CSRK）
>     SMP-->>BTM: SMP_AUTH_CMPL_EVT
>     BTM-->>BTIF: BTA_DM_AUTH_CMPL_EVT
>     BTIF-->>App: bond_state_changed(BONDED)
> ```

**逐步说明：**

1. **`SMP_Pair(bd_addr)`** — `stack/smp/smp_api.c`
   置位 `SMP_PAIR_FLAGS_WE_STARTED_DD`，调用 `L2CA_ConnectFixedChnl(L2CAP_SMP_CID)` 建立 SMP 信道。

2. **`smp_sm_event(SMP_L2CAP_CONN_EVT)`** — `stack/smp/smp_main.c`
   L2CAP 固定信道就绪后触发，向对端发送 Pairing Request PDU，状态进入 `ST_PAIR_REQ_RSP`。

3. **`smp_decide_asso_model()`** — `stack/smp/smp_act.c:31`
   根据 `loc_io_caps`、`peer_io_caps`、`peer_oob_flag` 三要素决定关联模型，并设置 TK 来源。

4. **`smp_send_confirm()`** — `stack/smp/smp_act.c`
   调用 `smp_gen_p2()` + AES-128 计算 Confirm 值（`CConfirm = f4(PKax, PKbx, Na, 0)`）。

5. **`smp_send_init()`** — `stack/smp/smp_act.c`
   发送本端 Random 值，双方交换后验证 Confirm，生成 STK（`STK = s1(TK, SRand, MRand)`）。

6. **`btm_ble_start_encrypt(STK)`** — `stack/btm/btm_ble.c`
   向 HCI 下发 `HCI_BLE_Start_Encryption` 命令，使用 STK 建立加密链路。

7. **Key 分发阶段** — `smp_send_enc_info()` / `smp_send_id_info()` / `smp_send_csrk_info()`
   加密链路建立后在加密保护下分发长期密钥（见第五章）。

8. **`smp_proc_pairing_cmpl()`** — `stack/smp/smp_act.c`
   触发上层回调 `SMP_AUTH_CMPL_EVT`，状态机经 `ST_RELEASE_DELAY` 回到 `ST_IDLE`。

---

## 五、BLE Key 分发机制

BLE 配对完成后，双方在加密信道内互相分发长期密钥，用于后续重连直接恢复加密（无需重新配对）。

### 5.1 分发的密钥类型

| SMP Opcode | 密钥 | 用途 |
|-----------|------|------|
| `SMP_OPCODE_ENCRYPT_INFO` (0x06) | **LTK**（Long Term Key） | 后续重连加密 |
| `SMP_OPCODE_MASTER_ID` (0x07) | **EDIV + Rand** | LTK 的检索索引 |
| `SMP_OPCODE_IDENTITY_INFO` (0x08) | **IRK**（Identity Resolving Key） | 解析随机地址 |
| `SMP_OPCODE_ID_ADDR` (0x09) | **Identity Address** | 设备真实地址 |
| `SMP_OPCODE_SIGN_INFO` (0x0A) | **CSRK**（Connection Signature Key） | 签名写操作 |

### 5.2 Key 分发完整时序

> [!example]- Key 分发时序图（点击展开）
> ```mermaid
> sequenceDiagram
>     participant M as Master（发起方）
>     participant S as Slave（响应方）
>
>     Note over M,S: 链路已用 STK 加密
>
>     S->>M: Encryption Information (LTK)
>     S->>M: Master Identification (EDIV + Rand)
>     S->>M: Identity Information (IRK)
>     S->>M: Identity Address Information
>     S->>M: Signing Information (CSRK)
>
>     M->>S: Encryption Information (LTK)
>     M->>S: Master Identification (EDIV + Rand)
>     M->>S: Identity Information (IRK)
>     M->>S: Identity Address Information
>     M->>S: Signing Information (CSRK)
>
>     Note over M,S: 绑定完成，后续重连使用 LTK 直接加密
> ```

> [!note] 重连加密流程
> 后续重连时，Master 发送 `HCI_BLE_Start_Encryption(EDIV, Rand)`，双方各自用 IRK 解出 LTK，直接建立加密链路，**无需重新执行 SMP 配对流程**。

---

## 六、回调链与层间通信

### 6.1 上行回调链（配对完成通知）

> [!example]- 回调层次图（点击展开）
> ```mermaid
> graph LR
>     SMP["SMP Layer\nsmp_proc_pairing_cmpl()"]
>     BTM["BTM Layer\nbtm_ble_smp_cback()"]
>     BTA["BTA Layer\nbta_dm_ble_smp_cback()"]
>     BTIF["BTIF Layer\nbtif_dm_upstreams_evt()"]
>     APP["Application\nbond_state_changed()"]
>
>     SMP -->|"SMP_AUTH_CMPL_EVT\n(tSMP_CALLBACK)"| BTM
>     BTM -->|"BTA_DM_AUTH_CMPL_EVT"| BTA
>     BTA -->|"HAL callback"| BTIF
>     BTIF -->|"JNI / binder"| APP
> ```

### 6.2 关键事件对照

| BTA 事件 | 触发时机 | BTIF 处理函数 |
|---------|---------|-------------|
| `BTA_DM_PIN_REQ_EVT` | 需要输入 PIN 码 | `btif_dm_pin_req_evt()` |
| `BTA_DM_SP_CFM_REQ_EVT` | SSP 数字确认请求 | `btif_dm_ssp_cfm_req_evt()` |
| `BTA_DM_SP_KEY_NOTIF_EVT` | Passkey 显示通知 | `btif_dm_ssp_key_notif_evt()` |
| `BTA_DM_AUTH_CMPL_EVT` | 认证/配对完成 | `btif_dm_auth_cmpl_evt()` |
| `BTA_DM_BLE_KEY_EVT` | BLE Key 分发通知 | `btif_dm_ble_key_evt()` |
| `BTA_DM_BLE_SEC_REQ_EVT` | BLE Security Request | `btif_dm_ble_sec_req_evt()` |

---

## 七、安全考量

### 7.1 Just Works 的 MITM 风险

Just Works 模型的 TK 固定为全零（`0x00...00`），任何中间人都可以预测 Confirm 值并伪造配对。Bluedroid 的处理策略：

- **不提供 MITM 保护**，只保证窃听防护
- 在 `btif_dm_ssp_cfm_req_evt()` 中，Just Works + 双方无绑定位 → `BOND_TYPE_TEMPORARY`
- 临时配对不持久化密钥，最大限度降低安全风险

### 7.2 SMP 定时器残留风险

```c
// stack/smp/smp_int.h
typedef struct {
    ...
    TIMER_LIST_ENT   rsp_timer_ent;   // ← 配对失败后必须停止此定时器
    ...
} tSMP_CB;
```

若 `smp_cleanup_pairing_ble_dev()` 未正确调用 `btu_stop_timer(&smp_cb.rsp_timer_ent)`，残留定时器会在下次配对中提前触发超时，导致配对莫名失败。

### 7.3 配对碰撞（Pairing Collision）

当主从双端同时发起配对时：

```c
// btm_pair_flags 同时被置位
BTM_PAIR_FLAGS_WE_STARTED_DD    // 我方发起
BTM_PAIR_FLAGS_PEER_STARTED_DD  // 对端发起
```

按 BT Core Spec，**Master 应放弃并等待 Slave 的流程完成**。若代码未正确处理 collision，两端均进入等待状态导致死锁。

### 7.4 LTK 失效处理

重连时加密失败（`HCI_ERR_KEY_MISSING`）的正确处理：

```c
// stack/btm/btm_ble.c
case HCI_ERR_KEY_MISSING:
    // 正确做法：清除本端旧绑定，重新发起 SMP 配对
    BTM_DeleteStoredLinkKey(&bd_addr, NULL);
    SMP_Pair(bd_addr);
    break;
    // 错误做法：静默失败，上层无感知，连接永远无法加密
```

---

## 八、常见配对异常场景

| 场景 | 现象 | 可能原因 | 严重性 |
|------|------|---------|:-----:|
| 配对超时 | `btm_sec_pairing_timeout()` 触发 | 用户未及时输入 PIN/Passkey | 低 |
| Confirm 值不匹配 | `SMP_CONFIRM_VALUE_ERR` | Passkey 输入错误 / TK 不一致 | 中 |
| Authentication Failed | `HCI_ERR_AUTH_FAILURE` | Link Key 不匹配 / 认证被拒绝 | 中 |
| LTK 失效 | 重连加密失败 | 一端清除了绑定信息 | 中 |
| 配对碰撞 | 流程卡死，双端等待 | 主从同时发起配对 | 高 |
| SMP 定时器误触发 | 配对莫名超时 | 上次配对的定时器未清除 | 高 |
| OOB 数据缺失 | `SMP_OOB_FAIL` | 配置了 OOB 但未提供数据 | 高 |

---

## 九、排查方法

### 9.1 关键日志 Tag

```
BTM_SEC    BLE_SMP    BTA_DM    BTIF_DM
```

### 9.2 关键日志关键字速查

| 关键字 | 含义 |
|-------|------|
| `btm_sec_bond_by_transport` | 配对入口，确认 transport 类型 |
| `SMP_Pair` | BLE 配对发起 |
| `smp_sm_event state=` | SMP 状态转移，跟踪流程推进 |
| `smp_decide_asso_model` | 关联模型选择结果 |
| `btm_ble_start_encrypt` | 加密请求下发 |
| `SMP_ENCRYPTED_EVT` | 链路加密建立 |
| `SMP_AUTH_CMPL_EVT` | SMP 配对完成 |
| `btm_sec_auth_complete` | BR/EDR 认证完成 |
| `Pairing failed reason=` | 失败原因码，对照 SMP error code |

### 9.3 BLE 配对失败六步定位法

```
Step 1  SMP_L2CAP_CONN_EVT 是否触发？     → 否：L2CAP 固定信道建立失败
Step 2  Pairing REQ/RSP 是否完成？        → 否：IO Caps 协商问题
Step 3  Confirm/Random 是否成功交换？     → 否：TK 不一致（Passkey 输入错误）
Step 4  btm_ble_start_encrypt 是否下发？  → 否：STK 生成失败
Step 5  SMP_ENCRYPTED_EVT 是否触发？      → 否：Controller 加密失败
Step 6  Key 分发 PDU 是否完整？           → 否：上层 Key 存储问题
```

### 9.4 关键变量检查

| 变量 | 含义 | 所在结构 |
|------|------|---------|
| `smp_cb.state` | 当前 SMP 状态机状态 | `tSMP_CB` |
| `smp_cb.loc_io_caps` | 本地 IO 能力 | `tSMP_CB` |
| `smp_cb.peer_io_caps` | 对端 IO 能力 | `tSMP_CB` |
| `smp_cb.flags` | 配对标志位 | `tSMP_CB` |
| `btm_cb.pairing_state` | BTM 配对状态 | `tBTM_CB` |
| `btm_cb.pairing_flags` | BTM 配对标志位 | `tBTM_CB` |
| `pairing_cb.bond_type` | BTIF 绑定类型 | `btif_dm_pairing_cb_t` |

---

## 十、相关源码索引

| 功能点 | 文件 | 关键符号 |
|-------|------|---------|
| 配对入口 API | `bta/dm/bta_dm_api.c` | `BTA_DmBond()`, `BTA_DmBondByTransport()` |
| BTIF 配对状态 | `btif/src/btif_dm.c` | `pairing_cb`, `btif_dm_create_bond()` |
| BTM 配对状态机 | `stack/btm/btm_sec.c` | `btm_sec_bond_by_transport()`, `btm_sec_change_pairing_state()` |
| BTM 配对状态枚举 | `stack/btm/btm_int.h:725` | `BTM_PAIR_STATE_*` |
| BTM 配对标志位 | `stack/btm/btm_int.h:742` | `BTM_PAIR_FLAGS_*` |
| IO Capability 处理 | `stack/btm/btm_sec.c` | `btm_io_capabilities_req()`, `btm_io_capabilities_rsp()` |
| SSP 认证完成 | `stack/btm/btm_sec.c` | `btm_sec_auth_complete()`, `btm_sec_encrypt_change()` |
| BLE 安全管理 | `stack/btm/btm_ble.c` | `btm_ble_start_encrypt()`, `btm_ble_smp_cback()` |
| SMP 对外 API | `stack/smp/smp_api.c` | `SMP_Pair()`, `SMP_PasskeyReply()`, `SMP_Register()` |
| SMP 状态机主表 | `stack/smp/smp_main.c` | `smp_sm_event()`, `smp_master_entry_map[]` |
| SMP 动作函数 | `stack/smp/smp_act.c` | `smp_decide_asso_model()`, `smp_send_confirm()`, `smp_proc_pairing_cmpl()` |
| SMP 状态枚举 | `stack/smp/smp_int.h:92` | `SMP_ST_*` |
| SMP 控制块 | `stack/smp/smp_int.h:157` | `tSMP_CB`, `smp_cb` |
| SMP Key 生成 | `stack/smp/smp_keys.c` | STK/LTK 的 AES-128 实现 |
| SMP L2CAP 承载 | `stack/smp/smp_l2c.c` | `smp_l2cap_if_init()` |

---

> [!quote] 延伸阅读
> - [[Bluedroid绑定机制深度解析]] — 专有绑定与通用绑定的深度对比
> - [[Bluedroid一个月学习计划]] — 整体学习路径规划
> - Bluetooth Core Spec Vol 3, Part H — Security Manager Protocol（SMP）
> - Bluetooth Core Spec Vol 2, Part C, Section 4.2 — SSP Association Models
