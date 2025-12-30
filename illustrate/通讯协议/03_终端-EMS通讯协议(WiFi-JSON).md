# 终端通讯协议 (WiFi JSON)

## 一、协议概述

### 1.1 通讯架构
```
PC终端 ←→ [WiFi/TCP] ←→ EMS主控(RK3566)
(客户端)                    (服务端)
```

### 1.2 通讯参数
| 参数名称 | 默认值 | 可配置范围 | 说明 |
|---------|--------|-----------|------|
| 传输协议 | TCP/IP | - | 面向连接 |
| 监听端口 | 9000 | 1024~65535 | EMS终端服务端口（注：8080已分配给OCPP） |
| 数据格式 | JSON | - | UTF-8编码 |
| 心跳周期 | 10秒 | 5~60秒 | 连接保活 |
| 连接超时 | 30秒 | 10~120秒 | 无心跳断开 |
| 最大连接数 | 5 | 1~10 | 同时连接的终端数，超限拒绝新连接 |

---

## 二、TCP消息分帧

### 2.1 分帧格式
采用**长度前缀**方式解决TCP粘包/拆包问题：

```
┌──────────────────┬─────────────────────────────┐
│ 4字节长度(大端)  │         JSON内容            │
│   Length         │ {"version":"1.0",...}       │
└──────────────────┴─────────────────────────────┘
```

**字段说明**：
| 字段 | 长度 | 字节序 | 说明 |
|------|------|--------|------|
| Length | 4字节 | 大端(Big-Endian) | JSON内容的字节长度(不含Length本身) |
| JSON | Length字节 | - | UTF-8编码的JSON字符串 |

**示例**（十六进制）：
```
发送 {"version":"1.0"} (长度=18字节):
00 00 00 12 7B 22 76 65 72 73 69 6F 6E 22 3A 22 31 2E 30 22 7D
└─Length─┘ └──────────────── JSON UTF-8 ────────────────────┘
```

### 2.2 长度限制与异常处理
| 场景 | 处理策略 |
|------|----------|
| Length = 0 | 丢弃，继续读取下一帧 |
| Length > 1MB (1048576) | 断开连接，记录日志 |
| JSON语法错误（无法解析） | 丢弃该帧，记录日志，继续处理下一帧（无法回包） |
| JSON可解析但字段校验失败 | 返回错误码10004，继续处理下一帧 |
| 单帧读取超时(5秒) | 断开连接 |

**超时说明**：
- 单帧读取超时：开始读取一帧后，5秒内未读完Length字节则断开
- 连接空闲超时：由心跳机制控制（30秒无心跳断开），与单帧超时无关

**错误区分说明**：
- JSON语法错误：`{invalid json`、缺少引号等，无法提取msg_id
- 字段校验失败：JSON合法但缺少必填字段、类型错误等，可提取msg_id回包

### 2.3 接收处理流程
```
1. 读取4字节，解析为Length(大端)
2. 校验Length范围(0 < Length ≤ 1MB)
3. 读取Length字节作为JSON内容
4. 解析JSON，处理消息
5. 重复步骤1
```

---

## 三、消息帧格式

### 3.1 帧结构
所有消息采用JSON格式，外层包装统一结构：

```json
{
  "version": "1.0",
  "msg_id": "uuid-string",
  "msg_type": "request|response|notify|heartbeat",
  "timestamp": 1703750400000,
  "token": "session_token",
  "action": "action_name",
  "data": { ... },
  "error": { ... }
}
```

### 3.2 字段说明
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| version | string | 是 | 协议版本，当前"1.0" |
| msg_id | string | 是 | 消息唯一标识，UUID格式 |
| msg_type | string | 是 | 消息类型枚举 |
| timestamp | number | 是 | 消息发送时间，毫秒级Unix时间戳 |
| token | string | 条件 | 会话令牌，登录成功后的请求必须携带 |
| action | string | 条件 | 操作名称，heartbeat响应可省略，其他消息必填 |
| data | object | 否 | 业务数据，请求/响应/通知的具体内容 |
| error | object | 否 | 错误信息，仅响应失败时存在 |

**msg_id规则**：
- 请求方生成UUID格式的msg_id
- 响应的msg_id必须等于原请求的msg_id（用于请求-响应匹配）
- 通知消息由EMS生成新的msg_id

**token字段规则**：
- `sys_login` 请求：不需要token
- `sys_heartbeat`：不需要token
- 其他所有请求：必须携带有效token
- 响应和通知：不需要token

**token生命周期**：
- 有效期：登录响应中的`expires_in`秒（默认1800秒）
- 登出失效：调用`sys_logout`后立即失效
- 断线处理：连接断开后token保持有效，重连后可继续使用
- 多端登录：同一用户允许多个终端同时登录，各自独立token
- 连接数限制：达到最大连接数时拒绝新连接（错误码10007），不影响已有token

**token安全属性**：
| 属性 | 要求 | 说明 |
|------|------|------|
| 格式 | 十六进制字符串 | 仅包含0-9、a-f字符 |
| 长度 | 64字符（256位） | 提供足够的熵 |
| 随机性 | 密码学安全随机数 | 使用CSPRNG生成 |
| 刷新机制 | 不支持 | 过期后需重新登录 |
| 抗重放 | 依赖TLS | 内网环境不实现，生产环境建议TLS |

**token生成示例**（伪代码）：
```
token = hex(CSPRNG(32字节))  // 生成32字节随机数，转为64字符十六进制
```

**字段缺省规则**：
- 字段值为`null`等同于字段不存在
- 查询过滤条件为`null`表示不过滤该条件
- 接收方必须忽略未知字段（协议兼容性）

**数值字段边界处理（全局规则）**：

| 字段类型 | 越界处理 | 精度处理 | 说明 |
|----------|----------|----------|------|
| 上报类（query响应） | 原样返回 | 四舍五入到定义精度 | 设备实际值，不拒绝 |
| 控制类（ctrl请求） | 拒绝，返回10004 | 四舍五入到定义精度 | 严格校验，防止误操作 |
| 配置类（config_write） | 拒绝，返回10004 | 四舍五入到定义精度 | 严格校验，防止误配置 |

**示例**：
- 上报：FOC实际转速31000rpm（超限）→ 原样返回31000
- 控制：设置目标转速31000rpm（超限）→ 拒绝，返回错误码10004
- 精度：输入50.56Nm → 存储/返回50.6Nm（四舍五入到0.1）

### 3.3 响应结构规范
所有响应(msg_type=response)必须遵循以下规则：

**成功响应**：
```json
{
  "version": "1.0",
  "msg_id": "原请求的msg_id",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "原请求的action",
  "data": {
    "success": true,
    ...其他业务数据
  }
}
```

**失败响应**：
```json
{
  "version": "1.0",
  "msg_id": "原请求的msg_id",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "原请求的action",
  "data": {
    "success": false
  },
  "error": {
    "code": 10001,
    "message": "错误描述"
  }
}
```

**规则**：
- `data.success` 字段必须存在，表示操作是否成功
- 失败时 `error` 对象必须存在，包含 `code` 和 `message`
- 成功时 `error` 对象不存在

### 3.4 错误对象结构
```json
{
  "code": 10001,
  "message": "错误描述"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| code | number | 是 | 错误码，见错误码定义章节 |
| message | string | 是 | 人类可读的错误描述 |

---

## 四、消息类型枚举

### 4.1 msg_type 定义
| 类型 | 值 | 方向 | 说明 |
|------|-----|------|------|
| 请求 | request | 终端→EMS | 终端发起的请求 |
| 响应 | response | EMS→终端 | EMS对请求的响应 |
| 通知 | notify | EMS→终端 | EMS主动推送 |
| 心跳 | heartbeat | 双向 | 连接保活 |

### 4.2 action 分类
| 分类 | action前缀 | 说明 |
|------|-----------|------|
| 系统 | sys_ | 系统级操作 |
| 状态查询 | query_ | 查询设备状态 |
| 控制指令 | ctrl_ | 控制设备 |
| 参数配置 | config_ | 读写配置 |
| 告警 | alarm_ | 告警相关 |
| 数据订阅 | sub_ | 数据订阅 |

### 4.3 device_id 规范
所有设备ID统一使用**字符串**类型，格式为 `{设备类型}{编号}`：

| 设备类型 | 前缀 | 编号范围 | 示例 |
|----------|------|----------|------|
| FOC电机 | FOC | 001-005 | "FOC001", "FOC002" |
| BMS电池 | BMS | 001-005 | "BMS001", "BMS002" |
| 充电桩 | CP | 001-005 | "CP001", "CP002" |

**规则**：
- 编号固定3位，不足补零
- 最大支持5个同类设备
- 设备ID在系统内唯一

---

## 五、系统消息

### 5.1 心跳 (sys_heartbeat)

**心跳消息规则**：
- 心跳使用`msg_type: "heartbeat"`，不使用`data.success`字段
- 心跳响应的`msg_id`必须等于请求的`msg_id`
- 心跳响应可省略`action`字段（兼容简化实现）

#### 终端→EMS 心跳请求
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440000",
  "msg_type": "heartbeat",
  "timestamp": 1703750400000,
  "action": "sys_heartbeat",
  "data": {}
}
```

#### EMS→终端 心跳响应
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440000",
  "msg_type": "heartbeat",
  "timestamp": 1703750400100,
  "action": "sys_heartbeat",
  "data": {
    "server_time": 1703750400100,
    "uptime": 3600
  }
}
```

### 5.2 登录认证 (sys_login)

**密码传输规则**：
- 算法：SHA-256
- 编码：十六进制小写字符串（64字符）
- 输入：`password = SHA256(明文密码)`
- 示例：明文`admin123` → `240be518fabd2724ddb6f04eeb1da5967448d7e831c08c8fa822809f74c720a9`

**防重放说明**：
- 当前版本不实现防重放（内网环境）
- 生产环境建议：启用TLS或添加nonce/timestamp校验

#### 请求
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440001",
  "msg_type": "request",
  "timestamp": 1703750400000,
  "action": "sys_login",
  "data": {
    "username": "admin",
    "password": "240be518fabd2724ddb6f04eeb1da5967448d7e831c08c8fa822809f74c720a9",
    "client_type": "pc"
  }
}
```

#### 响应（成功）
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440001",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "sys_login",
  "data": {
    "success": true,
    "token": "session_token",
    "role": "admin",
    "expires_in": 1800
  }
}
```

#### 响应（失败）
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440001",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "sys_login",
  "data": {
    "success": false
  },
  "error": {
    "code": 10001,
    "message": "用户名或密码错误"
  }
}
```

### 5.3 登出 (sys_logout)

#### 请求
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440002",
  "msg_type": "request",
  "timestamp": 1703750400000,
  "token": "session_token",
  "action": "sys_logout",
  "data": {}
}
```

#### 响应
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440002",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "sys_logout",
  "data": {
    "success": true
  }
}
```

---

## 六、状态查询消息

### 6.1 系统状态查询 (query_system_status)

#### 请求
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440010",
  "msg_type": "request",
  "timestamp": 1703750400000,
  "token": "session_token",
  "action": "query_system_status",
  "data": {}
}
```

#### 响应
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440010",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "query_system_status",
  "data": {
    "success": true,
    "ems_status": "running",
    "uptime": 3600,
    "cpu_usage": 25.5,
    "memory_usage": 45.2,
    "devices": {
      "foc": {"connected": true, "count": 1},
      "bms": {"connected": true, "count": 1},
      "charger": {"connected": true, "count": 1}
    },
    "active_alarms": 0
  }
}
```

**字段说明**：
| 字段 | 类型 | 单位 | 说明 |
|------|------|------|------|
| ems_status | string | - | 系统状态：running/stopped/error |
| uptime | number | 秒 | 系统运行时间 |
| cpu_usage | number | % | CPU使用率，0.0-100.0 |
| memory_usage | number | % | 内存使用率，0.0-100.0 |

### 6.2 FOC状态查询 (query_foc_status)

#### 请求
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440011",
  "msg_type": "request",
  "timestamp": 1703750400000,
  "token": "session_token",
  "action": "query_foc_status",
  "data": {
    "device_id": "FOC001"
  }
}
```

#### 响应
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440011",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "query_foc_status",
  "data": {
    "success": true,
    "device_id": "FOC001",
    "run_status": 1,
    "control_mode": 1,
    "actual_speed": 3000,
    "actual_torque": 50.5,
    "bus_voltage": 48.0,
    "bus_current": 25.5,
    "input_power": 1.2,
    "motor_temp": 45.5,
    "controller_temp": 38.2,
    "fault_code": 0
  }
}
```

**字段说明**：
| 字段 | 类型 | 单位 | 说明 |
|------|------|------|------|
| run_status | number | - | 0=停止 1=运行 2=故障 3=待机 |
| control_mode | number | - | 0=停止 1=速度 2=扭矩 3=位置 |
| actual_speed | number | rpm | 实际转速，-30000~30000 |
| actual_torque | number | Nm | 实际扭矩，精度0.1 |
| bus_voltage | number | V | 母线电压，精度0.1 |
| bus_current | number | A | 母线电流，精度0.1 |
| input_power | number | kW | 输入功率，精度0.1 |
| motor_temp | number | °C | 电机温度，精度0.1 |
| controller_temp | number | °C | 控制器温度，精度0.1 |
| fault_code | number | - | 故障码位域，0表示无故障 |

#### 响应（失败示例）
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440011",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "query_foc_status",
  "data": {
    "success": false
  },
  "error": {
    "code": 20001,
    "message": "设备不存在"
  }
}
```

### 6.3 BMS状态查询 (query_bms_status)

#### 请求
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440012",
  "msg_type": "request",
  "timestamp": 1703750400000,
  "token": "session_token",
  "action": "query_bms_status",
  "data": {
    "device_id": "BMS001"
  }
}
```

#### 响应
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440012",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "query_bms_status",
  "data": {
    "success": true,
    "device_id": "BMS001",
    "total_voltage": 450.5,
    "total_current": 100.2,
    "soc": 85,
    "soh": 98,
    "max_cell_voltage": 3.65,
    "min_cell_voltage": 3.60,
    "max_temp": 35.5,
    "min_temp": 32.0,
    "charge_status": 1,
    "discharge_status": 0,
    "fault_flags": 0,
    "dcl_power": 60.0,
    "ccl_power": 50.0
  }
}
```

**字段说明**：
| 字段 | 类型 | 单位 | 说明 |
|------|------|------|------|
| total_voltage | number | V | 总电压，精度0.1 |
| total_current | number | A | 总电流，充电为正，精度0.1 |
| soc | number | % | 荷电状态，0-100 |
| soh | number | % | 健康状态，0-100 |
| max_cell_voltage | number | V | 最高单体电压，精度0.001 |
| min_cell_voltage | number | V | 最低单体电压，精度0.001 |
| max_temp | number | °C | 最高温度，精度0.1 |
| min_temp | number | °C | 最低温度，精度0.1 |
| charge_status | number | - | 0=未充电 1=充电中 |
| discharge_status | number | - | 0=未放电 1=放电中 |
| fault_flags | number | - | 故障标志位域 |
| dcl_power | number | kW | 最大充电功率限制 |
| ccl_power | number | kW | 最大放电功率限制 |

### 6.4 充电桩状态查询 (query_charger_status)

#### 请求
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440013",
  "msg_type": "request",
  "timestamp": 1703750400000,
  "token": "session_token",
  "action": "query_charger_status",
  "data": {
    "device_id": "CP001"
  }
}
```

#### 响应
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440013",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "query_charger_status",
  "data": {
    "success": true,
    "device_id": "CP001",
    "status": "Charging",
    "connector_id": 1,
    "voltage": 450.5,
    "current": 100.2,
    "power": 45.2,
    "energy": 15.8,
    "soc": 85,
    "duration": 3600,
    "error_code": "NoError"
  }
}
```

**字段说明**：
| 字段 | 类型 | 单位 | 说明 |
|------|------|------|------|
| status | string | - | 状态枚举，见下表 |
| connector_id | number | - | 连接器编号，1-N |
| voltage | number | V | 输出电压，精度0.1 |
| current | number | A | 输出电流，精度0.1 |
| power | number | kW | 输出功率，精度0.1 |
| energy | number | kWh | 本次充电电量，精度0.1 |
| soc | number | % | 电池SOC，0-100 |
| duration | number | 秒 | 本次充电时长 |
| error_code | string | - | 错误码枚举，见下表 |

**status枚举**：
| 值 | 说明 |
|-----|------|
| Available | 可用 |
| Preparing | 准备中 |
| Charging | 充电中 |
| SuspendedEVSE | 桩端暂停 |
| SuspendedEV | 车端暂停 |
| Finishing | 结束中 |
| Reserved | 已预约 |
| Unavailable | 不可用 |
| Faulted | 故障 |

**error_code枚举**：
| 值 | 说明 |
|-----|------|
| NoError | 无错误 |
| ConnectorLockFailure | 电子锁故障 |
| EVCommunicationError | 车辆通讯故障 |
| GroundFailure | 接地故障 |
| HighTemperature | 高温故障 |
| InternalError | 内部错误 |
| OverCurrentFailure | 过流故障 |
| OverVoltage | 过压故障 |
| UnderVoltage | 欠压故障 |
| PowerMeterFailure | 电表故障 |
| OtherError | 其他错误 |

---

## 七、控制指令消息

### 7.1 FOC电机控制 (ctrl_foc_motor)

#### 请求
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440020",
  "msg_type": "request",
  "timestamp": 1703750400000,
  "token": "session_token",
  "action": "ctrl_foc_motor",
  "data": {
    "device_id": "FOC001",
    "command": "start",
    "mode": 1,
    "target_speed": 3000,
    "target_torque": 0
  }
}
```

**command枚举**：
| 值 | 说明 |
|-----|------|
| start | 启动电机 |
| stop | 停止电机 |
| emergency_stop | 紧急停止 |
| set_speed | 设置目标转速 |
| set_torque | 设置目标扭矩 |

**mode枚举**：
| 值 | 说明 |
|-----|------|
| 0 | 停止 |
| 1 | 速度模式 |
| 2 | 扭矩模式 |
| 3 | 位置模式 |

**参数说明**：
| 字段 | 类型 | 单位 | 说明 |
|------|------|------|------|
| target_speed | number | rpm | 目标转速，-30000~30000 |
| target_torque | number | Nm | 目标扭矩，精度0.1 |

#### 响应（成功）
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440020",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "ctrl_foc_motor",
  "data": {
    "success": true,
    "device_id": "FOC001"
  }
}
```

#### 响应（失败示例 - 设备故障）
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440020",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "ctrl_foc_motor",
  "data": {
    "success": false
  },
  "error": {
    "code": 20004,
    "message": "设备故障，无法执行控制指令"
  }
}
```

#### 响应（失败示例 - 参数越界）
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440021",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "ctrl_foc_motor",
  "data": {
    "success": false
  },
  "error": {
    "code": 10004,
    "message": "参数错误：target_speed超出有效范围(-30000~30000)"
  }
}
```

### 7.2 BMS控制 (ctrl_bms)

#### 请求
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440022",
  "msg_type": "request",
  "timestamp": 1703750400000,
  "token": "session_token",
  "action": "ctrl_bms",
  "data": {
    "device_id": "BMS001",
    "command": "enable_charge",
    "value": 1
  }
}
```

**command枚举**：
| 值 | 说明 |
|-----|------|
| enable_charge | 充电使能 |
| enable_discharge | 放电使能 |
| emergency_stop | 紧急停止 |
| fault_reset | 故障复位 |
| start_balance | 启动均衡 |

#### 响应
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440022",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "ctrl_bms",
  "data": {
    "success": true,
    "device_id": "BMS001"
  }
}
```

#### 响应（失败示例 - 设备离线）
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440022",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "ctrl_bms",
  "data": {
    "success": false
  },
  "error": {
    "code": 20002,
    "message": "设备离线"
  }
}
```

### 7.3 充电桩控制 (ctrl_charger)

#### 请求
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440023",
  "msg_type": "request",
  "timestamp": 1703750400000,
  "token": "session_token",
  "action": "ctrl_charger",
  "data": {
    "device_id": "CP001",
    "command": "start",
    "connector_id": 1,
    "id_tag": "user001"
  }
}
```

**command枚举**：
| 值 | 说明 |
|-----|------|
| start | 启动充电 |
| stop | 停止充电 |
| unlock | 解锁充电枪 |
| reset | 重启充电桩 |

#### 响应
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440023",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "ctrl_charger",
  "data": {
    "success": true,
    "device_id": "CP001"
  }
}
```

#### 响应（失败示例 - 操作被拒绝）
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440023",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "ctrl_charger",
  "data": {
    "success": false
  },
  "error": {
    "code": 20005,
    "message": "操作被拒绝：充电桩当前状态不允许启动充电"
  }
}
```

---

## 八、参数配置消息

### 8.1 读取配置 (config_read)

#### 请求
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440030",
  "msg_type": "request",
  "timestamp": 1703750400000,
  "token": "session_token",
  "action": "config_read",
  "data": {
    "category": "foc",
    "device_id": "FOC001",
    "keys": ["max_speed", "max_torque", "accel_time"]
  }
}
```

**category枚举**：
| 值 | 说明 |
|-----|------|
| system | 系统配置 |
| foc | FOC配置 |
| bms | BMS配置 |
| charger | 充电桩配置 |

#### 响应（成功）
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440030",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "config_read",
  "data": {
    "success": true,
    "category": "foc",
    "device_id": "FOC001",
    "values": {
      "max_speed": 3000,
      "max_torque": 100.0,
      "accel_time": 2000
    }
  }
}
```

**字段说明**：
| 字段 | 类型 | 单位 | 说明 |
|------|------|------|------|
| max_speed | number | rpm | 最大转速限制 |
| max_torque | number | Nm | 最大扭矩限制，精度0.1 |
| accel_time | number | ms | 加速时间 |

### 8.2 写入配置 (config_write)

#### 请求
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440031",
  "msg_type": "request",
  "timestamp": 1703750400000,
  "token": "session_token",
  "action": "config_write",
  "data": {
    "category": "foc",
    "device_id": "FOC001",
    "values": {
      "max_speed": 2500,
      "accel_time": 3000
    }
  }
}
```

#### 响应（成功）
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440031",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "config_write",
  "data": {
    "success": true,
    "applied": ["max_speed", "accel_time"],
    "need_restart": false
  }
}
```

#### 响应（失败 - 参数越界）
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440031",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "config_write",
  "data": {
    "success": false
  },
  "error": {
    "code": 10004,
    "message": "参数错误：max_speed超出有效范围(0~30000)"
  }
}
```

---

## 九、告警消息

### 9.1 告警推送 (alarm_notify)

EMS主动推送告警到终端：

```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440040",
  "msg_type": "notify",
  "timestamp": 1703750400000,
  "action": "alarm_notify",
  "data": {
    "alarm_id": "ALM-20231228-001",
    "source": "bms",
    "device_id": "BMS001",
    "level": "L3",
    "type": "over_temp",
    "code": 1017,
    "message": "充电高温告警",
    "value": 48.5,
    "threshold": 45.0,
    "occur_time": 1703750400000
  }
}
```

**字段说明**：
| 字段 | 类型 | 单位 | 说明 |
|------|------|------|------|
| alarm_id | string | - | 告警唯一标识 |
| source | string | - | 告警来源：foc/bms/charger/system |
| device_id | string | - | 设备ID |
| level | string | - | 告警级别，见下表 |
| type | string | - | 告警类型 |
| code | number | - | 告警码 |
| message | string | - | 告警描述 |
| value | number | - | 触发告警的实际值 |
| threshold | number | - | 告警阈值 |
| occur_time | number | ms | 告警发生时间，毫秒级Unix时间戳 |

**level枚举**：
| 值 | 说明 |
|-----|------|
| L1 | 警告 |
| L2 | 告警 |
| L3 | 故障 |
| L4 | 严重故障 |

### 9.2 告警查询 (alarm_query)

#### 请求
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440041",
  "msg_type": "request",
  "timestamp": 1703750400000,
  "token": "session_token",
  "action": "alarm_query",
  "data": {
    "active_only": true,
    "source": null,
    "level": null,
    "limit": 100
  }
}
```

#### 响应（成功）
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440041",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "alarm_query",
  "data": {
    "success": true,
    "total": 2,
    "alarms": [
      {
        "alarm_id": "ALM-20231228-001",
        "source": "bms",
        "device_id": "BMS001",
        "level": "L3",
        "type": "over_temp",
        "code": 1017,
        "message": "充电高温告警",
        "occur_time": 1703750400000,
        "acknowledged": false
      }
    ]
  }
}
```

### 9.3 告警确认 (alarm_ack)

#### 请求
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440042",
  "msg_type": "request",
  "timestamp": 1703750400000,
  "token": "session_token",
  "action": "alarm_ack",
  "data": {
    "alarm_id": "ALM-20231228-001"
  }
}
```

#### 响应（成功）
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440042",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "alarm_ack",
  "data": {
    "success": true,
    "alarm_id": "ALM-20231228-001"
  }
}
```

---

## 十、数据订阅消息

### 10.1 订阅实时数据 (sub_realtime)

#### 请求
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440050",
  "msg_type": "request",
  "timestamp": 1703750400000,
  "token": "session_token",
  "action": "sub_realtime",
  "data": {
    "subscribe": true,
    "topics": ["foc_status", "bms_status", "charger_status"],
    "interval": 1000
  }
}
```

**字段说明**：
| 字段 | 类型 | 单位 | 说明 |
|------|------|------|------|
| subscribe | boolean | - | true=订阅，false=取消订阅 |
| topics | array | - | 订阅主题列表，见下表 |
| interval | number | ms | 推送间隔，毫秒，范围100~10000 |

**topics枚举**：
| 值 | 说明 |
|-----|------|
| foc_status | FOC状态 |
| bms_status | BMS状态 |
| charger_status | 充电桩状态 |
| system_status | 系统状态 |
| alarms | 告警 |

#### 响应（成功）
```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440050",
  "msg_type": "response",
  "timestamp": 1703750400100,
  "action": "sub_realtime",
  "data": {
    "success": true,
    "subscribed": ["foc_status", "bms_status", "charger_status"]
  }
}
```

### 10.2 实时数据推送 (sub_data_push)

订阅后EMS周期性推送：

```json
{
  "version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440051",
  "msg_type": "notify",
  "timestamp": 1703750400000,
  "action": "sub_data_push",
  "data": {
    "topic": "foc_status",
    "device_id": "FOC001",
    "values": {
      "run_status": 1,
      "actual_speed": 3000,
      "actual_torque": 50.5,
      "bus_voltage": 48.0,
      "bus_current": 25.5
    }
  }
}
```

### 10.3 订阅推送交付语义

**交付保证**：
- 推送采用"尽力交付"（best-effort），不保证100%送达
- 网络拥塞或终端处理慢时，EMS可能跳过中间数据，只推送最新状态

**断线处理**：
- 连接断开后，订阅自动失效
- 重连后需重新发送`sub_realtime`请求恢复订阅
- 断线期间的数据不补发

**重订阅行为**：
- 重复订阅同一topic：更新interval，不产生重复推送
- 订阅新topic：追加到已有订阅列表

---

## 十一、通讯流程

### 11.1 连接建立流程
```
1. 终端建立TCP连接到EMS:9000
2. 终端发送sys_login请求
3. EMS验证并返回token
4. 终端开始心跳周期
5. 终端可发送业务请求
```

### 11.2 心跳保活流程
```
每10秒:
  终端 → EMS: heartbeat请求
  EMS → 终端: heartbeat响应

超时处理:
  30秒无心跳 → EMS断开连接
  终端检测断开 → 自动重连
```

### 11.3 数据订阅流程
```
1. 终端发送sub_realtime请求
2. EMS确认订阅
3. EMS按interval周期推送数据
4. 终端取消订阅(subscribe=false)或断开连接
```

---

## 十二、错误码定义

### 12.1 系统错误码 (10000-10999)
| 错误码 | 说明 |
|--------|------|
| 10001 | 用户名或密码错误 |
| 10002 | 会话已过期 |
| 10003 | 权限不足 |
| 10004 | 请求参数错误 |
| 10005 | 服务器内部错误 |
| 10006 | 无效的token |
| 10007 | 连接数已达上限 |

### 12.2 设备错误码 (20000-29999)
| 错误码 | 说明 |
|--------|------|
| 20001 | 设备不存在 |
| 20002 | 设备离线 |
| 20003 | 设备忙 |
| 20004 | 设备故障 |
| 20005 | 操作被拒绝 |

### 12.3 通讯错误码 (30000-39999)
| 错误码 | 说明 |
|--------|------|
| 30001 | 通讯超时 |
| 30002 | 数据校验失败 |
| 30003 | 协议版本不匹配 |

---

## 十三、枚举命名规范

### 13.1 命名策略
| 类型 | 风格 | 示例 |
|------|------|------|
| action | snake_case | `sys_login`, `query_foc_status` |
| msg_type | 小写 | `request`, `response`, `notify` |
| 充电桩status | PascalCase | `Available`, `Charging`（兼容OCPP） |
| 充电桩error_code | PascalCase | `NoError`, `OverVoltage`（兼容OCPP） |
| 其他枚举 | snake_case | `over_temp`, `enable_charge` |

### 13.2 OCPP兼容说明
充电桩相关字段（status、error_code）采用PascalCase以兼容OCPP 1.6规范，其他字段统一使用snake_case。

---

## 十四、安全部署建议

### 14.1 部署假设
- 本协议设计用于**内网环境**（局域网WiFi）
- EMS与终端在同一受信任网络内
- 不直接暴露于公网

### 14.2 安全建议
| 场景 | 建议措施 |
|------|----------|
| 内网部署 | 使用防火墙隔离，限制端口访问 |
| 跨网段访问 | 建议使用VPN隧道 |
| 公网暴露（不推荐） | 必须启用TLS加密 |
| 密码传输 | 当前使用hash，生产环境建议TLS |

---

## 十五、消息追踪与排障

### 15.1 msg_id生成规则
- 格式：UUID v4（如`550e8400-e29b-41d4-a716-446655440000`）
- 生成方：请求由终端生成，通知由EMS生成
- 唯一性：同一连接内不重复

### 15.2 可选扩展字段（预留）
| 字段 | 用途 | 说明 |
|------|------|------|
| trace_id | 跨系统追踪 | 可选，用于关联多个请求 |
| client_id | 终端标识 | 可选，用于区分多终端 |

### 15.3 日志关联建议
```
推荐日志格式：
[时间戳] [msg_id] [action] [方向] [结果]

示例：
[2025-12-28 10:00:00.123] [550e8400-...] [query_foc_status] [REQ] 收到请求
[2025-12-28 10:00:00.156] [550e8400-...] [query_foc_status] [RSP] 成功
```

---

**文档版本**: v1.0
**编制日期**: 2025-12-28
**协议类型**: WiFi JSON
**通信接口**: TCP/IP
