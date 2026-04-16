# app-api 接口定义：用户是否展示 AI 功能模块

## 背景
- 触发时机：用户进入小程序首页时调用，用于判断小程序是否展示 AI 功能入口按钮。
- 接口调用方：App 端（因此对外路径需包含 `/app-api` 前缀；项目会对 `controller.app` 包下的 Controller 自动加该前缀）。
- 核心逻辑：服务端接收用户 id，调用“爱创云后台接口”获取该用户是否开通 AI 功能模块，并返回给前端。

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
- 爱创云接口调用失败：使用业务错误码（建议单独定义，例如 `AI_FEATURE_QUERY_FAILED`）

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
2. 调用爱创云后台接口，入参携带 `userId`
3. 解析爱创云返回，得到 `enabled`（或等价字段）
4. 以 `CommonResult.success(enabled)` 返回给前端

### 1.5 对接“爱创云后台接口”（约定）
由于仓库中暂无现成爱创云的客户端封装，本接口定义阶段先约定对接形态，便于后续落地：
- 调用方式：HTTP（GET/POST 由爱创云接口实际定义决定）
- 入参：userId
- 出参：是否开通/可用（enabled）
- 鉴权：由爱创云侧要求决定（可能为固定 Token / AppKey+签名 / OAuth2 等）

建议配置项（示例命名，落地时可按实际项目配置规范调整）：
- `yudao.ai-chuangyun.base-url`：爱创云网关地址
- `yudao.ai-chuangyun.token`：鉴权 token（如需要）

## 约束与说明
- 本接口按需求明确“前端传入 userId”。如果后续接入登录态（token），建议改为后端从登录态解析用户 id，避免被篡改。

