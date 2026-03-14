# API 代码编写规范

本规范定义了编写 API 代码时必须遵循的约定，确保自动同步到 Apifox 后文档完整、准确、可用。

**核心原则：代码是 API 文档的唯一事实来源。** 所有描述、示例、校验规则都在代码中维护，不在 Apifox 中手动编辑。

---

## 1. 数据模型（Schema）

### 1.1 每个字段必须有 description

```python
# 正确
name: str = Field(..., description="物品名称，不超过 100 个字符")

# 错误 — Apifox 中该字段备注为空
name: str
name: str = Field(...)
```

### 1.2 每个字段应有 examples

```python
# 正确 — Apifox 中自动填充示例值
price: float = Field(..., description="价格，单位：元", examples=[9.99])
email: str = Field(..., description="邮箱地址", examples=["zhangsan@example.com"])

# 错误 — Apifox 示例为空，Mock 数据不可读
price: float = Field(..., description="价格")
```

**注意：examples 必须使用虚构数据，禁止使用真实用户信息、真实 Token、内网 IP。**

### 1.3 添加校验规则

```python
# 数值范围
price: float = Field(..., ge=0, description="价格", examples=[9.99])
page: int = Field(1, ge=1, description="页码")
page_size: int = Field(20, ge=1, le=100, description="每页数量")

# 字符串长度
name: str = Field(..., min_length=1, max_length=100, description="物品名称")
password: str = Field(..., min_length=8, description="密码，至少 8 位")
```

### 1.4 可选字段用 Optional + None 默认值

```python
# 正确 — Apifox 中标记为"非必填"
description: Optional[str] = Field(None, description="物品描述")

# 错误 — 该字段会被标记为"必填"
description: str = Field("", description="物品描述")
```

### 1.5 枚举类型用 Enum

```python
class OrderStatus(str, Enum):
    pending = "pending"
    paid = "paid"
    shipped = "shipped"
    cancelled = "cancelled"

# Apifox 中自动显示可选值列表
status: OrderStatus = Field(..., description="订单状态")
```

### 1.6 模型命名约定

| 用途 | 命名格式 | 示例 |
|------|---------|------|
| 创建请求体 | `{Entity}Create` | `ItemCreate`, `UserCreate` |
| 更新请求体 | `{Entity}Update` | `ItemUpdate`, `UserUpdate` |
| 单条响应 | `{Entity}Response` | `ItemResponse`, `UserResponse` |
| 列表响应 | `{Entity}ListResponse` | `ItemListResponse` |
| 通用成功提示 | `MessageResponse` | `{"message": "操作成功"}` |
| 通用错误 | `ErrorResponse` | `{"detail": "资源不存在"}` |
| 校验错误 | `ValidationErrorResponse` | `{"detail": [{"field": "...", "message": "..."}]}` |

### 1.7 列表响应统一结构

```python
class ItemListResponse(BaseModel):
    items: List[ItemResponse] = Field(..., description="物品列表")
    total: int = Field(..., description="总数量", examples=[100])
```

不要返回裸数组，统一包装为 `{ items: [...], total: N }`，方便前端分页。

---

## 2. 路由（Router）

### 2.1 每个路由必须有 summary 和 description

```python
@router.get(
    "/items",
    summary="获取物品列表",                              # 必填 — Apifox 接口名称
    description="分页查询物品列表，支持按名称搜索",          # 必填 — Apifox 接口说明
    response_model=ItemListResponse,
)
```

- `summary`：简短的接口名称（一般不超过 15 字）
- `description`：完整说明，包括功能、限制条件、特殊行为

### 2.2 必须声明 tags

```python
# 在 Router 级别声明，该 Router 下所有接口共享
item_router = APIRouter(prefix="/items", tags=["物品管理"])
```

tags 决定了 Apifox 左侧目录的分组。没有 tags 的接口会堆在一起，不可接受。

### 2.3 必须声明 response_model

```python
# 正确 — Apifox 自动生成响应文档
@router.get("/items/{item_id}", response_model=ItemResponse)

# 错误 — Apifox 响应文档为空
@router.get("/items/{item_id}")
```

### 2.4 创建接口声明 status_code=201

```python
@router.post(
    "/items",
    response_model=ItemResponse,
    status_code=201,    # POST 创建资源应返回 201
)
```

### 2.5 参数必须有 description

```python
# Query 参数
keyword: Optional[str] = Query(None, description="搜索关键词，模糊匹配物品名称")
page: int = Query(1, description="页码", ge=1)

# Path 参数
item_id: int = Path(..., description="物品 ID", examples=[1])
```

---

## 3. 错误响应

### 3.1 定义公共错误响应字典

在 views 文件顶部定义，所有路由复用：

```python
RESP_401 = {401: {"model": ErrorResponse, "description": "未登录或令牌无效"}}
RESP_403 = {403: {"model": ErrorResponse, "description": "无权限执行此操作"}}
RESP_404 = {404: {"model": ErrorResponse, "description": "资源不存在"}}
RESP_409 = {409: {"model": ErrorResponse, "description": "资源冲突（如重复创建）"}}
RESP_422 = {422: {"model": ValidationErrorResponse, "description": "请求参数校验失败"}}
```

### 3.2 每个路由按需引用

```python
# 查询详情 — 可能 401 或 404
@router.get(
    "/{item_id}",
    response_model=ItemResponse,
    responses={**RESP_401, **RESP_404},
)

# 创建 — 可能 401、409（重复）、422（校验失败）
@router.post(
    "",
    response_model=ItemResponse,
    status_code=201,
    responses={**RESP_401, **RESP_409, **RESP_422},
)

# 删除 — 可能 401、403（无权限）、404
@router.delete(
    "/{item_id}",
    response_model=MessageResponse,
    responses={**RESP_401, **RESP_403, **RESP_404},
)
```

### 3.3 业务特定错误可以内联

```python
@router.post(
    "/orders",
    responses={
        **RESP_401,
        **RESP_422,
        400: {"model": ErrorResponse, "description": "库存不足"},
    },
)
```

### 3.4 各操作的标准错误响应速查表

| 操作 | 401 | 403 | 404 | 409 | 422 |
|------|-----|-----|-----|-----|-----|
| 列表查询 | Y | | | | |
| 获取详情 | Y | | Y | | |
| 创建 | Y | | | Y | Y |
| 更新 | Y | Y | Y | | Y |
| 删除 | Y | Y | Y | | |
| 状态变更 | Y | Y | Y | Y | |

---

## 4. 废弃标记

接口不再推荐使用时，加 `deprecated=True`：

```python
@router.get(
    "/items/old-endpoint",
    summary="获取物品（旧版）",
    description="已废弃，请使用 GET /api/v1/items 代替",
    deprecated=True,
    response_model=ItemListResponse,
)
```

Apifox 中该接口会显示删除线，提醒前端不要再用。

---

## 5. 代码 → Apifox 映射速查

### 5.1 Python (FastAPI + Pydantic)

| 代码写法 | Apifox 位置 |
|---------|------------|
| `Field(description="...")` | 字段备注 |
| `Field(examples=[...])` | 字段示例 |
| `Field(ge=, le=, max_length=, ...)` | 字段校验规则 |
| `Optional[str] = None` | 标记为非必填 |
| `str = Field(...)` | 标记为必填 |
| `class OrderStatus(str, Enum)` | 枚举可选值 |
| `summary="..."` | 接口名称 |
| `description="..."` (路由级) | 接口说明 |
| `tags=["..."]` | 目录分组 |
| `response_model=` | 返回响应 Schema |
| `responses={404: ...}` | 错误响应定义 |
| `status_code=201` | 成功状态码 |
| `deprecated=True` | 废弃标记（删除线） |
| `Query(description="...")` | 请求参数备注 |
| `Path(description="...")` | 路径参数备注 |

### 5.2 Java (Spring Boot + springdoc)

| 代码写法 | Apifox 位置 |
|---------|------------|
| `@Schema(description="...")` | 字段备注 |
| `@Schema(example="...")` | 字段示例 |
| `@Schema(minimum="0", maximum="100")` | 字段校验规则 |
| `@Schema(requiredMode = NOT_REQUIRED)` | 标记为非必填 |
| `@Schema(allowableValues={"A","B"})` | 枚举可选值 |
| `@Operation(summary="...")` | 接口名称 |
| `@Operation(description="...")` | 接口说明 |
| `@Tag(name="...")` | 目录分组 |
| `@ApiResponse(responseCode="200")` | 返回响应 |
| `@ApiResponse(responseCode="404")` | 错误响应定义 |
| `@Operation(deprecated=true)` | 废弃标记 |
| `@Parameter(description="...")` | 请求参数备注 |

### 5.3 Go (swag 注解)

| 代码写法 | Apifox 位置 |
|---------|------------|
| `// @Summary 获取物品列表` | 接口名称 |
| `// @Description 分页查询...` | 接口说明 |
| `// @Tags 物品管理` | 目录分组 |
| `// @Param page query int false "页码"` | 请求参数备注 |
| `// @Success 200 {object} ItemResponse` | 返回响应 |
| `// @Failure 404 {object} ErrorResponse` | 错误响应定义 |
| `// @Deprecated` | 废弃标记 |

### 5.4 TypeScript (NestJS + @nestjs/swagger)

| 代码写法 | Apifox 位置 |
|---------|------------|
| `@ApiProperty({description: "..."})` | 字段备注 |
| `@ApiProperty({example: "..."})` | 字段示例 |
| `@ApiPropertyOptional()` | 标记为非必填 |
| `@ApiProperty({enum: Status})` | 枚举可选值 |
| `@ApiOperation({summary: "..."})` | 接口名称 |
| `@ApiOperation({description: "..."})` | 接口说明 |
| `@ApiTags("...")` | 目录分组 |
| `@ApiResponse({status: 200, type: Item})` | 返回响应 |
| `@ApiResponse({status: 404, type: Error})` | 错误响应定义 |
| `@ApiOperation({deprecated: true})` | 废弃标记 |
| `@ApiQuery({description: "..."})` | 请求参数备注 |

---

## 6. 禁止事项

| 禁止 | 原因 |
|------|------|
| 在 Apifox 中手动编辑接口描述 | 下次同步会被代码覆盖 |
| 在 Apifox 中手动添加接口 | 应在代码中添加路由 |
| 在示例值中使用真实数据 | JWT 令牌、邮箱、手机号、内网 IP 会泄露到文档 |
| 路由不写 summary | Spectral lint 会报错，PR 无法合并 |
| 路由不写 response_model | Apifox 中响应文档为空 |
| 路由不写 tags | 接口不分组，文档不可浏览 |
| Schema 字段不写 description | Apifox 字段备注为空，前端不知道字段含义 |

---

## 7. 完整示例

一个符合规范的最小接口：

```python
# schemas.py
class ItemCreate(BaseModel):
    name: str = Field(
        ...,
        description="物品名称，不超过 100 个字符",
        max_length=100,
        examples=["苹果"],
    )
    price: float = Field(
        ...,
        description="价格，单位：元",
        ge=0,
        examples=[9.99],
    )
    description: Optional[str] = Field(
        None,
        description="物品描述，支持 Markdown",
        max_length=2000,
    )


class ItemResponse(BaseModel):
    id: int = Field(..., description="物品 ID", examples=[1])
    name: str = Field(..., description="物品名称")
    price: float = Field(..., description="价格")
    description: Optional[str] = Field(None, description="物品描述")
    created_at: str = Field(..., description="创建时间", examples=["2024-01-01T00:00:00Z"])


# views.py
item_router = APIRouter(prefix="/items", tags=["物品管理"])

RESP_401 = {401: {"model": ErrorResponse, "description": "未登录或令牌无效"}}
RESP_404 = {404: {"model": ErrorResponse, "description": "资源不存在"}}
RESP_422 = {422: {"model": ValidationErrorResponse, "description": "请求参数校验失败"}}


@item_router.post(
    "",
    summary="创建物品",
    description="创建一个新的物品，初始状态为 draft",
    response_model=ItemResponse,
    status_code=201,
    responses={**RESP_401, **RESP_422},
)
async def create_item(body: ItemCreate):
    ...


@item_router.get(
    "/{item_id}",
    summary="获取物品详情",
    description="根据 ID 获取物品详细信息",
    response_model=ItemResponse,
    responses={**RESP_401, **RESP_404},
)
async def get_item(
    item_id: int = Path(..., description="物品 ID", examples=[1]),
):
    ...
```
