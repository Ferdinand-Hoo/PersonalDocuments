---
title: Bluedroid 绑定机制深度解析：通用绑定与专有绑定
tags:
  - bluedroid
  - bluetooth
  - 协议栈
  - 安全
  - 源码分析
created: 2026-05-06
status: active
---

# Bluedroid 绑定机制深度解析：通用绑定与专有绑定

> [!info] 阅读背景
> 本文基于 Android AOSP Bluedroid 源码，结合 Bluetooth Core Spec 5.x，深入分析经典蓝牙（BR/EDR）中**专有绑定（Dedicated Bonding）**与**通用绑定（General Bonding）**两种机制的实现原理、代码路径与关键差异。

---

## 一、绑定机制概述

在蓝牙经典协议中，**绑定（Bonding）**的本质是在两台设备之间交换并持久保存**链路密钥（Link Key）**，使得后续连接无需重复配对。

Bluetooth Core Spec 定义了三种配对模式：

| 配对模式 | 说明 | 链路密钥是否持久化 |
|---------|------|:---:|
| **No Bonding** | 仅做临时认证，不保存密钥 | ✗ |
| **Dedicated Bonding** | 专门建立连接以完成绑定，绑定后立即断开 | ✓ |
| **General Bonding** | 在正常业务连接的过程中顺带完成绑定 | ✓ |

这两种绑定模式的核心区别在于：**连接的目的不同，因此绑定完成后的行为也不同。**

---

## 二、协议层的表示：auth_req 字段

绑定模式在 HCI IO Capability 交换阶段，通过 `auth_req` 字段对外声明。

### 2.1 auth_req 位定义

```c
// stack/include/btm_api.h
#define BTM_AUTH_SP_NO      0   // No MITM, No Bonding
#define BTM_AUTH_SP_YES     1   // MITM,    No Bonding
#define BTM_AUTH_AP_NO      2   // No MITM, Dedicated Bonding  ← DD_BOND bit
#define BTM_AUTH_AP_YES     3   // MITM,    Dedicated Bonding
#define BTM_AUTH_SPGB_NO    4   // No MITM, General Bonding    ← GB_BIT
#define BTM_AUTH_SPGB_YES   5   // MITM,    General Bonding

#define BTM_AUTH_DD_BOND    2   // 专有绑定位（bit1）
#define BTM_AUTH_GB_BIT     4   // 通用绑定位（bit2）
#define BTM_AUTH_BONDS      6   // 绑定掩码（bit1 | bit2）
#define BTM_AUTH_YN_BIT     1   // MITM 位（bit0）
```

用一张位图来理解：

```
bit2  bit1  bit0
 GB   DD   MITM
  0    0    0   → No Bonding, No MITM  (BTM_AUTH_SP_NO  = 0)
  0    0    1   → No Bonding, MITM     (BTM_AUTH_SP_YES = 1)
  0    1    0   → Dedicated, No MITM   (BTM_AUTH_AP_NO  = 2)
  0    1    1   → Dedicated, MITM      (BTM_AUTH_AP_YES = 3)
  1    0    0   → General,   No MITM   (BTM_AUTH_SPGB_NO  = 4)
  1    0    1   → General,   MITM      (BTM_AUTH_SPGB_YES = 5)
```

> [!tip] 判断是否携带绑定意图
> `auth_req & BTM_AUTH_BONDS` 不为零，表示本次配对携带绑定意图（Dedicated 或 General）。
> 这个掩码在代码中大量使用，用于判断是否需要持久化链路密钥。

### 2.2 默认 DD auth_req

```c
// include/bt_target.h
#ifndef BTM_DEFAULT_DD_AUTH_REQ
#define BTM_DEFAULT_DD_AUTH_REQ   BTM_AUTH_AP_YES  // 专有绑定 + MITM 保护
#endif
```

Bluedroid 对专有绑定的默认要求是**强制 MITM 保护**，防止中间人攻击。

---

## 三、BTIF 层：pairing_cb 与绑定类型

BTIF 层是 Android HAL 接口层，它维护一个全局的配对上下文结构体 `pairing_cb`，在整个配对过程中记录状态。

### 3.1 pairing_cb 结构体

```c
// btif/src/btif_dm.c
#define BOND_TYPE_UNKNOWN    0  // 初始状态（memset 后的默认值）
#define BOND_TYPE_PERSISTENT 1  // 持久绑定，链路密钥需要保存
#define BOND_TYPE_TEMPORARY  2  // 临时配对，链路密钥不保存

typedef struct {
    bt_bond_state_t state;
    BD_ADDR         bd_addr;
    UINT8           bond_type;        // ← 核心字段
    UINT8           pin_code_len;
    UINT8           is_ssp;
    UINT8           auth_req;
    UINT8           io_cap;
    UINT8           is_local_initiated;
    ...
} btif_dm_pairing_cb_t;

static btif_dm_pairing_cb_t pairing_cb;
```

> [!warning] 一个重要的安全修复
> 历史上曾有一个 CVE（CVE-2014-7914），原因是 `pairing_cb` 的旧版本用 `is_temp` 标志，`memset` 为零后默认为 `is_temp=FALSE`（即默认持久化）。这导致 Just Works 临时配对的链路密钥被错误地持久化保存。
>
> 修复方案（commit `0cd5a85`）将语义反转：将字段重命名为 `bond_type`，`memset` 后的零值 = `BOND_TYPE_UNKNOWN`，在逻辑上等同于**临时配对**，确保安全兜底。

### 3.2 绑定类型的判定：`btif_dm_ssp_cfm_req_evt()`

在 SSP（Secure Simple Pairing）IO Capability 确认阶段，系统判断此次配对是否需要持久化：

```c
// btif/src/btif_dm.c
// if just_works and bonding bit is not set → TEMPORARY
if (p_ssp_cfm_req->just_works
    && !(p_ssp_cfm_req->loc_auth_req & BTM_AUTH_BONDS)
    && !(p_ssp_cfm_req->rmt_auth_req & BTM_AUTH_BONDS)
    && !(check_cod(..., COD_HID_POINTING)))
    pairing_cb.bond_type = BOND_TYPE_TEMPORARY;
else
    pairing_cb.bond_type = BOND_TYPE_PERSISTENT;
```

**临时配对的条件（三者同时满足）：**
1. 使用 Just Works 方式（无数字确认）
2. 本地 `auth_req` 未设置绑定位
3. 对端 `auth_req` 未设置绑定位
4. 设备不是 HID 指点设备（鼠标等外设即使 Just Works 也做持久化）

### 3.3 auth_req 的设置：`btif_dm_proc_io_req()`

在 IO Capability 交换时，BTIF 层根据发起方决定使用何种绑定模式：

```c
// btif/src/btif_dm.c
void btif_dm_proc_io_req(..., BOOLEAN is_orig)
{
    if (pairing_cb.is_local_initiated)
    {
        // 本地主动发起 → 专有绑定 + MITM
        *p_auth_req = BTA_AUTH_DD_BOND | BTA_AUTH_SP_YES;  // = 3
    }
    else if (!is_orig)
    {
        // 对端发起 → 跟随对端的绑定需求
        *p_auth_req = (pairing_cb.auth_req & BTA_AUTH_BONDS);
        if (yes_no_bit || (pairing_cb.io_cap & BTM_IO_CAP_IO))
            *p_auth_req |= BTA_AUTH_SP_YES;
    }
    else if (yes_no_bit)
    {
        // 已存储设备重连 → 通用绑定
        *p_auth_req = BTA_AUTH_GEN_BOND | yes_no_bit;  // = 5
    }
}
```

**三种场景的 auth_req 决策：**

| 场景 | auth_req 值 | 绑定类型 |
|------|------------|---------|
| 本地主动发起配对 | `DD_BOND \| MITM` = 3 | 专有绑定 |
| 对端发起，跟随对端 | 复制对端 bonding bits | 取决于对端 |
| 已存储设备（MITM需求）| `GEN_BOND \| MITM` = 5 | 通用绑定 |

---

## 四、专有绑定（Dedicated Bonding）完整流程

专有绑定的核心特征：**建立 ACL 连接的唯一目的是完成绑定，绑定完成后立即断开连接。**

### 4.1 发起阶段

```c
// stack/btm/btm_sec.c → BTM_SecBond()
memcpy(btm_cb.pairing_bda, bd_addr, BD_ADDR_LEN);
btm_cb.pairing_flags = BTM_PAIR_FLAGS_WE_STARTED_DD;  // 0x01，标记我方发起 DD
```

随后调用 `btm_sec_dd_create_conn()`，设置关键标志并创建 ACL 连接：

```c
// btm_sec_dd_create_conn()
btm_cb.pairing_flags |= BTM_PAIR_FLAGS_DISC_WHEN_DONE;  // 0x04，完成后断开
l2cu_create_conn(p_lcb, BT_TRANSPORT_BR_EDR);
btm_sec_change_pairing_state(BTM_PAIR_STATE_WAIT_PIN_REQ);
```

> [!note] BTM_PAIR_FLAGS 标志位含义
> ```c
> #define BTM_PAIR_FLAGS_WE_STARTED_DD    0x01  // 我方发起专有绑定
> #define BTM_PAIR_FLAGS_PEER_STARTED_DD  0x02  // 对端发起专有绑定
> #define BTM_PAIR_FLAGS_DISC_WHEN_DONE   0x04  // 完成后立即断连
> #define BTM_PAIR_FLAGS_PIN_REQD         0x08  // 已请求 PIN
> #define BTM_PAIR_FLAGS_WE_CANCEL_DD     0x40  // 正在取消绑定流程
> ```

### 4.2 IO Capability 阶段：注入 DD_BOND 位

在 IO Capability 交换时，BTM 层强制将 `DD_BOND` 位注入到我方的 `auth_req`：

```c
// btm_sec.c → btm_io_capabilities_req() 和 BTM_IoCapRsp()
if (btm_cb.pairing_flags & BTM_PAIR_FLAGS_WE_STARTED_DD)
{
    auth_req = BTM_AUTH_DD_BOND | (auth_req & BTM_AUTH_YN_BIT);
    // 保留 MITM 位，替换绑定类型为 Dedicated
}
```

### 4.3 配对完成：触发断连

认证完成后，`btm_sec_connected()` 检测到是专有绑定，通知链路密钥，并触发 L2CAP 层启动断连定时器：

```c
// btm_sec.c → btm_sec_connected()
if (btm_cb.pairing_flags & BTM_PAIR_FLAGS_WE_STARTED_DD)
    res = TRUE;

// 通知上层链路密钥已生成
(*btm_cb.api.p_auth_complete_callback)(..., HCI_SUCCESS);
btm_sec_change_pairing_state(BTM_PAIR_STATE_IDLE);

if (res)
    l2cu_update_lcb_4_bonding(p_dev_rec->bd_addr, TRUE);  // 标记此连接为 bonding 连接
```

L2CAP 层随即触发断连：

```c
// l2c_utils.c → l2cu_start_post_bond_timer()
if (p_lcb->idle_timeout == 0)
{
    // 立即发送 HCI Disconnect
    btsnd_hcic_disconnect(p_lcb->handle, HCI_ERR_PEER_USER);
    p_lcb->link_state = LST_DISCONNECTING;
}
```

`l2c_link.c` 也在 ACL 连接建立时检测绑定连接并提前返回，不走正常的 L2CAP 信道建立流程：

```c
// l2c_link.c → l2c_link_hci_conn_complete()
if (p_lcb->is_bonding)
{
    if (l2cu_start_post_bond_timer(handle))
        return TRUE;  // 专有绑定，不建立 L2CAP 信道
}
```

### 4.4 对端检测专有绑定

当对端在 IO Capability Response 中设置了 `DD_BOND` 位时：

```c
// btm_sec.c → btm_io_capabilities_rsp()
if (btm_cb.pairing_state == BTM_PAIR_STATE_INCOMING_SSP
    && (evt_data.auth_req & BTM_AUTH_DD_BOND))
{
    btm_cb.pairing_flags |= BTM_PAIR_FLAGS_PEER_STARTED_DD;
}
```

### 4.5 专有绑定完整时序图

> [!example]- 时序图（点击展开）
> ```mermaid
> sequenceDiagram
>     participant App as App
>     participant Stack as BTM/BTIF
>     participant R as 对端
> 
>     App->>Stack: BTM_SecBond()
>     Note over Stack: pairing_flags =<br>WE_STARTED_DD | DISC_WHEN_DONE
>     Stack->>R: HCI Connect Request
>     R-->>Stack: HCI Connect Complete
>     Note over Stack: is_bonding=TRUE<br>不建立 L2CAP 信道
> 
>     Stack->>R: IO Cap Req (DD_BOND|MITM)
>     R-->>Stack: IO Cap Rsp
>     Stack->>R: SSP Confirm
>     R-->>Stack: Auth Complete
> 
>     Note over Stack: 通知链路密钥<br>保存至存储
>     Stack->>R: HCI Disconnect
>     Stack->>App: bond_state_changed(BONDED)
> ```

---

## 五、通用绑定（General Bonding）完整流程

通用绑定的核心特征：**绑定发生在正常的业务连接过程中，绑定完成后连接继续保持，双方可以立即建立服务信道。**

### 5.1 典型场景

- 首次连接蓝牙耳机（A2DP）
- 首次连接蓝牙键盘（HID）
- 已配对设备重连并需要重新认证

### 5.2 auth_req 设置

对于通用绑定，`btif_dm_proc_io_req()` 会设置 `GEN_BOND` 位：

```c
// 已存储设备重连，设置通用绑定位
*p_auth_req = BTA_AUTH_GEN_BOND | yes_no_bit;  // = 4 | 1 = 5
```

`BTM_AUTH_GB_BIT = 4`（bit2），区别于专有绑定的 `BTM_AUTH_DD_BOND = 2`（bit1）。

### 5.3 连接保持

通用绑定过程中 **不设置** `BTM_PAIR_FLAGS_WE_STARTED_DD`，因此：
- `btm_sec_connected()` 中 `res = FALSE`，不调用 `l2cu_update_lcb_4_bonding(TRUE)`
- L2CAP 不启动 post-bond 断连定时器
- ACL 连接在认证后继续保持
- 上层可以立即建立 L2CAP 服务信道（RFCOMM、AVDT 等）

### 5.4 通用绑定完整时序图

> [!example]- 时序图（点击展开）
> ```mermaid
> sequenceDiagram
>     participant App as App
>     participant Stack as BTM/BTIF
>     participant R as 对端
> 
>     App->>Stack: 发起 A2DP / HFP 连接
>     Stack->>R: HCI Connect Request
>     R-->>Stack: HCI Connect Complete
> 
>     Stack->>R: IO Cap Req (GEN_BOND|MITM)
>     R-->>Stack: IO Cap Rsp
>     Stack->>R: SSP Confirm
>     R-->>Stack: Auth Complete
> 
>     Note over Stack: 保存链路密钥
>     Stack->>App: bond_state_changed(BONDED)
>     Note over Stack,R: ACL 保持，继续建立服务信道
>     Stack->>R: L2CAP Connect Req (A2DP/HFP)
> ```

---

## 六、临时配对（Temporary Pairing）：No Bonding 场景

除了两种绑定模式，还有一种情况需要特别了解——**临时配对**，即配对完成后不保存链路密钥。

### 6.1 触发条件

在 `btif_dm_ssp_cfm_req_evt()` 中，当 Just Works + 双方均未设置 bonding bit 时：

```c
pairing_cb.bond_type = BOND_TYPE_TEMPORARY;
```

### 6.2 对应的三个关键影响

**① 不上报 BONDING 状态**

```c
// bond_state_changed()
if (pairing_cb.bond_type == BOND_TYPE_TEMPORARY)
    state = BT_BOND_STATE_NONE;  // 应用层永远看不到这次"绑定"
```

**② 不保存链路密钥**

```c
// btif_dm_auth_cmpl_evt()
if (pairing_cb.bond_type == BOND_TYPE_TEMPORARY)
{
    // 不调用 btif_storage_add_bonded_device()
    bond_state_changed(BT_STATUS_SUCCESS, &bd_addr, BT_BOND_STATE_NONE);
    return;
}
```

**③ 链路密钥仅在本次连接中有效**

连接断开后，双方均无持久化的密钥，下次连接需要重新配对。

---

## 七、两种绑定模式对比总结

> [!example]- 决策流程图（点击展开）
> ```mermaid
> graph TD
>     A[发起配对] --> B{本地主动\nBTM_SecBond?}
> 
>     B -->|是| C[专有绑定]
>     B -->|否| D{auth_req\n含绑定位?}
>     D -->|是| E[通用绑定]
>     D -->|否| F{Just Works\n双方无绑定位?}
>     F -->|是| G[临时配对]
>     F -->|否| E
> 
>     C --> C1(绑定后断开 ACL)
>     C --> C2(密钥持久化)
>     C --> C3(DD_BOND=2)
> 
>     E --> E1(ACL 保持)
>     E --> E2(密钥持久化)
>     E --> E3(GB_BIT=4)
> 
>     G --> G1(ACL 保持)
>     G --> G2(密钥不保存)
>     G --> G3(应用层不感知)
> ```

| 特性 | 专有绑定 | 通用绑定 | 临时配对 |
|------|:-------:|:-------:|:-------:|
| `auth_req` 中的绑定位 | `DD_BOND`（bit1=1） | `GB_BIT`（bit2=1） | 均为 0 |
| 连接目的 | 仅为绑定 | 业务连接顺带绑定 | 无绑定意图 |
| 绑定后 ACL 状态 | **立即断开** | **保持连接** | 保持连接 |
| 链路密钥持久化 | ✓ | ✓ | ✗ |
| 上层感知 bond 状态 | ✓ | ✓ | ✗ |
| 典型使用场景 | 设置页面手动配对 | 首次连接耳机/HFP | 蓝牙键盘 Just Works |
| BTM 标志位 | `WE_STARTED_DD=0x01` | 无 | `bond_type=TEMPORARY` |
| BTIF bond_type | `PERSISTENT` | `PERSISTENT` | `TEMPORARY` |

---

## 八、从源码看安全考量

### 8.1 MITM 保护策略

- 专有绑定默认强制 MITM（`BTM_DEFAULT_DD_AUTH_REQ = BTM_AUTH_AP_YES`）
- 通用绑定的 MITM 要求取决于应用层配置
- Just Works 模式不提供 MITM 保护，因此强制为临时配对（不持久化）

### 8.2 历史漏洞：CVE-2014-7914

该漏洞的根源就是 `pairing_cb` 的默认值语义问题：

- **漏洞**：`is_temp=0` 被误解为"持久化"，导致 Just Works 的链路密钥被存储
- **修复**：反转语义，`bond_type=0` = `UNKNOWN` = 按临时处理，保存必须显式标记为 `PERSISTENT`
- **原则**：安全相关的默认值应选择**最保守的行为**

### 8.3 HID 指点设备的特殊处理

鼠标等 HID 设备即使走 Just Works，也会被强制设为 `PERSISTENT`：

```c
// 蓝牙鼠标 Just Works 也要持久化，否则每次使用都要重新配对
&& !(check_cod(..., COD_HID_POINTING))
```

这是一个典型的**用户体验优先于安全纯粹性**的工程权衡。

---

## 九、相关源码索引

| 功能点 | 文件 | 关键符号 |
|--------|------|---------|
| auth_req 常量定义 | `stack/include/btm_api.h` | `BTM_AUTH_*`, `BTM_AUTH_BONDS` |
| pairing_flags 定义 | `stack/btm/btm_int.h` | `BTM_PAIR_FLAGS_*` |
| 默认 DD auth_req | `include/bt_target.h` | `BTM_DEFAULT_DD_AUTH_REQ` |
| BTIF 绑定类型定义 | `btif/src/btif_dm.c` | `BOND_TYPE_*`, `pairing_cb` |
| 绑定类型判定 | `btif/src/btif_dm.c` | `btif_dm_ssp_cfm_req_evt()` |
| auth_req 设置逻辑 | `btif/src/btif_dm.c` | `btif_dm_proc_io_req()` |
| 链路密钥存储决策 | `btif/src/btif_dm.c` | `btif_dm_auth_cmpl_evt()` |
| DD 连接发起 | `stack/btm/btm_sec.c` | `BTM_SecBond()`, `btm_sec_dd_create_conn()` |
| IO Cap 注入 DD 位 | `stack/btm/btm_sec.c` | `btm_io_capabilities_req()`, `BTM_IoCapRsp()` |
| 认证后断连触发 | `stack/btm/btm_sec.c` | `btm_sec_connected()` |
| L2CAP 断连定时器 | `stack/l2cap/l2c_utils.c` | `l2cu_start_post_bond_timer()` |
| L2CAP bonding 标记 | `stack/l2cap/l2c_utils.c` | `l2cu_update_lcb_4_bonding()` |
| L2CAP 连接后处理 | `stack/l2cap/l2c_link.c` | `l2c_link_hci_conn_complete()` |

---

> [!quote] 延伸阅读
> - [[Bluedroid一个月学习计划]] — 整体学习路径规划
> - Bluetooth Core Spec Vol 3, Part C, Section 4.2 — Bonding Procedures
> - Bluetooth Core Spec Vol 2, Part E, Section 7.1.6 — IO Capability Reply
