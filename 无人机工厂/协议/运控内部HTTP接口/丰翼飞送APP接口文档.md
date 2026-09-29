# 丰翼飞送 (FengYi Delivery) 功能API文档

## 系统结构

丰翼飞送系统由两个主要层级组成：

```
MainComponentPager (主菜单)
  └─ FengYiDelveryMainPager (丰翼飞送主界面)
      ├─ 功能1: 选择航站 (getTerminal)
      ├─ 功能2: 装货箱管理 (LoadBoxPager)
      └─ 功能3: 运单处理 (FengYiDeliveryPager)
          ├─ Tab 0: 待揽件 (orderStatus=0)
          ├─ Tab 1: 待派件 (orderStatus=1)
          ├─ Tab 2: 待起飞 (orderStatus=3)
          ├─ Tab 3: 已完成 (orderStatus=4)
          └─ Tab 4: 异常 (orderStatus=5)
```

---

## 功能1: 选择航站

**入口**: FengYiDelveryMainPager (fengyi_delivery_main_pager.dart)

### 获取航站列表

**接口**: `/uocs-tes/fyWaybill/selectTcTerminalList`
**请求方式**: POST
**调用位置**: fengyi_delivery_main_pager.dart:684

### 请求参数
```json
{
  "longitude": "string",     // GPS经度，可选
  "latitude": "string",      // GPS纬度，可选
  "query": "string"          // 搜索关键词，可选
}
```

### 响应数据
```json
{
  "success": "boolean",
  "errorMessage": "string",
  "obj": [
    {
      "terminalId": "int",
      "terminalName": "string",        // 航站名称
      "terminalCode": "string",        // 航站代码
      "terminalAddress": "string"      // 航站地址
    }
  ]
}
```

### 业务流程
```
用户点击航站选择 (AppBar中的航站名称)
  ↓
调用 getTerminal(true, false)
  ↓
用户输入搜索关键词（防抖500ms后查询）
  ↓
调用 /uocs-tes/fyWaybill/selectTcTerminalList
  ↓
展示航站列表
  ↓
用户选择航站，进入运单处理页面
```

---

## 功能2: 装货箱管理

**入口**: FengYiDelveryMainPager.goToBindBoxPager()

**跳转**: LoadBoxPager (load_box_pager.dart)

### 主要功能
- 扫描或手动输入运单号
- 获取对应的装货箱列表
- 绑定装货箱

**相关API**:
- `/uocs-tes/fyWaybill/getLoadingBoxListByOrder` - 获取装货箱列表
- `/uocs-tes/fyWaybill/bindingLoadingBox` - 绑定装货箱

---

## 功能3: 运单处理

**入口**: FengYiDelveryMainPager.goToDeliveryPager()

**主界面**: FengYiDeliveryPager (fengyi_delivery_pager.dart)

### Tab标签页结构 (5个标签)

#### Tab 0: 待揽件 (orderStatus=0)

**含义**: 运单已创建，等待取件

**可执行操作**:
1. 上传揽件照片
2. 确认揽件

**相关API**:
```
POST /uocs-tes/fyWaybill/selectTcWaybillList
  params: {
    "pageNum": 0,
    "pageSize": 10,
    "terminalId": "航站ID",
    "orderStatus": 0,
    "tcWaybillNo": "运单号(可选)"
  }

POST /uocs-tes/fyWaybill/confirmPickup
  params: {
    "tcWaybillNo": "运单号",
    "userCode": "用户ID",
    "fileList": [文件列表]
  }
```

---

#### Tab 1: 待派件 (orderStatus=1)

**含义**: 已揽件，等待派件

**可执行操作**:
1. 查看装货箱列表
2. 更新派件地址
3. 上传派件照片
4. 确认接收（中转站）或 确认派件（最后一公里）
5. 异常处理

**相关API**:
```
POST /uocs-tes/fyWaybill/selectTcWaybillList
  params: {
    "pageNum": 1,
    "pageSize": 10,
    "terminalId": "航站ID",
    "orderStatus": 1,
    "tcWaybillNo": "运单号(可选)"
  }

GET /uocs-tes/fyWaybill/getLoadingBoxListByOrder
  params: {
    "waybillNo": "运单号"
  }

POST /uocs-tes/fyWaybill/confirmReceive
  params: {
    "id": "订单ID",
    "tcWaybillNo": "运单号",
    "operater": "操作员ID",
    "operateName": "操作员名称",
    "transportWay": "配送方式",
    "fyOrderId": "丰翼订单ID",
    "userCode": "用户ID",
    "fileList": [文件列表]
  }

POST /uocs-tes/fyWaybill/confirmSend
  params: {
    "id": "订单ID",
    "tcWaybillNo": "运单号",
    "operater": "操作员ID",
    "operateName": "操作员名称",
    "sender": "派件人",
    "sendName": "派件人名称",
    "sendTm": "派件时间",
    "fileList": [文件列表]
  }
```

---

#### Tab 2: 待起飞 (orderStatus=3)

**含义**: 订单准备飞送

**可执行操作**:
1. 查看运单信息
2. 选择无人机和航线
3. 绑定装货箱
4. 执行飞送任务

**相关API**: 同Tab 1

---

#### Tab 3: 已完成 (orderStatus=4)

**含义**: 运单派件完成

**可执行操作**:
- 查看完成的运单记录
- 打印凭证

**相关API**:
```
POST /uocs-tes/fyWaybill/selectTcWaybillList
  params: {
    "pageNum": 3,
    "pageSize": 10,
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
POST /uocs-tes/fyWaybill/selectTcWaybillList
  params: {
    "pageNum": 4,
    "pageSize": 10,
    "terminalId": "航站ID",
    "orderStatus": 5
  }

POST /uocs-sys/dict/getDictListByBusinessCode
  params: {
    "dictTypeCode": "WaybillExceptionReason"
  }
```

---

## 完整的运单生命周期

### 完整订单处理流程

```
1. 待揽件 (orderStatus=0)
   ├─ 查询 /uocs-tes/fyWaybill/selectTcWaybillList (orderStatus=0)
   ├─ 获取订单列表
   ├─ 用户上传揽件照片
   └─ 调用 /uocs-tes/fyWaybill/confirmPickup
        └─ 成功：进入待派件状态

2. 待派件 (orderStatus=1)
   ├─ 查询 /uocs-tes/fyWaybill/selectTcWaybillList (orderStatus=1)
   ├─ 获取订单列表
   ├─ 可选：查看装货箱 /uocs-tes/fyWaybill/getLoadingBoxListByOrder
   ├─ 用户上传派件照片
   ├─ 选择操作：
   │  ├─ 确认接收: /uocs-tes/fyWaybill/confirmReceive
   │  ├─ 确认派件: /uocs-tes/fyWaybill/confirmSend
   │  └─ 异常处理: /uocs-tes/fyWaybill/orderException
   └─ 根据选择进入相应状态

3. 待起飞 (orderStatus=3)
   └─ 系统自动或用户手动进入此状态

4. 已完成 (orderStatus=4)
   ├─ 成功派件后进入此状态
   ├─ 自动触发打印: printFengYi()
   └─ 显示在"已完成"标签页

5. 异常 (orderStatus=5)
   ├─ 如果选择异常处理
   ├─ 调用 /uocs-sys/dict/getDictListByBusinessCode
   ├─ 获取异常原因列表
   └─ 用户选择异常类型提交
```

---

## 核心数据模型

### OrderBeanDataFengYi (运单数据)
```dart
class OrderBeanDataFengYi {
  int? id;                           // 运单ID
  String? fyCode;                    // 丰翼运单编码
  String? tcWaybillNo;               // 同城子运单号
  int? ordeStatus;                   // 订单状态 (0-5)
  String? senderName;                // 寄件人
  String? senderTel;                 // 寄件人电话
  String? senderAddr;                // 寄件地址
  String? receiverName;              // 收件人
  String? receiverTel;               // 收件人电话
  String? receiverAddr;              // 收件地址
  String? createTm;                  // 创建时间
  String? receiver;                  // 揽件人ID
  String? receiveName;               // 揽件人名称
  String? receiveTm;                 // 揽件时间
  String? sender;                    // 派件人ID
  String? sendName;                  // 派件人名称
  String? sendTm;                    // 派件时间
  String? exceptionTm;               // 异常时间
  String? exceptionReason;           // 异常原因
}
```

### DeliverySameCityTerminalBean (航站信息)
```dart
class DeliverySameCityTerminalBean {
  int? terminalId;
  String? terminalName;              // 航站名称
  String? terminalCode;              // 航站代码
  String? terminalAddress;           // 航站地址
}
```

---

## 订单状态说明

| 值 | 状态名 | 描述 | 所在Tab |
|----|--------|------|--------|
| 0 | 待揽件 | 订单已创建，等待取件 | Tab 0 |
| 1 | 待派件 | 已取件，等待派件 | Tab 1 |
| 3 | 待起飞 | 准备飞送 | Tab 2 |
| 4 | 已完成 | 派件完成 | Tab 3 |
| 5 | 异常 | 异常订单 | Tab 4 |

---

## 用户界面流程

### 主要操作流程

```
1️⃣ 选择航站
   - 点击AppBar中的航站名称
   - 输入搜索关键词
   - 选择航站

2️⃣ 进入运单处理
   - 自动进入"待揽件"Tab
   - 显示该航站下的所有待揽件运单

3️⃣ 待揽件处理
   - 扫描运单号或从列表选择
   - 上传揽件照片
   - 滑动验证
   - 确认揽件成功

4️⃣ 待派件处理
   - 切换到"待派件"Tab
   - 选择运单
   - 上传派件照片
   - 选择操作（派件/接收/异常）
   - 滑动验证
   - 完成操作

5️⃣ 打印
   - 派件成功后自动打印凭证
   - 需要蓝牙连接打印机
```

---

## 特殊功能

### 1. 蓝牙打印
- **时机**: 运单派件完成后
- **方法**: `printFengYi(OrderBeanDataFengYi bean)`
- **需要**: BLE蓝牙连接打印机
- **位置**: fengyi_delivery_pager.dart:215

### 2. 照片上传
- **支持**: 揽件、接收、派件阶段
- **格式**: UploadImageBean对象列表
- **位置**: goToSelectImagePager() 方法

### 3. 滑动验证
- **作用**: 防止误操作
- **位置**: SlideVerifyWidget
- **失败处理**: resetWidget() 重置验证

### 4. 搜索功能
- **范围**: 在当前状态的运单中搜索
- **参数**: tcWaybillNo（运单号）
- **防抖**: 1秒延迟

---

## API调用汇总表

| 功能 | 接口 | 方式 | 参数关键字 | 位置 |
|------|------|------|-----------|------|
| 选择航站 | `/uocs-tes/fyWaybill/selectTcTerminalList` | POST | query | main_pager:684 |
| 查询订单列表 | `/uocs-tes/fyWaybill/selectTcWaybillList` | POST | orderStatus | pager:5935 |
| 获取装货箱 | `/uocs-tes/fyWaybill/getLoadingBoxListByOrder` | GET | waybillNo | pager:3580 |
| 确认揽件 | `/uocs-tes/fyWaybill/confirmPickup` | POST | tcWaybillNo | pager:2972 |
| 确认接收 | `/uocs-tes/fyWaybill/confirmReceive` | POST | tcWaybillNo | pager:5786 |
| 确认派件 | `/uocs-tes/fyWaybill/confirmSend` | POST | tcWaybillNo | pager:5825 |
| 异常处理 | `/uocs-tes/fyWaybill/orderException` | POST | exceptionType | pager:5855 |
| 获取异常类型 | `/uocs-sys/dict/getDictListByBusinessCode` | POST | dictTypeCode | pager:5641 |
| 修改订单 | `/uocs-tes/fyWaybill/changeOrder` | POST | id | pager:4557 |

---

## 重要说明

1. **进入前要求**: 必须先选择航站，否则无法进入运单处理页面

2. **GPS定位**: 选择航站时会自动获取用户GPS位置，用于显示距离

3. **权限要求**:
   - 位置权限 (GPS定位)
   - 蓝牙权限 (打印)
   - 摄像头权限 (拍照上传)

4. **用户信息**: 所有操作都从 SpUtil 获取:
   - userId
   - userName

5. **Tab刷新**: 切换Tab时自动刷新该Tab的数据

6. **搜索机制**: 搜索框的输入会触发1秒后的查询，支持增量搜索