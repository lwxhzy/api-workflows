# api-workflows

可复用的 GitHub Actions 工作流，为 API 项目提供 **PR 审查** 和 **Apifox 自动同步** 能力。

支持任意语言和框架（Python / Java / Go / TypeScript / ...），只要能导出 `openapi.json` 即可接入。

业务项目只需添加一个约 30 行的 YAML 文件 + 配置 Secrets。

> **编码规范**：接入前务必阅读 [CONVENTIONS.md](./CONVENTIONS.md)，它定义了代码中如何编写描述、示例、错误响应等，确保同步到 Apifox 的文档完整可用。

## 功能概览

| 触发时机 | Workflow | 做什么 |
|---------|----------|--------|
| PR → 目标分支 | `api-review.yml` | Spectral lint 质量校验 → oasdiff 破坏性变更检测 → PR 评论展示变更摘要 |
| 合并/推送 → 目标分支 | `sync-apifox.yml` | 导出 OpenAPI spec → 调用 Apifox API 自动同步接口文档 |

## 快速接入（3 步）

### 1. 添加 Workflow 文件

在你的项目中创建 `.github/workflows/api-ci.yml`：

```yaml
name: API CI

on:
  pull_request:
    branches: [dev]          # 改成你的目标分支
    paths:
      - "src/**"             # 改成你的 API 源码路径
  push:
    branches: [dev]
    paths:
      - "src/**"

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    if: github.event_name == 'pull_request'
    uses: lwxhzy/api-workflows/.github/workflows/api-review.yml@main
    with:
      setup-command: "pip install -r requirements.txt"       # 改成你的安装命令
      export-command: "python scripts/export_openapi.py"     # 改成你的导出命令
      base-branch: dev
    secrets: inherit

  sync:
    if: github.event_name == 'push'
    uses: lwxhzy/api-workflows/.github/workflows/sync-apifox.yml@main
    with:
      setup-command: "pip install -r requirements.txt"
      export-command: "python scripts/export_openapi.py"
    secrets: inherit
```

### 2. 配置 GitHub Secrets

进入仓库 **Settings → Secrets and variables → Actions**，添加：

| Secret 名称 | 说明 | 获取方式 |
|-------------|------|---------|
| `APIFOX_ACCESS_TOKEN` | Apifox API 访问令牌 | Apifox → 头像 → 账号设置 → API 访问令牌 → 新建 |
| `APIFOX_PROJECT_ID` | Apifox 项目 ID | Apifox → 项目设置 → 基本设置 → 项目 ID |

> 如果使用 GitHub Organization，`APIFOX_ACCESS_TOKEN` 可以设为 Organization 级别的 Secret，所有仓库共享，只需配一次。

### 3. 确保项目有导出命令

你的项目需要有一个命令，能将 OpenAPI spec 导出为当前目录下的 `openapi.json` 文件。

## 各语言接入示例

### Python (FastAPI)

```yaml
setup-command: "pip install -r requirements.txt"
export-command: "python scripts/export_openapi.py"
```

创建 `scripts/export_openapi.py`：

```python
import json, sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent.parent))
from main import app

spec = app.openapi()
Path(sys.argv[1] if len(sys.argv) > 1 else "openapi.json").write_text(
    json.dumps(spec, ensure_ascii=False, indent=2), encoding="utf-8"
)
```

### Python (Django + drf-spectacular)

```yaml
setup-command: "pip install -r requirements.txt"
export-command: "python manage.py spectacular --file openapi.json"
```

无需额外脚本，drf-spectacular 自带导出命令。

### Java (Spring Boot + springdoc-openapi)

```yaml
setup-command: "mvn dependency:resolve -q -B"
export-command: "mvn -q springdoc-openapi:generate"
```

在 `pom.xml` 中配置 springdoc-openapi-maven-plugin，输出路径设为 `openapi.json`。

### Go (swag)

```yaml
setup-command: "go install github.com/swaggo/swag/cmd/swag@latest"
export-command: "swag init -o . --outputTypes json && mv docs/swagger.json openapi.json"
```

### TypeScript (NestJS + @nestjs/swagger)

```yaml
setup-command: "npm ci"
export-command: "npx ts-node scripts/export-openapi.ts"
```

创建 `scripts/export-openapi.ts`：

```typescript
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from '../src/app.module';
import * as fs from 'fs';

async function main() {
  const app = await NestFactory.create(AppModule, { logger: false });
  const config = new DocumentBuilder().setTitle('API').setVersion('1.0').build();
  const doc = SwaggerModule.createDocument(app, config);
  fs.writeFileSync('openapi.json', JSON.stringify(doc, null, 2));
  await app.close();
}
main();
```

### TypeScript (Express + tsoa)

```yaml
setup-command: "npm ci"
export-command: "npx tsoa spec"
```

在 `tsoa.json` 中配置 `outputDirectory` 和 `specFileBaseName`，确保输出为 `openapi.json`。

## Workflow 参数参考

### api-review.yml

PR 阶段自动审查 API 变更。

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `setup-command` | string | 是 | — | 安装项目依赖的命令 |
| `export-command` | string | 是 | — | 导出 OpenAPI spec 到 `openapi.json` 的命令 |
| `base-branch` | string | 否 | `dev` | 用于 diff 对比的基准分支 |

**审查流程：**

1. 导出 PR 分支的 OpenAPI spec
2. 导出基准分支的 OpenAPI spec（首次 PR 无基准时自动跳过）
3. [Spectral](https://github.com/stoplightio/spectral) lint — 校验 spec 质量
4. [oasdiff](https://github.com/Tufin/oasdiff) breaking — 检测破坏性变更（有则工作流失败）
5. oasdiff changelog — 生成变更日志
6. 在 PR 评论中展示变更摘要（重复推送自动更新同一条评论）

**自定义 lint 规则：** 在你的项目根目录放一个 `.spectral.yaml`，Spectral 会自动加载。不放则使用默认规则。

### sync-apifox.yml

合并到目标分支后自动同步到 Apifox。

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `setup-command` | string | 是 | — | 安装项目依赖的命令 |
| `export-command` | string | 是 | — | 导出 OpenAPI spec 到 `openapi.json` 的命令 |

| Secret | 必填 | 说明 |
|--------|------|------|
| `APIFOX_ACCESS_TOKEN` | 是 | Apifox API 访问令牌 |
| `APIFOX_PROJECT_ID` | 是 | 目标 Apifox 项目 ID |

**同步行为：**

- 使用 `OVERWRITE_EXISTING` 策略 — 已有同路径接口会被覆盖更新
- 数据模型（Schema）同样覆盖更新
- 不会删除 Apifox 中已有但 spec 中不存在的接口

## 同步内容

代码中定义的以下信息都会自动同步到 Apifox：

| 类别 | 内容 |
|------|------|
| 接口 | 路径、HTTP 方法、summary、description、tags 分组 |
| 参数 | Query / Path / Header 参数的名称、类型、描述、默认值、校验规则 |
| 请求体 | 字段结构、类型、是否必填、描述、示例值 |
| 响应体 | 状态码、字段结构、类型、描述 |
| 数据模型 | Schema 类的完整定义 |
| 全局信息 | API 标题、描述、版本号 |

## 团队协作规范：代码为唯一事实来源

本 workflow 采用 **代码为准（Code-First）** 策略：所有 API 的描述、注释、示例值都在代码中维护，同步时覆盖 Apifox 中的对应内容。

### 为什么

- 描述跟着代码走，PR 审查时可以一起 review
- 不会出现"代码改了但文档没更新"的问题
- 多人协作时不会互相覆盖——代码有 git 合并机制，Apifox 没有

### 在代码里写什么

以 FastAPI + Pydantic 为例：

```python
# schemas.py — 字段级别的描述、示例、校验规则
class ItemCreate(BaseModel):
    name: str = Field(
        ...,
        description="物品名称，不超过 50 个字符",   # → Apifox 字段备注
        examples=["苹果"],                          # → Apifox 示例值
        max_length=50,                              # → Apifox 校验规则
    )
    price: float = Field(
        ...,
        description="价格，单位：元",
        ge=0,
        examples=[9.99],
    )


# views.py — 接口级别的名称、说明、分组
@router.get(
    "/items",
    summary="获取物品列表",                          # → Apifox 接口名称
    description="分页查询物品列表，支持按名称搜索",     # → Apifox 接口说明
    tags=["物品管理"],                                # → Apifox 目录分组
    response_model=ItemListResponse,
)
async def list_items(
    keyword: Optional[str] = Query(
        None,
        description="搜索关键词，模糊匹配物品名称",    # → Apifox 参数备注
    ),
):
    ...
```

其他语言/框架的注解方式不同，但原理一致——在代码中标注 summary、description、example，同步后自动填充到 Apifox。

### 代码注释与 Apifox 的映射关系

| OpenAPI 字段 | 同步到 Apifox | 代码中怎么写（各语言） |
|-------------|--------------|---------------------|
| `summary` | 接口名称 | Python: `summary=` / Java: `@Operation(summary=)` / Go: `@Summary` / TS: `@ApiOperation({summary:})` |
| `description` | 接口说明 | Python: `description=` / Java: `@Operation(description=)` / Go: `@Description` / TS: `@ApiOperation({description:})` |
| `tags` | 分组目录 | Python: `tags=` / Java: `@Tag(name=)` / Go: `@Tags` / TS: `@ApiTags()` |
| Schema `description` | 字段备注 | Python: `Field(description=)` / Java: `@Schema(description=)` / Go: `// field comment` / TS: `@ApiProperty({description:})` |
| Schema `example` | 示例值 | Python: `Field(examples=)` / Java: `@Schema(example=)` / Go: `@Example` / TS: `@ApiProperty({example:})` |

### Apifox 里做什么

同步**不会影响**以下 Apifox 独有功能，可以放心在 Apifox 中维护：

- Mock 规则（自定义返回数据）
- 测试用例（接口自动化测试）
- 环境配置（开发 / 测试 / 生产环境变量）
- 前后置脚本
- 接口运行历史

### 不要做的事

- 不要在 Apifox 中手动修改接口的描述、参数说明、字段备注——下次同步会被代码覆盖
- 不要在 Apifox 中手动添加新接口——应该在代码中添加路由，通过同步自动创建

## 完整接入示例

可参考 [lwxhzy/test-apifox](https://github.com/lwxhzy/test-apifox) 项目，它是一个使用本 workflow 的 FastAPI 示例。
