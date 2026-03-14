# API 编码规范（Apifox 同步）

编写 API 代码时必须遵循以下规则，确保同步到 Apifox 后文档完整可用。

## Schema 字段

- 每个字段 **必须** 有 `description`
- 每个字段 **应该** 有 `examples`（使用虚构数据，禁止真实 Token / 邮箱 / IP）
- 可选字段用 `Optional[T] = Field(None, ...)`
- 添加校验规则：`ge`、`le`、`min_length`、`max_length` 等
- 枚举用 `class Status(str, Enum)`
- 命名：`{Entity}Create` / `{Entity}Update` / `{Entity}Response` / `{Entity}ListResponse`
- 列表响应统一 `{ items: [...], total: N }` 结构

## 路由

- **必须** 有 `summary`（接口名称，≤15 字）和 `description`（完整说明）
- **必须** 有 `tags`（在 Router 级别声明）
- **必须** 有 `response_model`
- 创建接口用 `status_code=201`
- 所有 Query / Path 参数 **必须** 有 `description`
- 废弃接口加 `deprecated=True`

## 错误响应

在 views 顶部定义公共错误响应，所有路由复用：

```python
RESP_401 = {401: {"model": ErrorResponse, "description": "未登录或令牌无效"}}
RESP_403 = {403: {"model": ErrorResponse, "description": "无权限执行此操作"}}
RESP_404 = {404: {"model": ErrorResponse, "description": "资源不存在"}}
RESP_409 = {409: {"model": ErrorResponse, "description": "资源冲突（如重复创建）"}}
RESP_422 = {422: {"model": ValidationErrorResponse, "description": "请求参数校验失败"}}
```

按需引用：`responses={**RESP_401, **RESP_404}`

速查：列表查询→401 | 详情→401+404 | 创建→401+409+422 | 更新→401+403+404+422 | 删除→401+403+404 | 状态变更→401+403+404+409

## 通用模型

每个项目需定义以下通用模型：

```python
class MessageResponse(BaseModel):
    message: str = Field(..., description="提示信息", examples=["操作成功"])

class ErrorResponse(BaseModel):
    detail: str = Field(..., description="错误信息", examples=["资源不存在"])

class ValidationErrorItem(BaseModel):
    field: str = Field(..., description="出错字段", examples=["price"])
    message: str = Field(..., description="错误描述", examples=["值必须大于 0"])

class ValidationErrorResponse(BaseModel):
    detail: List[ValidationErrorItem] = Field(..., description="校验错误列表")
```
