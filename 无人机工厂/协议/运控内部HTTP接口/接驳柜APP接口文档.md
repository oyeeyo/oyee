# 无人接驳柜 (FengChao) 功能API文档

## 系统结构

无人接驳柜模块由两个主要层级组成：

```
MainComponentPager (主菜单)
  └─ FengChaoMainPager (无人接驳柜主界面)
      ├─ 功能1: 选择航站 + 查询柜机
      ├─ 功能2: 管理装货箱
      └─ 功能3: 订单处理
          ├─ Tab 0: 待揽件 (orderStatus=0)
          ├─ Tab 1: 待起飞 (orderStatus=1)
          ├─ Tab 2: 待派件 (orderStatus=3)
          ├─ Tab 3: 已完成 (orderStatus=4)
          └─ Tab 4: 异常 (orderStatus=5)
```

---

## 功能1: 航站选择 + 柜机查询

**入口**: FengChaoMainPager (fengchao_main_pager.dart)

### 1.1 获取航站列表

**接口**: `/uocs-tes/fcWaybill/selectTcTerminalList`
**请求方式**: POST
**调用位置**: fengchao_main_pager.dart:177

### 请求参数
```json
{
  "query": "string",         // 搜索关键词，可选
  "longitude": "string",     // GPS经度
  "latitude": "string",      // GPS纬度
  "userCode": "string"       // 用户ID
}
```

### 响应数据
```json
{
  "success": "boolean",
  "obj": [
    {
      "terminalId": "int",
      "terminalName": "string",
      "distance": "string"
    }
  ]
}
```

---

### 1.2 查询柜机列表（分页）

**接口**: `/uocs-tes/fcWaybill/selectCabinetList/{page}/{pageSize}`
**请求方式**: POST
**调用位置**: fengchao_main_pager.dart:248, 286

### 请求参数
```json
{
  "searchWord": "string",    // 搜索关键词（柜号），可选
  "refTerminalId": "int"     // 航站ID
}
```

### 响应数据
```json
{
  "success": "boolean",
  "errorMessage": "string",
  "obj": {
    "total": "int",
    "records": [
      {
        "id": "int",
        "cabinetCode": "string",       // 柜号
        "refTerminalId": "int",        // 航站ID
        "longitude": "string",         // 经度
        "latitude": "string",          // 纬度
        "distance": "string",          // 距离(km)
        "modelCode": "string",         // 型号
        "fengxiongName": "string",     // 柜机名称
        "isCanUse": "int"              // 是否可用 (0=可用, 1=不可用)
      }
    ]
  }
}
```

### 业务流程
```
用户进入无人接驳柜模块
  ↓
1. 获取用户GPS位置
2. 调用 /uocs-tes/fcWaybill/selectTcTerminalList 获取航站
3. 选择航站
  ↓
4. 搜索柜机名/柜号（防抖2秒）
5. 调用 /uocs-tes/fcWaybill/selectCabinetList 获取柜机列表（分页）
6. 按距离排序，展示可用柜机
```

---

## 功能2: 装货箱管理

### 2.1 查询装货箱列表

**接口**: `/uocs-tes/fcWaybill/selectLoadingBoxList/{page}/{pageSize}`
**请求方式**: POST
**调用位置**: fengchao_main_pager.dart:2567

### 请求参数
```json
{
  "terminalId": "int",       // 航站ID
  "searchWord": "string",    // 搜索关键词，可选
  "orderStatus": "int"       // 订单状态
}
```

### 响应数据
```json
{
  "success": "boolean",
  "obj": [
    {
      "id": "int",
      "loadingBoxCode": "string",   // 装货箱编号
      "boxState": "int"             // 箱子状态
    }
  ]
}
```

---

### 2.2 保存装货箱（创建）

**接口**: `/uocs-tes/fcWaybill/saveLoadingBox`
**请求方式**: POST
**调用位置**: fengchao_main_pager.dart:2522

### 请求参数
```json
{
  "loadingBoxCode": "string",   // 装货箱编号
  "takeoffTerminalId": "int",   // 起飞航站ID
  "operatorName": "string",     // 操作人名称
  "fcCabinetId": "int"          // 柜机ID（可选）
}
```

### 响应数据
```json
{
  "success": "boolean",
  "errorMessage": "string",
  "obj": null
}
```

---

### 2.3 更新装货箱

**接口**: `/uocs-tes/fcWaybill/updateLoadingBox`
**请求方式**: POST
**调用位置**: fengchao_main_pager.dart:2548

### 请求参数
```json
{
  "id": "int",
  "loadingBoxCode": "string",
  "operatorName": "string",
  "status": "int"
}
```

### 响应数据
```json
{
  "success": "boolean",
  "errorMessage": "string",
  "obj": null
}
```

---

## 功能3: 订单处理

**主界面**: FengChaoMainPager (fengchao_main_pager.dart)

### Tab标签页结构 (5个标签)

#### Tab 0: 待揽件 (orderStatus=0)

**含义**: 订单已创建，等待在柜机揽件

**可执行操作**:
1. 扫描或手动输入运单号
2. 确认揽件
3. 打印凭证

**相关API**:
```
POST /uocs-tes/fcWaybill/selectOrderList
  params: {
    "terminalId": "航站ID",
    "orderStatus": 0,
    "searchWord": "运单号(可选)"
  }

POST uocs-tes/fcWaybill/confirmPickup
  params: {
    "id": "订单ID",
    "operator": "操作员ID",
    "operatorName": "操作员名称"
  }
```

---

#### Tab 1: 待起飞 (orderStatus=1)

**含义**: 已揽件，等待无人机起飞

**可执行操作**:
1. 查看订单信息
2. 装载货物到柜机
3. 启动飞送任务

**相关API**:
```
POST /uocs-tes/fcWaybill/selectOrderList
  params: {
    "terminalId": "航站ID",
    "orderStatus": 1,
    "searchWord": "运单号(可选)"
  }

POST /uocs-tes/fcWaybill/saveLoadingBox
  params: {
    "loadingBoxCode": "装货箱编号",
    "takeoffTerminalId": "航站ID",
    "operatorName": "操作员名称"
  }
```

---

#### Tab 2: 待派件 (orderStatus=3)

**含义**: 飞送完成，等待派件

**可执行操作**:
1. 从柜机取出货物
2. 确认接收或派件
3. 更新订单状态

**相关API**:
```
POST /uocs-tes/fcWaybill/selectOrderList
  params: {
    "terminalId": "航站ID",
    "orderStatus": 3
  }

POST uocs-tes/fcWaybill/confirmReceive
  params: {
    "id": "订单ID",
    "operator": "操作员ID",
    "operatorName": "操作员名称"
  }

POST /uocs-tes/fcWaybill/updateOrderStatus
  params: {
    "id": "订单ID",
    "orderStatus": 4,
    "operator": "操作员ID"
  }
```

---

#### Tab 3: 已完成 (orderStatus=4)

**含义**: 派件完成

**可执行操作**:
- 查看完成的订单记录

**相关API**:
```
POST /uocs-tes/fcWaybill/selectOrderList
  params: {
    "terminalId": "航站ID",
    "orderStatus": 4
  }
```

---

#### Tab 4: 异常 (orderStatus=5)

**含义**: 异常订单

**可执行操作**:
- 查看异常原因
- 查看异常备注

**相关API**:
```
POST /uocs-tes/fcWaybill/selectOrderList
  params: {
    "terminalId": "航站ID",
    "orderStatus": 5
  }

POST /uocs-sys/dict/getDictListByBusinessCode
  params: {
    "dictTypeCode": "WaybillExceptionReason"
  }
```

---

## 完整的订单生命周期

### FengChao订单处理流程

```
1. 待揽件 (orderStatus=0)
   ├─ 查询 /uocs-tes/fcWaybill/selectOrderList (orderStatus=0)
   ├─ 获取订单列表
   ├─ 扫描或手动输入运单号
   └─ 调用 uocs-tes/fcWaybill/confirmPickup
        └─ 成功：进入待起飞状态

2. 待起飞 (orderStatus=1)
   ├─ 查询 /uocs-tes/fcWaybill/selectOrderList (orderStatus=1)
   ├─ 获取订单列表
   ├─ 创建或选择装货箱: /uocs-tes/fcWaybill/saveLoadingBox
   └─ 启动飞送任务
        └─ 飞送完成进入待派件状态

3. 待派件 (orderStatus=3)
   ├─ 查询 /uocs-tes/fcWaybill/selectOrderList (orderStatus=3)
   ├─ 获取订单列表
   ├─ 从柜机取出货物
   ├─ 确认接收: uocs-tes/fcWaybill/confirmReceive
   └─ 更新状态: /uocs-tes/fcWaybill/updateOrderStatus
        └─ 进入已完成状态

4. 已完成 (orderStatus=4)
   └─ 派件完成

5. 异常 (orderStatus=5)
   └─ 订单异常
```

---

## 核心数据模型

### FCOrderBean (接驳柜订单)
```dart
class FCOrderBean {
  int? id;
  String? fcOrderNo;              // 接驳柜订单号
  String? fcOrderCode;            // 接驳柜订单编码
  int? refTerminalId;             // 航站ID
  String? senderName;             // 寄件人
  String? senderTel;              // 寄件人电话
  String? senderAddr;             // 寄件地址
  String? receiverName;           // 收件人
  String? receiverTel;            // 收件人电话
  String? receiverAddr;           // 收件地址
  int? orderStatus;               // 订单状态 (0-5)
  String? createTm;               // 创建时间
  String? receiver;               // 揽件人ID
  String? receiverUserName;       // 揽件人名称
  String? receiveTm;              // 揽件时间
  String? sender;                 // 派件人ID
  String? senderUserName;         // 派件人名称
  String? sendTm;                 // 派件时间
}
```

### FCCabineBean (接驳柜)
```dart
class FCCabineBean {
  int? id;
  String? cabinetCode;            // 柜号
  int? refTerminalId;             // 航站ID
  String? longitude;              // 经度
  String? latitude;               // 纬度
  double? distance;               // 距离(km)
  String? modelCode;              // 型号代码
  String? fengxiongName;          // 柜机名称
  int? isCanUse;                  // 是否可用 (0=可用, 1=不可用)
}
```

### DeliverySameCityTerminalBean (航站信息)
```dart
class DeliverySameCityTerminalBean {
  int? terminalId;
  String? terminalName;           // 航站名称
  String? distance;               // 距离(km)
  String? longitude;              // 经度
  String? latitude;               // 纬度
}
```

---

## 订单状态说明

| 值 | 状态名 | 描述 | 所在Tab |
|----|--------|------|--------|
| 0 | 待揽件 | 订单已创建，等待在柜机揽件 | Tab 0 |
| 1 | 待起飞 | 已揽件，等待无人机起飞 | Tab 1 |
| 3 | 待派件 | 飞送完成，等待派件 | Tab 2 |
| 4 | 已完成 | 派件完成 | Tab 3 |
| 5 | 异常 | 异常订单 | Tab 4 |

---

## 柜机状态说明

| 值 | 状态 | 描述 |
|----|------|------|
| 0 | 可用 | 柜机正常，可以使用 |
| 1 | 不可用 | 柜机故障或维护中 |

---

## API调用汇总表

| 功能 | 接口 | 方式 | 参数关键字 | 位置 |
|------|------|------|-----------|------|
| 获取航站 | `/uocs-tes/fcWaybill/selectTcTerminalList` | POST | query | main_pager:177 |
| 查询柜机列表 | `/uocs-tes/fcWaybill/selectCabinetList` | POST | searchWord | main_pager:248 |
| 加载更多柜机 | `/uocs-tes/fcWaybill/selectCabinetList` | POST | searchWord | main_pager:286 |
| 查询订单列表 | `/uocs-tes/fcWaybill/selectOrderList` | POST | orderStatus | main_pager:745 |
| 加载更多订单 | `/uocs-tes/fcWaybill/selectOrderList` | POST | orderStatus | main_pager:802 |
| 获取装货箱 | `/uocs-tes/fcWaybill/selectLoadingBoxList` | POST | terminalId | main_pager:2567 |
| 确认揽件 | `uocs-tes/fcWaybill/confirmPickup` | POST | id | main_pager:5206 |
| 确认接收 | `uocs-tes/fcWaybill/confirmReceive` | POST | id | main_pager:2916 |
| 保存装货箱 | `/uocs-tes/fcWaybill/saveLoadingBox` | POST | loadingBoxCode | main_pager:2522 |
| 更新装货箱 | `/uocs-tes/fcWaybill/updateLoadingBox` | POST | id | main_pager:2548 |
| 更新订单状态 | `/uocs-tes/fcWaybill/updateOrderStatus` | POST | id | main_pager:2213 |
| 获取异常类型 | `/uocs-sys/dict/getDictListByBusinessCode` | POST | dictTypeCode | main_pager:1108 |

---

## 用户界面流程

### 主要操作流程

```
1️⃣ 选择航站 + 搜索柜机
   - 获取GPS位置
   - 调用 /uocs-tes/fcWaybill/selectTcTerminalList 获取航站
   - 选择航站
   - 搜索柜机名/柜号（防抖2秒）
   - 调用 /uocs-tes/fcWaybill/selectCabinetList 获取柜机列表
   - 按距离排序，展示在上方

2️⃣ 切换Tab进入订单管理
   - 自动进入"待揽件"Tab (orderStatus=0)
   - 显示该航站下的所有待揽件订单

3️⃣ 待揽件处理 (Tab 0)
   - 扫描或手动输入运单号
   - 搜索到订单后
   - 调用 uocs-tes/fcWaybill/confirmPickup 确认揽件
   - 滑动验证
   - 成功进入待起飞状态

4️⃣ 待起飞处理 (Tab 1)
   - 切换到"待起飞"Tab
   - 选择订单
   - 创建或选择装货箱
   - 调用 /uocs-tes/fcWaybill/saveLoadingBox
   - 启动飞送任务

5️⃣ 待派件处理 (Tab 2)
   - 切换到"待派件"Tab
   - 从柜机取出货物
   - 选择订单
   - 调用 uocs-tes/fcWaybill/confirmReceive 确认接收
   - 调用 /uocs-tes/fcWaybill/updateOrderStatus 更新状态
   - 滑动验证
   - 完成派件

6️⃣ 查看完成订单 (Tab 3 & Tab 4)
   - 切换到"已完成"Tab查看完成的订单
   - 切换到"异常"Tab查看异常订单
```

---

## 特殊功能

### 1. 柜机排序
- **时机**: 获取柜机列表后
- **规则**: 按距离从近到远排序（bubbleSort）
- **展示**: 最近的柜机显示在最前面

### 2. 搜索防抖
- **订单搜索**: 2秒延迟（searchList）
- **柜机搜索**: 2秒延迟（search）

### 3. 分页加载
- **柜机列表**: 每页20条，支持加载更多
- **订单列表**: 每页10条，支持加载更多

### 4. 滑动验证
- **作用**: 防止误操作
- **位置**: SlideVerifyWidget
- **应用**: 揽件、接收、派件等重要操作

---

## 与无人机飞送的集成

无人接驳柜与飞送系统的关系：

```
FengChao (无人接驳柜)
  ├─ 生成装货箱
  ├─ 绑定运单
  └─ 装载货物

  ↓ (通过loadingBox关联)

Landing (无人机飞送)
  ├─ 绑定装货箱到无人机
  ├─ 执行飞送任务
  └─ 到达目的地柜机投放货物
```

**关键关联**:
- `FCCabineBean` 对应 `CabinetBean` (飞送系统的柜机数据)
- 装货箱作为两个系统的交接点

---

## 重要说明

1. **进入前要求**: 必须先选择航站，否则无法进入订单处理

2. **GPS定位**: 获取航站列表时会自动获取用户GPS位置，用于显示距离

3. **权限要求**:
   - GPS定位权限
   - 摄像头权限（扫描二维码或拍照）

4. **用户信息**: 所有操作都从 SpUtil 获取:
   - userId (操作员ID)
   - userName (操作员名称)

5. **Tab刷新**: 切换Tab时自动刷新该Tab的数据

6. **订单搜索**: 支持在当前orderStatus下搜索运单号

7. **柜机排序**: 获取后自动按距离排序，最近的在最前