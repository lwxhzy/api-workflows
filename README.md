# api-workflows

可复用的 GitHub Actions 工作流，为 Python API 项目提供 **PR 审查** 和 **Apifox 自动同步** 能力。

业务项目只需添加一个约 30 行的 YAML 文件 + 配置 Secrets，即可接入全套 API CI 流水线。

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
      - "app/**"             # 改成你的 API 源码路径
      - "main.py"
      - "requirements.txt"
  push:
    branches: [dev]          # 同上
    paths:
      - "app/**"
      - "main.py"
      - "requirements.txt"

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    if: github.event_name == 'pull_request'
    uses: lwxhzy/api-workflows/.github/workflows/api-review.yml@main
    with:
      export-command: "python scripts/export_openapi.py"   # 改成你的导出命令
      base-branch: dev                                      # 改成你的目标分支
    secrets: inherit

  sync:
    if: github.event_name == 'push'
    uses: lwxhzy/api-workflows/.github/workflows/sync-apifox.yml@main
    with:
      export-command: "python scripts/export_openapi.py"   # 同上
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

**FastAPI 示例** — 创建 `scripts/export_openapi.py`：

```python
import json
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent.parent))

from main import app

def main():
    output_path = sys.argv[1] if len(sys.argv) > 1 else "openapi.json"
    spec = app.openapi()
    Path(output_path).write_text(
        json.dumps(spec, ensure_ascii=False, indent=2), encoding="utf-8"
    )

if __name__ == "__main__":
    main()
```

**其他框架**只要能生成 `openapi.json` 即可，例如：

```yaml
# Django + drf-spectacular
export-command: "python manage.py spectacular --file openapi.json"

# Flask + apispec
export-command: "python scripts/export_spec.py"
```

## Workflow 参数参考

### api-review.yml

PR 阶段自动审查 API 变更。

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `export-command` | string | 是 | — | 导出 OpenAPI spec 到 `openapi.json` 的命令 |
| `python-version` | string | 否 | `3.12` | Python 版本 |
| `requirements-file` | string | 否 | `requirements.txt` | 依赖文件路径 |
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
| `export-command` | string | 是 | — | 导出 OpenAPI spec 到 `openapi.json` 的命令 |
| `python-version` | string | 否 | `3.12` | Python 版本 |
| `requirements-file` | string | 否 | `requirements.txt` | 依赖文件路径 |

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
| 数据模型 | Pydantic / Schema 类的完整定义 |
| 全局信息 | API 标题、描述、版本号 |

## 完整接入示例

可参考 [lwxhzy/test-apifox](https://github.com/lwxhzy/test-apifox) 项目，它是一个使用本 workflow 的 FastAPI 示例。
