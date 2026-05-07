---
title: HMI 代码生成器：基于 UI 资源 JSON 的 MVP 代码自动生成工具
tags:
  - HMI
  - 工具
  - 代码生成
  - MVP架构
  - Python
created: 2026-05-07
status: active
---

# HMI 代码生成器：基于 UI 资源 JSON 的 MVP 代码自动生成工具

> [!info] 阅读背景
> 本文介绍一种面向车机 HMI 项目的代码生成工具。该工具从 UI 资源 JSON 文件中自动提取页面结构与控件信息，按照项目既有的 MVP + Handler 架构规范，批量生成 View、Presenter、Handler 层的 C++ 骨架代码，消除新模块开发中的重复劳动。

---

## 一、背景与动机

### 1.1 HMI 项目架构概述

车机 HMI 项目采用了一种**改进的 MVP 架构**，在传统的 Model-View-Presenter 之上引入了独立的 Handler 层，形成四层分离结构。这种分层的核心依据并非教科书式的设计模式，而是**实际工程约束**——各层的代码形态由不同的外部因素决定：

| 层级 | 约束来源 | 代码方式 | 可自动化程度 |
|:---:|:---:|:---:|:---:|
| **View** | UI 资源文件（`.fty` / JSON） | 模板化 | ★★★★★ |
| **Presenter** | UI 资源中的页面与控件结构 | 模板化 | ★★★★☆ |
| **Handler** | 业务需求 | 手写 | ★★☆☆☆（仅骨架） |
| **Mode** | 中间件 Proxy 接口 | 受限于中间件定义 | ★☆☆☆☆ |

![[HMI四层架构约束来源.png]]

> [!tip] Handler 层的定位
> Handler 层是整个架构中**唯一真正自由的手写层**，夹在两个外部约束之间：上层 View/Presenter 的接口形状由 UI 资源决定，下层 Mode 的接口形状由中间件 Proxy 决定。Handler 实际上承担了**防腐层（Anti-Corruption Layer）** 的角色，隔离了 UI 变更和中间件变更对彼此的影响。

### 1.2 问题陈述

在实际开发过程中，每次新增一个功能模块（如 Wifi、蓝牙、空调等），开发人员都需要手动完成以下重复工作：

- 为每个页面创建 View 类：声明控件 ID 成员变量、编写 `GetIDByStringName()` 初始化代码、注册按钮点击回调、编写回调转发方法
- 创建 Presenter 类：声明事件转发接口、管理多个 Handler、编写 `bindView()` / `bindMode()` 绑定逻辑
- 创建 Handler 骨架：声明 `initialize()` 方法、定义 `req*` 请求方法

这些代码结构高度一致，差异仅在于控件名称和数量。以 Wifi 模块为例，3 个页面共需手写 16 个文件，其中超过 80% 的代码属于可推导的样板代码。

### 1.3 解决思路

UI 资源仓库（`linux-qcm6125-ui-resource`）中的 JSON 文件已经完整描述了每个页面的控件结构，包括控件名称、类型（按钮、文本、图片、列表等）以及交互属性（是否需要通知回调）。这些信息恰好是生成 View / Presenter / Handler 代码所需的全部输入。

因此，本工具的核心思路是：

```
UI 资源 JSON → 解析提取 → 数据模型 → 模板渲染 → C++ 源文件
```

---

## 二、UI 资源 JSON 结构分析

### 2.1 JSON 文件来源

UI 资源文件位于 `linux-qcm6125-ui-resource` 仓库中，按分辨率和项目型号组织：

```
linux-qcm6125-ui-resource/
├── json/
│   ├── json_720x1440_FQC0601_557/      # 557 项目（720×1440）
│   │   ├── Setting/
│   │   │   ├── [0276]IDP_WIFI_LIST.json
│   │   │   ├── [0278]IDP_WIFI_PASSWORD.json
│   │   │   └── [0279]IDP_WIFI_INFOR.json
│   │   └── ...
│   ├── json_1920x720_VQC1006_332BEV/   # 332BEV 项目（1920×720）
│   └── json_1920x720_VQC1007_965Tonale/
└── output/                              # 编译产物（.fty + *_all_map.h）
```

JSON 文件经过工具链编译后，生成 `.fty` 二进制资源和 `*_all_map.h` 头文件，后者包含所有控件的字符串 ID 到数字 ID 的映射表。运行时，HMI 代码通过 `GetIDByStringName("IDB_WIFI_LIST_SHOW_MORE")` 查表获取数字 ID。

### 2.2 JSON 顶层结构

每个 JSON 文件描述一个页面，顶层包含两个数组：

```json
{
    "page": [{
        "name": "IDP_WIFI_LIST",
        "controls": [ ... ]
    }],
    "idres": [
        { "name": "IDP_WIFI_LIST",          "id": 18087936 },
        { "name": "IDB_WIFI_LIST_SHOW_MORE", "id": 18087940 },
        ...
    ]
}
```

- `page[0].controls`：控件列表，每个控件包含名称和类型配置
- `idres`：ID 映射表（与编译产物 `*_all_map.h` 对应）

### 2.3 控件类型识别

控件类型通过 `ctlconfig` 对象中**存在哪个配置 key** 来判断：

```json
{
    "name": "IDB_WIFI_LIST_SHOW_MORE",
    "ctlconfig": {
        "buttoncfg": { "notify": true, ... }
    }
}
```

| `ctlconfig` 中的 key | 控件类型 | 名称前缀 | 是否可交互 |
|:---:|:---:|:---:|:---:|
| `buttoncfg` | 按钮 | `IDB_` | 由 `notify` 字段决定 |
| `textcfg` | 文本 | `IDT_` | 否 |
| `imagecfg` | 图片 | `IDI_` / `IDC_` | 由 `notify` 字段决定 |
| `listboxcfg` | 列表 | `IDL_` | 由 `notify` 字段决定 |
| `widgetcfg` | 组件 | `IDW_` | 否 |

> [!note] notify 字段
> `notify=true` 表示该控件需要注册点击回调。这是代码生成器区分"是否生成 `RegisterShortClick` + 回调方法"的关键依据。

---

## 三、工具整体架构设计

### 3.1 设计目标

- **输入灵活**：支持交互式 CLI，可传入单个 JSON 文件、多个文件或整个目录
- **输出可控**：默认输出到 `./generated/` 预览，支持 `--inplace` 直接写入 HMI 主仓库
- **生成完整**：覆盖 View（`.h/.cpp`）+ Presenter（`.h/.cpp`）+ Handler 骨架（`.h/.cpp`）
- **零外部依赖**：仅使用 Python 标准库，不引入 Jinja2 等第三方模板引擎

### 3.2 整体流水线

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   JSON 文件      │     │    PageModel    │     │  C++ 源文件      │
│  (UI 资源)       │────▶│   (数据模型)     │────▶│  (View/P/H)     │
└─────────────────┘     └─────────────────┘     └─────────────────┘
      Parser                 Model                 Generator
        │                      │                      │
   json_parser.py         page_model.py     view_generator.py
                                            presenter_generator.py
                                            handler_generator.py
```

### 3.3 模块结构

```
tools/codegen/
├── main.py                     # CLI 入口，交互式参数收集
├── parser/
│   └── json_parser.py          # JSON 解析 → PageModel
├── model/
│   └── page_model.py           # 数据模型定义
├── generator/
│   ├── view_generator.py       # View .h/.cpp 生成
│   ├── presenter_generator.py  # Presenter .h/.cpp 生成
│   └── handler_generator.py    # Handler 骨架生成
├── templates/
│   ├── view_h.tpl              # View 头文件模板
│   ├── view_cpp.tpl            # View 实现文件模板
│   ├── base_view_h.tpl         # 基类 View 接口模板
│   ├── presenter_h.tpl         # Presenter 头文件模板
│   ├── presenter_cpp.tpl       # Presenter 实现文件模板
│   ├── ipresenter_h.tpl        # IPresenter 接口模板
│   ├── handler_h.tpl           # Handler 头文件模板
│   └── handler_cpp.tpl         # Handler 实现文件模板
└── utils/
    └── naming.py               # ID 命名转换工具
```

---

## 四、各模块详细设计

### 4.1 Parser（解析层）

Parser 负责读取 JSON 文件，提取页面名称和控件列表，输出 `PageModel` 对象。

核心逻辑：

1. 读取 `page[0].name` 获取页面名称
2. 遍历 `page[0].controls[]`，检查 `ctlconfig` 中存在哪个配置 key 来判定控件类型
3. 提取 `notify` 字段判断是否可交互
4. 支持传入目录路径，自动扫描目录下所有 `.json` 文件

### 4.2 Model（数据模型）

```python
class ControlType(Enum):
    BUTTON = "button"
    TEXT = "text"
    IMAGE = "image"
    LISTBOX = "listbox"
    WIDGET = "widget"

@dataclass
class Control:
    name: str              # "IDB_WIFI_LIST_SHOW_MORE"
    control_type: ControlType
    notify: bool = False   # 是否需要注册点击回调

@dataclass
class PageModel:
    page_name: str         # "IDP_WIFI_LIST"
    controls: List[Control]
```

`PageModel` 提供了便捷的分类属性：

| 属性 | 返回内容 |
|---|---|
| `interactive_controls` | 所有 `notify=true` 的 Button 和 ListBox |
| `text_controls` | 所有 Text 控件 |
| `image_controls` | 所有 Image 控件 |
| `widget_controls` | 所有 Widget 控件 |

### 4.3 Naming（命名转换工具）

命名转换是代码生成的核心环节，负责将 UI 资源 ID 名称转换为符合项目规范的 C++ 标识符。

> [!example] 转换示例（module = "Wifi"）
>
> | 函数 | 输入 | 输出 |
> |---|---|---|
> | `id_to_member_var` | `"IDB_WIFI_LIST_SHOW_MORE"` | `m_showMoreBtnId` |
> | `id_to_member_var` | `"IDT_WIFI_LIST_TITLE"` | `m_titleTextId` |
> | `id_to_member_var` | `"IDC_IMG_01140002"` | `m_img01140002Id` |
> | `id_to_callback` | `"IDB_WIFI_LIST_SHOW_MORE"` | `onShowMoreBtnClick` |
> | `id_to_callback` | `"IDL_WIFI_LIST"` | `onListItemClick` |
> | `page_to_view_class` | `"IDP_WIFI_LIST"` | `CWifiListView` |
> | `page_to_handler_class` | `"IDP_WIFI_LIST"` | `CWifiListHandler` |

转换步骤：
1. 去掉类型前缀（`IDB_`、`IDT_` 等）
2. 去掉模块名（`WIFI_`）
3. 去掉页面名（`LIST_`）
4. 将剩余部分从 `UPPER_SNAKE` 转为 `camelCase`
5. 添加类型后缀（`Btn`、`Text`、`Img`、`List`）

> [!warning] 特殊处理
> 对于 `IDC_IMG_01140002` 这类无语义的自动生成 ID，保留原始 hex 后缀，生成 `m_img01140002Id`，避免产生无意义的变量名。

### 4.4 Generator（代码生成层）

Generator 根据控件类型生成不同的代码片段，然后填充到模板中。

#### 4.4.1 View 生成规则

| 控件类型 | 生成内容 |
|---|---|
| **Button**（`notify=true`） | 成员变量声明 + `GetIDByStringName` 初始化 + `RegisterShortClick` 注册 + 回调方法（转发到 Presenter） |
| **ListBox**（`notify=true`） | 同 Button，回调方法额外传递 `wparam`（列表项索引） |
| **Text** | 成员变量声明 + `GetIDByStringName` 初始化 + `setText` setter 方法 |
| **Image** | 成员变量声明 + `GetIDByStringName` 初始化 + `show()` / `hide()` 方法 |
| **Widget** | 成员变量声明 + `GetIDByStringName` 初始化 |

生成的 View 代码遵循现有项目规范：

```cpp
class CWifiListView : public CViewBase, public CWifiView {
public:
    CWifiListView(unsigned int pageid, IGUIControl *guicontrol);
    bool preInitialize() final;
    bool initialize(IViewRegisterNotification *notiregister) final;
    void bindPresenter(CWifiPresenter *presenter);

    // Text setters
    void setTitleText(const std::string &text);

    // Image show/hide
    void showBackground();
    void hideBackground();

private:
    CWifiPresenter *mPresenter{nullptr};
    unsigned int m_pageId{0};
    unsigned int m_showMoreBtnId{0};
    // ...

    // Click handlers
    void onShowMoreBtnClick(unsigned int wparam, void *lparam);
};
```

#### 4.4.2 Presenter 生成规则

Presenter 汇聚所有页面的交互事件：

- 收集所有页面中 `notify=true` 的控件，生成对应的事件转发方法
- **自动去重**：不同页面中同名按钮（如 `IDB_*_BACK`）只生成一个方法声明
- 为每个页面生成对应的 Handler 成员变量
- 生成 `initialize()` 方法，初始化所有 Handler 并绑定对应的 View 和 Mode

#### 4.4.3 Handler 生成规则

Handler 只生成骨架，方法体留空：

- 生成 `initialize(View*, Mode*, Presenter*)` 绑定方法
- 为每个可交互控件生成 `req*` 方法（如 `reqShowMore()`、`reqListItemSelect(unsigned int index)`）
- 方法体标记 `// TODO: implement`，等待开发者填写业务逻辑

### 4.5 Templates（模板层）

模板文件使用 `${PLACEHOLDER}` 占位符，由 Generator 在运行时替换。

以 `view_h.tpl` 为例：

```cpp
#pragma once

#include <framework/Gui/CViewBase.h>
#include "${BASE_VIEW_HEADER}"

class ${PRESENTER_CLASS};

class ${CLASS_NAME} : public CViewBase, public ${BASE_VIEW_CLASS} {
public:
    ${CLASS_NAME}(unsigned int pageid, IGUIControl *guicontrol);
    bool preInitialize() final;
    bool initialize(IViewRegisterNotification *notiregister) final;
    void bindPresenter(${PRESENTER_CLASS} *presenter);

${PUBLIC_METHODS}
private:
    ${PRESENTER_CLASS} *mPresenter{nullptr};
    unsigned int m_pageId{0};
${MEMBER_VARS}
${CALLBACK_DECLS}
};
```

> [!tip] 为什么选择自定义模板而非 Jinja2？
> 项目的模板需求简单（仅字符串替换），使用 Python 标准库的 `str.replace()` 即可满足。避免引入外部依赖，使工具在任何安装了 Python3 的车机开发环境中都能直接运行。

---

## 五、生成产物

以 Wifi 模块为例，输入 3 个 JSON 文件（List、Password、Infor），生成 16 个文件：

```
Wifi/
├── View/ViewUIA03/
│   ├── CWifiView.h              ← 基类接口（空）
│   ├── CWifiListView.h/.cpp     ← 列表页 View
│   ├── CWifiPasswordView.h/.cpp ← 密码页 View
│   └── CWifiInforView.h/.cpp    ← 信息页 View
├── Presenter/Presenter_Common/
│   ├── IWifiPresenter.h         ← Presenter 纯虚接口
│   ├── CWifiPresenter.h/.cpp    ← Presenter 实现
└── Handler/
    ├── CWifiListHandler.h/.cpp     ← 列表 Handler 骨架
    ├── CWifiPasswordHandler.h/.cpp ← 密码 Handler 骨架
    └── CWifiInforHandler.h/.cpp    ← 信息 Handler 骨架
```

---

## 六、使用方式

### 6.1 基本用法

```bash
cd /home/linux-qcm6125-ui-resource/tools/codegen

# 交互式运行，输出到 ./generated/ 目录预览
python3 main.py

# 直接写入 HMI 主仓库对应目录
python3 main.py --inplace
```

### 6.2 交互流程

```
[HMI Code Generator]
========================================
Module name (e.g. Wifi): Wifi
JSON file paths (one per line, empty line to finish):
  > /home/linux-qcm6125-ui-resource/json/json_720x1440_FQC0601_557/Setting
  >
Output directory [./generated/Wifi]:
  >

Parsing JSON files...
  ✓ IDP_WIFI_LIST: 9 controls (4 images, 2 texts, 2 buttons, 1 listboxs)
  ✓ IDP_WIFI_PASSWORD: 7 controls (1 images, 2 texts, 3 buttons, 1 widgets)
  ✓ IDP_WIFI_INFOR: 10 controls (3 images, 5 texts, 2 buttons)

Generating code...
  ✓ View/ViewUIA03/CWifiView.h
  ✓ View/ViewUIA03/CWifiListView.h
  ✓ View/ViewUIA03/CWifiListView.cpp
  ...
  ✓ Handler/CWifiInforHandler.cpp

Done! 16 files generated in ./generated/Wifi/
Use --inplace to write directly to /home/linux-qcm6125-hmi/Source/Apps/Wifi/
```

> [!note] 输入支持
> - 支持传入**文件路径**或**目录路径**（自动扫描目录下所有 `.json` 文件）
> - 支持 `~` 自动展开为 home 目录

---

## 七、关键设计决策

### 7.1 为什么 Handler 只生成骨架？

Handler 是**唯一纯手写的业务层**。它的职责是在 View 层（UI 资源约束）和 Mode 层（中间件约束）之间做差异适配和业务编排。生成器提供了方法签名和绑定关系作为起点，但具体的业务逻辑必须由开发者根据需求实现。

### 7.2 Presenter 重复回调的去重

不同页面可能包含同名按钮（如多个页面都有 `IDB_*_BACK` 返回按钮），如果不去重会导致 Presenter 中出现重复的方法声明。生成器在收集所有页面的交互事件时，使用 `set` 对回调方法名去重，确保每个方法只声明一次。

### 7.3 模板与逻辑分离

将 C++ 代码结构放在 `.tpl` 模板文件中，生成逻辑放在 Python Generator 中。这样当项目编码规范发生变化时（如修改缩进风格、调整 include 顺序等），只需修改模板文件，无需改动生成逻辑。

---

## 八、后续扩展方向

| 方向 | 说明 |
|---|---|
| 支持更多控件类型 | 如 `slidercfg`（滑块）、`editorcfg`（编辑框）等 |
| 增量生成 | 检测已有代码，只补充新增控件，不覆盖手写修改 |
| Mode 层骨架 | 根据中间件 Proxy 头文件，生成 Mode 层的监听接口实现 |
| 多项目适配 | 支持不同分辨率/项目的 JSON 目录自动选择 |

---

## 九、相关资源

| 资源 | 路径 |
|---|---|
| 代码生成器 | `/home/linux-qcm6125-ui-resource/tools/codegen/` |
| UI 资源 JSON | `/home/linux-qcm6125-ui-resource/json/` |
| HMI 主仓库 | `/home/linux-qcm6125-hmi/` |
| 参考模块（Wifi） | `/home/linux-qcm6125-hmi/Source/Apps/Wifi/` |
| ID 映射实现 | `/home/linux-qcm6125-hmi/Source/LocalShare/IDConvert.h/.cpp` |
