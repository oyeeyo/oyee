# 血站配送 API 接口文档

## 1. 查询订单列表（分页）
**接口路径**: `/uocs-tes/cbsOrder/getOrderListInApp/{page}/{pageSize}`
**请求方式**: POST
**所在文件**: `blood_banks_pager.dart:85-87`, `search_order_pager.dart:86-88`

### 请求参数
```json
{
  "pickupTerminalId": "string",         // 揽件航站ID，可选
  "receiveTerminalId": "string",        // 目的地航站ID，可选
  "orderStatusType": "int",             // 订单状态类型，可选
                                        // 1 = 进行中 (待揽件、待接收)
                                        // 2 = 待接收
                                        // 3 = 已完成或异常
  "orderCreateTmType": "int",           // 订单创建时间类型，可选
                                        // 1 = 仅看今天
                                        // 2 = 本周
                                        // 3 = 所有
  "searchWord": "string",               // 搜索关键词（订单号/出库单号/无人机编号/箱号），可选
  "pageNum": "int",                     // 页码（从1开始）
  "pageSize": "int"                     // 每页条数
}
```

### 响应数据
```json
{
  "success": "boolean",                 // 请求是否成功
  "errorMessage": "string",             // 错误信息（success为false时）
  "obj": {
    "total": "int",                     // 总记录数
    "records": [
      {
        "id": "string",                 // 订单ID
        "orderNo": "string",            // 订单号
        "outboundOrderNo": "string",    // 出库单号
        "pickupTerminalId": "string",   // 揽件航站ID
        "pickupTerminalName": "string", // 揽件航站名称
        "receiveTerminalId": "string",  // 目的地航站ID
        "receiveTerminalName": "string",// 目的地航站名称
        "boxNum": "int",                // 箱数
        "boxNoList": "string",          // 箱号列表（逗号分隔）
        "createTm": "string",           // 订单创建时间，格式: yyyy-MM-dd HH:mm:ss
        "orderStatus": "int",           // 订单状态
                                        // 0 = 待揽件
                                        // 1 = 待接收
                                        // 4 = 已接收
                                        // 5 = 已签字确认
                                        // 6 = 异常订单
        "realPickupPerson": "string",   // 实际揽件人ID，可为null
        "realPickupPersonName": "string",// 实际揽件人姓名，可为null
        "realPickupTm": "string",       // 实际揽件时间，可为null
        "realReceiver": "string",       // 实际接收人ID，可为null
        "realReceiverName": "string",   // 实际接收人姓名，可为null
        "realReceiveTm": "string",      // 实际接收时间，可为null
        "signConfirmTm": "string",      // 签字确认时间（status=5时），可为null
        "isCarry": "int",               // 订单是否由丰翼承运（status=6时）
                                        // 1 = 是
                                        // 2 = 否
        "exceptionTypeName": "string",  // 异常类型名称（status=6时）
        "exceptionSubmitter": "string", // 异常提交人ID（status=6时）
        "exceptionSubmitterName": "string", // 异常提交人姓名（status=6时）
        "exceptionSubmitTm": "string",  // 异常提交时间（status=6时）
        "exceptionRemark": "string",    // 异常备注（status=6时）
        "boxInfoList": [                // 箱子信息列表
          {
            "id": "string",             // 箱子ID
            "boxNo": "string",          // 箱号
            "weight": "int"             // 重量(kg)
          }
        ]
      }
    ]
  }
}
```

---

## 2. 查询订单统计
**接口路径**: `/uocs-tes/cbsOrder/getOrderStats`
**请求方式**: POST
**所在文件**: `blood_banks_pager.dart:128`, `search_order_pager.dart:126`

### 请求参数
```json
{
  "pickupTerminalId": "string",         // 揽件航站ID，可选
  "receiveTerminalId": "string",        // 目的地航站ID，可选
  "orderStatusType": "int",             // 订单状态类型（通常为1 = 进行中）
  "orderCreateTmType": "int"            // 订单创建时间类型，可选
}
```

### 响应数据
```json
{
  "success": "boolean",
  "errorMessage": "string",
  "obj": {
    "inProgressCount": "int"            // 进行中的订单数量
  }
}
```

---

## 3. 获取揽件航站下拉列表
**接口路径**: `/uocs-tes/cbsOrder/getPickupTerminalDropList`
**请求方式**: POST
**所在文件**: `blood_banks_pager.dart:989-990`, `search_order_pager.dart:939-940`

### 请求参数
```json
{
  // 无参数，直接POST请求
}
```

### 响应数据
```json
{
  "success": "boolean",
  "errorMessage": "string",
  "obj": [
    {
      "pickupTerminalId": "string",     // 揽件航站ID
      "pickupTerminalName": "string"    // 揽件航站名称
    }
  ]
}
```

---

## 4. 获取目的地航站下拉列表
**接口路径**: `/uocs-tes/cbsOrder/getReceiveTerminalDropList`
**请求方式**: POST
**所在文件**: `blood_banks_pager.dart:992-993`, `search_order_pager.dart:941-942`

### 请求参数
```json
{
  // 无参数，直接POST请求
}
```

### 响应数据
```json
{
  "success": "boolean",
  "errorMessage": "string",
  "obj": [
    {
      "receiveTerminalId": "string",    // 目的地航站ID
      "receiveTerminalName": "string"   // 目的地航站名称
    }
  ]
}
```

---

## 5. 获取配送明细
**接口路径**: `/uocs-tes/cbsOrder/getDeliveryDetail/{orderNo}`
**请求方式**: GET
**所在文件**: `blood_banks_pager.dart:1397-1398`, `search_order_pager.dart:1343`

### 请求参数
```json
{
  "orderNo": "string"                   // 订单号（URL路径参数）
}
```

### 响应数据
```json
{
  "success": "boolean",
  "errorMessage": "string",
  "obj": [
    {
      "boxNo": "string",                // 箱号
      "transportWay": "int",            // 配送方式
                                        // 0 = 无人机配送
                                        // 1 = 陆地配送
                                        // 2 = 无人机+陆地配送
                                        // 3 = 配送方式未知
      "comNumList": "string"            // 无人机编号列表（逗号分隔），可为null
    }
  ]
}
```

---

## 6. 确认接收订单
**接口路径**: `/uocs-tes/cbsOrder/confirmReceive`
**请求方式**: POST
**所在文件**: `blood_banks_pager.dart:1425`, `search_order_pager.dart:1369`

### 请求参数
```json
{
  "id": "string",                       // 订单ID
  "realReceiver": "string",             // 接收人ID（userId）
  "updater": "string"                   // 更新人ID（userId）
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

## 7. 标记订单为异常
**接口路径**: `/uocs-tes/cbsOrder/label2AbnormalOrder`
**请求方式**: POST
**所在文件**: `blood_banks_pager.dart:2038`, `search_order_pager.dart:1995`

### 请求参数
```json
{
  "id": "string",                       // 订单ID
  "exceptionTypeId": "string",          // 异常类型ID
  "exceptionTypeName": "string",        // 异常类型名称
  "exceptionRemark": "string",          // 异常备注，可为null
  "isCarry": "int",                     // 订单是否由丰翼承运
                                        // 1 = 是
                                        // 2 = 否
  "exceptionSubmitter": "string",       // 异常提交人ID（userId）
  "updater": "string"                   // 更新人ID（userId）
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

## 8. 确认揽件/更新揽件信息
**接口路径**:
- `/uocs-tes/cbsOrder/updateWhenConfirmPickup` (初始确认揽件)
- `/uocs-tes/cbsOrder/updatePickup` (更新揽件信息)

**请求方式**: POST
**所在文件**: `blood_banks_pager.dart:2161-2164`, `search_order_pager.dart:2106-2108`

### 请求参数
```json
{
  "id": "string",                       // 订单ID
  "boxInfoList": [                      // 箱子信息列表
    {
      "id": "string",                   // 箱子ID
      "boxNo": "string",                // 箱号
      "weight": "int"                   // 重量(kg)
    }
  ],
  "updater": "string"                   // 更新人ID（userId）
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

## 9. 获取异常类型列表
**接口路径**: `/uocs-sys/dict/getDictListByBusinessCode`
**请求方式**: POST
**所在文件**: `blood_banks_pager.dart:2056`, `search_order_pager.dart:2013`

### 请求参数
```json
{
  "dictTypeCode": "string"              // 字典类型代码，值为 "orderExceptionType"
}
```

### 响应数据
```json
{
  "success": "boolean",
  "errorMessage": "string",
  "obj": [
    {
      "id": "string",                   // 异常类型ID
      "businessVal": "string",          // 异常类型名称（如：丢失、破损、延误等）
      "businessCode": "string"          // 异常类型代码
    }
  ]
}
```

---

## 数据模型定义

### BloodBankBean (订单)
```dart
class BloodBankBean {
  String? id;                           // 订单ID
  String? orderNo;                      // 订单号
  String? outboundOrderNo;              // 出库单号
  String? pickupTerminalId;             // 揽件航站ID
  String? pickupTerminalName;           // 揽件航站名称
  String? receiveTerminalId;            // 目的地航站ID
  String? receiveTerminalName;          // 目的地航站名称
  int? boxNum;                          // 箱数
  String? boxNoList;                    // 箱号列表（逗号分隔）
  String? createTm;                     // 订单创建时间
  int? orderStatus;                     // 订单状态 (0-6)
  String? realPickupPerson;             // 实际揽件人ID
  String? realPickupPersonName;         // 实际揽件人姓名
  String? realPickupTm;                 // 实际揽件时间
  String? realReceiver;                 // 实际接收人ID
  String? realReceiverName;             // 实际接收人姓名
  String? realReceiveTm;                // 实际接收时间
  String? signConfirmTm;                // 签字确认时间
  int? isCarry;                         // 是否由丰翼承运 (1=是, 2=否)
  String? exceptionTypeName;            // 异常类型名称
  String? exceptionSubmitter;           // 异常提交人ID
  String? exceptionSubmitterName;       // 异常提交人姓名
  String? exceptionSubmitTm;            // 异常提交时间
  String? exceptionRemark;              // 异常备注
  List<BoxInfo> boxInfoList = [];       // 箱子信息列表
}
```

### BoxInfo (箱子信息)
```dart
class BoxInfo {
  String? id;                           // 箱子ID
  String? boxNo;                        // 箱号
  int? weight;                          // 重量(kg)
}
```

### DeliveryDetailBean (配送明细)
```dart
class DeliveryDetailBean {
  String? boxNo;                        // 箱号
  int? transportWay;                    // 配送方式 (0-3)
  String? comNumList;                   // 无人机编号列表
}
```

### ExceptionBean (异常类型)
```dart
class ExceptionBean {
  String? id;                           // 异常类型ID
  String? businessVal;                  // 异常类型名称
  String? businessCode;                 // 异常类型代码
}
```

### PickupTerminalBean (揽件航站)
```dart
class PickupTerminalBean {
  String? pickupTerminalId;             // 揽件航站ID
  String? pickupTerminalName;           // 揽件航站名称
}
```

### ReceiveTerminalBean (目的地航站)
```dart
class ReceiveTerminalBean {
  String? receiveTerminalId;            // 目的地航站ID
  String? receiveTerminalName;          // 目的地航站名称
}
```

---

## 订单状态定义

| 状态值 | 状态名称 | 描述 | 可执行操作 |
|--------|---------|------|----------|
| 0 | 待揽件 | 订单已创建，等待揽件 | 确认揽件、订单异常 |
| 1 | 待接收 | 已揽件，等待接收 | 更新揽件信息、确认接收、订单异常 |
| 4 | 已接收 | 已确认接收 | 无 |
| 5 | 已签字确认 | 已签字确认 | 无 |
| 6 | 异常订单 | 订单已标记为异常 | 无 |

---

## 配送方式定义

| 值 | 配送方式 | 描述 |
|----|---------|------|
| 0 | 无人机配送 | 仅无人机配送 |
| 1 | 陆地配送 | 仅陆地配送 |
| 2 | 混合配送 | 无人机+陆地配送 |
| 3 | 未知 | 配送方式未知 |

---

## 常见业务流程

### 订单处理流程
1. **加载订单列表** - 调用 `/uocs-tes/cbsOrder/getOrderListInApp`
2. **筛选订单** - 根据航站、时间等条件筛选
3. **查看配送明细** - 调用 `/uocs-tes/cbsOrder/getDeliveryDetail` (status=1时)
4. **处理订单**：
   - **待揽件状态(0)**: 点击"确认揽件" → 调用 `/uocs-tes/cbsOrder/updateWhenConfirmPickup`
   - **待接收状态(1)**:
     - 点击"更新揽件信息" → 调用 `/uocs-tes/cbsOrder/updatePickup`
     - 点击"确认接收" → 调用 `/uocs-tes/cbsOrder/confirmReceive`
     - 点击"订单异常" → 调用 `/uocs-tes/cbsOrder/label2AbnormalOrder`

### 异常处理流程
1. 点击"订单异常"按钮
2. 获取异常类型列表 - 调用 `/uocs-sys/dict/getDictListByBusinessCode`
3. 用户选择异常类型和输入备注
4. 提交异常 - 调用 `/uocs-tes/cbsOrder/label2AbnormalOrder`
5. 订单状态变更为异常(6)

### 筛选流程
1. 点击"筛选"按钮
2. 调用 `/uocs-tes/cbsOrder/getPickupTerminalDropList` 获取揽件航站列表
3. 调用 `/uocs-tes/cbsOrder/getReceiveTerminalDropList` 获取目的地航站列表
4. 用户选择筛选条件（航站、时间）
5. 刷新订单列表数据

---

## 特殊说明

### 箱号列表显示
- 在 UI 中显示时，需要将 `boxNoList` 中的逗号替换为换行符：
  ```dart
  String name = boxName!.replaceAll(',', '\n');
  ```

### 订单操作按钮
- 状态0（待揽件）: 显示"确认揽件"按钮
- 状态1（待接收）: 显示"更新揽件信息"和"确认接收"按钮
- 状态4-6（已接收/已签字/异常）: 不显示操作按钮

### 航站信息获取
- 搜索页面支持关键词搜索（订单号、出库单号、无人机编号、箱号）
- 搜索实现防抖，延迟500ms执行
