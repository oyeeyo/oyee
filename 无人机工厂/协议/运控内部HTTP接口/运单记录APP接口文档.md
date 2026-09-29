# 运单记录 API 接口文档

## 1. 查询运单列表 (分页)
**接口路径**: `/selectWayBillListPage`
**请求方式**: POST
**所在文件**: `waybill_history_pager.dart:1590`

### 请求参数
```json
{
  "keyword": "string",              // 搜索关键词（无人机编号/运单号/操作人工号或姓名），可选
  "loadTerminalName": "string",     // 装货航站名称，可选
  "city": "string",                 // 城市，可选（移除了市字）
  "status": "int",                  // 运单状态，可选
                                    // 1 = 已扫描
                                    // 2 = 已接收
                                    // 3 = 已删除
                                    // 不传或其他值 = 不限状态
  "time": "string",                 // 日期，格式: yyyy-MM-dd，可选
  "pageNum": "int",                 // 页码（从1开始）
  "pageSize": "int"                 // 每页条数
}
```

### 响应数据
```json
{
  "success": "boolean",             // 请求是否成功
  "errorMessage": "string",         // 错误信息（success为false时）
  "obj": {
    "total": "int",                 // 总记录数
    "records": [
      {
        "id": "int",                // 运单组ID
        "waybillCode": "string",    // 运单编码
        "uavNo": "string",          // 无人机编号
        "city": "string",           // 装货城市
        "loadCity": "string",       // 装货城市
        "receiveCity": "string",    // 接收城市
        "totalWeight": "double",    // 总重量(kg)
        "status": "int",            // 状态 (未使用)
        "count": "int",             // 运单数量
        "waybillDetailsList": [
          {
            "id": "int",            // 运单详情ID
            "waybillId": "int",     // 运单ID
            "waybillNo": "string",  // 运单号
            "packageType": "int",   // 包裹类型
            "packageNo": "string",  // 包裹号
            "createTm": "string",   // 创建时间，格式: yyyy-MM-dd HH:mm:ss
            "creater": "string",    // 创建人ID/工号
            "createrName": "string",// 创建人姓名
            "receiveTm": "string",  // 接收时间，格式: yyyy-MM-dd HH:mm:ss，可为null
            "receiver": "string",   // 接收人ID/工号，可为null
            "receiverName": "string",// 接收人姓名，可为null
            "status": "int",        // 运单状态
                                    // 1 = 已扫描
                                    // 2 = 已接收
                                    // 3 = 已删除
            "loadTerminalName": "string",  // 装货航站名称
            "loadTerminalId": "string",    // 装货航站ID
            "loadCmsSiteCode": "string"    // 装货CMS地点代码
          }
        ]
      }
    ]
  }
}
```

---

## 2. 删除运单
**接口路径**: `/deleteWayBill`
**请求方式**: POST
**所在文件**: `waybill_history_pager.dart:1636`

### 请求参数
```json
{
  "id": "int",                      // 运单详情ID
  "totalWeight": "string",          // 删除后的总重量(kg)，格式为字符串
  "creater": "string",              // 创建人ID/工号
  "waybillNo": "string",            // 运单号
  "waybillId": "int"                // 运单ID
}
```

### 响应数据
```json
{
  "success": "boolean",             // 删除是否成功
  "errorMessage": "string",         // 错误信息（success为false时）
  "obj": null                       // 无返回数据
}
```

---

## 3. 选择航站
**接口路径**: `/uocs-eas/app/chooseTerminal`
**请求方式**: POST
**所在文件**: `choose_all_terminal_pager.dart`

### 请求参数
```json
{
  "terminalId": "string"            // 航站ID
}
```

### 响应数据
```json
{
  "success": "boolean",
  "errorMessage": "string",
  "obj": {
    "id": "string",                 // 航站ID
    "name": "string",               // 航站名称
    "code": "string",               // 航站代码
    "state": "int",                 // 航站状态 1=启用
    "longitude": "string",          // 经度
    "latitude": "string",           // 纬度
    "cityName": "string",           // 城市名称
    "distance": "string",           // 距离(km)，格式为字符串数字
    "cmsSiteCode": "string"         // CMS地点代码
  }
}
```

---

## 4. 获取航站列表
**接口路径**: `/uocs-tes/location/terminalList/{page}/{pageSize}`
**请求方式**: POST
**所在文件**: `waybill_history_pager.dart:245-246`

### 请求参数
```json
{
  "longitude": "string",            // 当前经度，可选
  "latitude": "string",             // 当前纬度，可选
  "keyword": "string",              // 搜索关键词（航站名称），可选
  "state": "int",                   // 航站状态过滤，可选 (1=启用)
  "page": "int",                    // 页码（URL路径参数）
  "pageSize": "int"                 // 每页条数（URL路径参数）
}
```

### 响应数据
```json
{
  "success": "boolean",
  "errorMessage": "string",
  "obj": {
    "total": "int",                 // 总航站数
    "records": [
      {
        "id": "string",             // 航站ID
        "name": "string",           // 航站名称
        "code": "string",           // 航站代码
        "state": "int",             // 航站状态 1=启用
        "longitude": "string",      // 经度
        "latitude": "string",       // 纬度
        "cityName": "string",       // 城市名称
        "distance": "string",       // 距离(km)，根据经纬度计算
        "cmsSiteCode": "string"     // CMS地点代码
      }
    ]
  }
}
```

---

## 5. 获取机构配置列表
**接口路径**: `/uocs-tes/terminal/selectNsCfgList`
**请求方式**: GET
**所在文件**: `waybill_history_pager.dart:130`

### 请求参数
```json
{
  // 无参数，直接GET请求
}
```

### 响应数据
```json
{
  "success": "boolean",
  "errorMessage": "string",
  "obj": [
    {
      // 运单配置数据（NsCfgDto）
      // 具体字段取决于服务端实现
    }
  ]
}
```

---

## 6. 切换运单服务地址
**接口路径**: `/uocs-tes/waybillManage/waybillServiceSwitch`
**请求方式**: GET
**所在文件**: `waybill_history_pager.dart:92`

### 请求参数
```json
{
  // 无参数
}
```

### 响应数据
```json
{
  "success": "boolean",
  "errorMessage": "string",
  "obj": "int"                      // 1=使用wayBillManage服务，其他值使用frpBase服务
}
```

---

## 数据模型定义

### WayBillBean (运单组)
```dart
class WayBillBean {
  int? id;                          // 运单组ID
  String? waybillCode;              // 运单编码
  String? uavNo;                    // 无人机编号
  String? city;                     // 装货城市
  double? totalWeight;              // 总重量(kg)
  int? status;                      // 状态(未使用)
  int? count;                       // 运单数量
  List<WaybillDetailsBean> waybillDetailsList;  // 运单详情列表
  bool isExpend;                    // UI展开状态
  String? loadCity;                 // 装货城市
  String? receiveCity;              // 接收城市
}
```

### WaybillDetailsBean (运单详情)
```dart
class WaybillDetailsBean {
  int? id;                          // 运单详情ID
  int? waybillId;                   // 运单ID
  String? waybillNo;                // 运单号
  int? packageType;                 // 包裹类型
  String? packageNo;                // 包裹号
  String? createTm;                 // 创建时间 (yyyy-MM-dd HH:mm:ss)
  String? creater;                  // 创建人ID/工号
  String? createrName;              // 创建人姓名
  String? receiveTm;                // 接收时间 (yyyy-MM-dd HH:mm:ss)
  String? receiver;                 // 接收人ID/工号
  String? receiverName;             // 接收人姓名
  int? status;                      // 运单状态 (1=已扫描, 2=已接收, 3=已删除)
  String? loadTerminalName;         // 装货航站名称
  String? loadTerminalId;           // 装货航站ID
  String? loadCmsSiteCode;          // 装货CMS地点代码
}
```

### TerminalBean (航站)
```dart
class TerminalBean {
  String? id;                       // 航站ID
  String? name;                     // 航站名称
  String? code;                     // 航站代码
  int? state;                       // 航站状态 (1=启用)
  String? longitude;                // 经度
  String? latitude;                 // 纬度
  String? cityName;                 // 城市名称
  String? distance;                 // 距离(km)
  String? cmsSiteCode;              // CMS地点代码
  bool? isLastSelected;             // 是否最近选中
}
```

---

## 状态码定义

### 运单状态 (waybillDetailsList[].status)
| 值 | 含义 | 样式 |
|---|---|---|
| 1 | 已扫描 | 橙色标签 |
| 2 | 已接收 | 绿色标签 |
| 3 | 已删除 | 灰色标签 |

### 航站状态 (terminal.state)
| 值 | 含义 |
|---|---|
| 1 | 启用 |
| 其他 | 禁用 |

---

## 常见业务流程

### 查询运单列表流程
1. 调用 `getData()` 方法
2. 构建查询参数（keyword, loadTerminalName, city, status, time, pageNum, pageSize）
3. 调用 `/selectWayBillListPage` 接口
4. 返回分页的运单数据，包含详细信息

### 删除运单流程
1. 用户点击删除按钮
2. 弹出确认对话框
3. 调用 `deleteItem()` 方法
4. 发送 `/deleteWayBill` 请求
5. 更新本地列表数据（状态改为3=已删除）

### 选择航站流程
1. 用户点击"不限航站"或航站筛选
2. 弹出航站选择窗口
3. 获取GPS定位信息（经纬度）
4. 调用 `/uocs-tes/location/terminalList` 获取航站列表
5. 用户选择航站后，保存到 SharedPreferences
6. 调用 `/uocs-eas/app/chooseTerminal` 接口
7. 后续查询运单时使用选中的航站进行过滤

---

## 特殊处理说明

### GPS定位
- 优先获取缓存位置：`Geolocator.getLastKnownPosition()`
- 如果缓存为null，获取当前位置：`Geolocator.getCurrentPosition()`
- 精度设置：`LocationAccuracy.high` 或 `LocationAccuracy.best`

### 时间范围判断
- 如果距离 > 1000m：提示选择航站
- 如果距离 ≤ 1000m：检查时间
  - 检查航站信息是否超过1天，超过则需重新选择航站

### 数据本地化存储
- SharedPreferences Key: `waybill-terminal` - 保存当前航站信息（JSON）
- SharedPreferences Key: `waybill-terminal-time` - 保存航站选择时间戳

### 运单代码规则
- 通过 `NsCfgCache` 和 `NsCfgUtils` 管理运单编码规则
- 需要在初始化时调用 `/uocs-tes/terminal/selectNsCfgList` 获取配置
1