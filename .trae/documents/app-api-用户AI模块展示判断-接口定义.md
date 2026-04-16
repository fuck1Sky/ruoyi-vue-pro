# app-api 接口定义：AI 能力相关

## 背景
- 触发时机：用户进入小程序首页时调用，用于判断小程序是否展示 AI 功能入口按钮。
- 接口调用方：App 端（因此对外路径需包含 `/app-api` 前缀；项目会对 `controller.app` 包下的 Controller 自动加该前缀）。
- 核心逻辑：服务端接收请求参数，调用外部接口/大模型接口获取结果，并返回给前端。

## 1. 判断当前用户是否展示 AI 功能模块
### 1.1 接口信息
- Method：GET
- Path：/app-api/ai/feature/enabled
- 描述：判断指定用户是否展示 AI 功能模块（小程序入口是否展示）

### 1.2 请求参数
| 参数名 | 位置 | 类型 | 必填 | 说明 |
|---|---|---|---|---|
| userId | query | long | 是 | 前端传入的用户编号 |

示例：
```http
GET /app-api/ai/feature/enabled?userId=123
```

### 1.3 响应
#### 成功响应
- 返回结构：`CommonResult<Boolean>`
- data 含义：`true` 表示展示 AI 功能模块；`false` 表示不展示

示例：
```json
{
  "code": 0,
  "msg": "",
  "data": true
}
```

#### 失败响应（示例约定）
- 参数缺失/非法：使用系统通用参数校验错误码
- 立祥接口调用失败：使用业务错误码（建议单独定义，例如 `AI_FEATURE_QUERY_FAILED`）

示例：
```json
{
  "code": 500,
  "msg": "AI 功能状态查询失败",
  "data": null
}
```

### 1.4 服务端内部逻辑（约定）
1. 校验 `userId` 非空且大于 0
2. 调用立祥接口，入参携带 `userId`
3. 解析立祥返回，得到 `enabled`（或等价字段）
4. 以 `CommonResult.success(enabled)` 返回给前端

### 1.5 对接“立祥接口”（约定）
由于仓库中暂无现成“立祥接口”的客户端封装，本接口定义阶段先约定对接形态，便于后续落地：
- 调用方式：HTTP（GET/POST 由立祥接口实际定义决定）
- 入参：userId
- 出参：是否开通/可用（enabled）
- 鉴权：由立祥侧要求决定（可能为固定 Token / AppKey+签名 / OAuth2 等）

建议配置项（示例命名，落地时可按实际项目配置规范调整）：
- `yudao.lixiang.base-url`：立祥接口网关地址
- `yudao.lixiang.token`：鉴权 token（如需要）

## 约束与说明
- 本接口按需求明确“前端传入 userId”。如果后续接入登录态（token），建议改为后端从登录态解析用户 id，避免被篡改。

## 2. 获取当前用户基本信息（透传给大模型）
### 2.1 接口信息
- Method：POST
- Path：/app-api/ai/user/basic-info
- 描述：用户点击 AI 图标时，上报用户基本信息给大模型端

### 2.2 请求体
- Content-Type：application/json

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| uid | long | 是 | 用户编号 |
| userName | string | 是 | 用户名称 |
| vendorName | string | 是 | 厂商名称 |

示例：
```json
{
  "uid": 123,
  "userName": "张三",
  "vendorName": "XX 厂商"
}
```

### 2.3 响应
- 返回结构：`CommonResult<Boolean>`
- data 含义：`true` 表示已接收并处理完成

示例：
```json
{
  "code": 0,
  "msg": "",
  "data": true
}
```

### 2.4 服务端内部逻辑（约定）
1. 校验入参字段完整性
2. 将 `uid/userName/vendorName` 透传给大模型接口
3. 不关心大模型返回内容，仅根据调用是否成功返回 `true/false`

## 3. 首页首次进入：拉取模型端首页数据并推送给前端
### 3.1 接口信息
- Method：GET
- Path：/app-api/ai/home/first-enter
- 描述：用户点击 AI 图标进入 AI 首页时，拉取模型端首页回参并返回给前端

### 3.2 请求参数
- 无

### 3.3 响应
- 返回结构：`CommonResult<Object>`
- data 含义：模型端首页回参（原样透传，结构由模型端定义）

示例：
```json
{
  "code": 0,
  "msg": "",
  "data": {
    "welcomeText": "你好",
    "cards": []
  }
}
```

### 3.4 服务端内部逻辑（约定）
1. 调用模型端接口
2. 获取模型端首页回参
3. 原样返回给前端

## 4. ASR：语音转文字
### 4.1 接口信息
- Method：POST
- Path：/app-api/ai/asr
- 描述：前端传入 COS 音频路径，服务端调用大模型 ASR 接口并返回识别结果

### 4.2 请求体
- Content-Type：application/json

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| cosPath | string | 是 | COS 文件路径/URL |

示例：
```json
{
  "cosPath": "cos://bucket/path/audio.wav"
}
```

### 4.3 响应
- 返回结构：`CommonResult<Object>`
- data 含义：大模型 ASR 结果（原样透传，结构由模型端定义）

示例：
```json
{
  "code": 0,
  "msg": "",
  "data": {
    "text": "今天的天气不错",
    "language": "zh-CN"
  }
}
```

### 4.4 服务端内部逻辑（约定）
1. 校验 `cosPath` 非空
2. 调用大模型 ASR 接口
3. 原样返回识别结果

## 5. TTS（切片 WebSocket）
### 5.1 接口信息
- Protocol：WebSocket
- Path：/app-api/ai/tts/ws
- 描述：文本转语音（切片）通道，服务端与前端通过 WebSocket 双向通讯

### 5.2 入参
- 入参：无（仅建立 WebSocket 连接）

### 5.3 服务端 -> 客户端消息（建议）
| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| sessionId | string | 是 | 会话编号 |
| seq | int | 是 | 音频分片序号，从 1 开始 |
| audioBase64 | string | 是 | 音频分片 base64 |
| isLast | boolean | 是 | 是否最后一片 |
| errorMsg | string | 否 | 错误信息 |

## 6. chat-Stream（流式输出）
### 6.1 接口信息
- Method：POST
- Path：/app-api/ai/chat/stream
- 描述：流式对话，服务端调用大模型接口并将结果流式返回给前端

### 6.2 请求体
- Content-Type：application/json

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| userId | long | 是 | 用户编号 |
| sessionId | string | 是 | 会话编号 |
| message | string | 是 | 用户输入消息 |

示例：
```json
{
  "userId": 123,
  "sessionId": "s-001",
  "message": "帮我推荐一个活动"
}
```

### 6.3 响应（流式）
- Content-Type：text/event-stream（SSE）
- 每条事件 data：模型端输出的增量文本片段（或 JSON 片段，按模型端约定）

示例：
```text
data: 你好

data: ，我来为你推荐

data: [DONE]
```

### 6.4 服务端内部逻辑（约定）
1. 校验入参字段完整性
2. 调用大模型 Stream 接口
3. 将大模型增量输出转发给前端

## 7. 会话中断
### 7.1 接口信息
- Method：POST
- Path：/app-api/ai/chat/interrupt
- 描述：中断指定会话的流式输出

### 7.2 请求体
- Content-Type：application/json

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| userId | long | 是 | 用户编号 |
| sessionId | string | 是 | 会话编号 |
| interrupt | boolean | 是 | 中断标识（true 表示中断） |

示例：
```json
{
  "userId": 123,
  "sessionId": "s-001",
  "interrupt": true
}
```

### 7.3 响应
- 返回结构：`CommonResult<Boolean>`

示例：
```json
{
  "code": 0,
  "msg": "",
  "data": true
}
```

### 7.4 服务端内部逻辑（约定）
1. 校验入参
2. 调用大模型中断接口，按 `userId + sessionId` 定位并中断
3. 返回中断是否成功

## 8. 大模型端调用：更新用户活动“规则/总结”内容
### 8.1 接口信息
- Method：POST
- Path：/app-api/ai/activity/rule/update
- 描述：大模型端回调更新用户活动关联数据（写入 `manage_user_activity_rule_rel`）

### 8.2 请求体
- Content-Type：application/json

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| userId | long | 是 | 用户编号 |
| activityId | long | 是 | 活动编号 |
| activityType | string | 是 | 活动类型 |
| activityRule | string | 是 | 活动规则/总结内容（落库到 `ai_summary`） |

示例：
```json
{
  "userId": 123,
  "activityId": 456,
  "activityType": "promotion_seckill",
  "activityRule": "该活动适合新用户，规则如下……"
}
```

### 8.3 响应
- 返回结构：`CommonResult<Boolean>`

### 8.4 服务端内部逻辑（约定）
1. 以 `userId + activityType + activityId` 查找或创建关联记录
2. 更新 `manage_user_activity_rule_rel.ai_summary = activityRule`
3. 返回成功与否

## 9. 获得活动相关数据（活动类型与通用图片的关联表、活动规则/总结）
### 9.1 接口信息
- Method：GET
- Path：/app-api/ai/activity/data
- 描述：对话中推荐活动时，查询活动图片与用户活动总结内容并返回给前端

### 9.2 请求参数
| 参数名 | 位置 | 类型 | 必填 | 说明 |
|---|---|---|---|---|
| activityId | query | long | 是 | 活动编号 |
| activityType | query | string | 是 | 活动类型 |
| userId | query | long | 是 | 用户编号 |

示例：
```http
GET /app-api/ai/activity/data?activityId=456&activityType=promotion_seckill&userId=123
```

### 9.3 响应
- 返回结构：`CommonResult<Object>`
- data 建议字段：
  - `activityId`：活动编号
  - `activityType`：活动类型
  - `imageUrl`：通用图片 URL（来自 `manage_activity_type_image_rel`）
  - `aiSummary`：活动规则/总结内容（来自 `manage_user_activity_rule_rel.ai_summary`，可能为空）

示例：
```json
{
  "code": 0,
  "msg": "",
  "data": {
    "activityId": 456,
    "activityType": "promotion_seckill",
    "imageUrl": "https://example.com/a.png",
    "aiSummary": "该活动适合新用户，规则如下……"
  }
}
```

### 9.4 服务端内部逻辑（约定）
1. 查询 `manage_activity_type_image_rel` 获取 `image_url`
2. 查询 `manage_user_activity_rule_rel` 获取 `ai_summary`
3. 组装返回

## 10. 数字人流式对话
### 10.1 接口信息
- Method：POST
- Path：/app-api/ai/avatar/chat/stream
- 描述：数字人流式对话，服务端调用立祥接口并将结果流式返回给前端

### 10.2 请求体
- Content-Type：application/json

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| userId | long | 是 | 用户编号 |

示例：
```json
{
  "userId": 123
}
```

### 10.3 响应（流式）
- Content-Type：text/event-stream（SSE）
- 每条事件 data：立祥接口返回的增量内容（按立祥约定）

### 10.4 服务端内部逻辑（约定）
1. 校验 `userId`
2. 调用立祥接口（流式）
3. 将立祥接口的增量输出转发给前端
