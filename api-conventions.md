# API 编码规范（Apifox 同步）

编写 API 代码时必须遵循以下规则，确保同步到 Apifox 后文档完整可用。

**语言识别**：AI 须自行分析当前项目的语言和框架（通过项目文件如 `requirements.txt`、`pom.xml`、`go.mod`、`package.json` 等判断），然后使用对应语言的写法来满足以下规则。规则本身是语言无关的。

---

## 1. Schema 字段

- 每个字段 **必须** 有 description
- 每个字段 **应该** 有 example（使用虚构数据，禁止真实 Token / 邮箱 / IP）
- 可选字段必须标记为非必填
- 添加校验规则（范围、长度等）
- 使用枚举约束可选值
- 命名：`{Entity}Create` / `{Entity}Update` / `{Entity}Response` / `{Entity}ListResponse`
- 列表响应统一 `{ items: [...], total: N }` 结构

各语言写法：

| 规则 | Python (Pydantic) | Java (springdoc) | Go (swag) | TypeScript (NestJS) |
|------|-------------------|------------------|-----------|---------------------|
| 字段描述 | `Field(description="...")` | `@Schema(description="...")` | `// @Description ...` 或字段注释 | `@ApiProperty({description: "..."})` |
| 示例值 | `Field(examples=[...])` | `@Schema(example="...")` | `example:"..."` tag | `@ApiProperty({example: "..."})` |
| 非必填 | `Optional[T] = Field(None)` | `@Schema(requiredMode=NOT_REQUIRED)` | `json:"name,omitempty"` | `@ApiPropertyOptional()` |
| 校验 | `Field(ge=0, max_length=100)` | `@Min(0) @Max(100)` | `minimum:"0" maximum:"100"` tag | `@Min(0) @Max(100)` (class-validator) |
| 枚举 | `class Status(str, Enum)` | `enum Status {...}` | `// @Param status query string false "状态" Enums(...)` | `enum Status {...}` |

## 2. 路由

- **必须** 有 summary（接口名称，≤15 字）和 description（完整说明）
- **必须** 有 tags（按模块分组）
- **必须** 声明响应模型
- 创建接口返回 201 状态码
- 所有 Query / Path 参数 **必须** 有 description
- 废弃接口标记 deprecated

各语言写法：

| 规则 | Python (FastAPI) | Java (springdoc) | Go (swag) | TypeScript (NestJS) |
|------|------------------|------------------|-----------|---------------------|
| 接口名称 | `summary="..."` | `@Operation(summary="...")` | `// @Summary ...` | `@ApiOperation({summary: "..."})` |
| 接口说明 | `description="..."` | `@Operation(description="...")` | `// @Description ...` | `@ApiOperation({description: "..."})` |
| 分组 | `tags=["物品管理"]` | `@Tag(name="物品管理")` | `// @Tags 物品管理` | `@ApiTags("物品管理")` |
| 响应模型 | `response_model=Item` | `@ApiResponse(content=@Content(schema=@Schema(impl=Item.class)))` | `// @Success 200 {object} Item` | `@ApiResponse({type: Item})` |
| 状态码 | `status_code=201` | `@ResponseStatus(CREATED)` | `// @Success 201 {object} Item` | `@HttpCode(201)` |
| 参数描述 | `Query(description="...")` | `@Parameter(description="...")` | `// @Param name query string false "描述"` | `@ApiQuery({description: "..."})` |
| 废弃 | `deprecated=True` | `@Deprecated` + `@Operation(deprecated=true)` | `// @Deprecated` | `@ApiOperation({deprecated: true})` |

## 3. 错误响应

每个接口必须声明可能的错误响应。定义公共错误模型，按需引用到各路由：

**标准错误码：**
- `401` — 未登录或令牌无效
- `403` — 无权限执行此操作
- `404` — 资源不存在
- `409` — 资源冲突（如重复创建）
- `422` — 请求参数校验失败

**各操作速查表：**

| 操作 | 401 | 403 | 404 | 409 | 422 |
|------|-----|-----|-----|-----|-----|
| 列表查询 | Y | | | | |
| 获取详情 | Y | | Y | | |
| 创建 | Y | | | Y | Y |
| 更新 | Y | Y | Y | | Y |
| 删除 | Y | Y | Y | | |
| 状态变更 | Y | Y | Y | Y | |

各语言写法：

| 语言 | 错误响应声明方式 |
|------|----------------|
| Python | `responses={401: {"model": ErrorResponse, "description": "..."}}`，用 `RESP_401` 等字典复用 |
| Java | `@ApiResponse(responseCode="401", description="...", content=@Content(schema=@Schema(implementation=ErrorResponse.class)))` |
| Go | `// @Failure 401 {object} ErrorResponse "未登录"` |
| TypeScript | `@ApiResponse({status: 401, type: ErrorResponse, description: "..."})` |

## 4. 通用错误模型

每个项目需定义以下通用模型（使用项目对应语言）：

| 模型 | 字段 | 用途 |
|------|------|------|
| `MessageResponse` | `message: string` | 通用成功提示 |
| `ErrorResponse` | `detail: string` | 业务错误 (401/403/404/409) |
| `ValidationErrorItem` | `field: string, message: string` | 单个校验错误 |
| `ValidationErrorResponse` | `detail: ValidationErrorItem[]` | 校验错误列表 (422) |
