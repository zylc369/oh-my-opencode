# CLI 参考

`oh-my-opencode` 命令行界面的完整参考。

## 基本用法

```bash
# 显示帮助
bunx oh-my-opencode

# 或使用 npx
npx oh-my-opencode
```

## 命令

| 命令 | 描述 |
| --- | --- |
| `install` | 交互式设置向导 |
| `doctor` | 环境诊断和健康检查 |
| `run` | OpenCode 会话运行器 |
| `mcp oauth` | MCP OAuth 身份验证管理 |
| `auth` | Google Antigravity OAuth 身份验证 |
| `get-local-version` | 显示本地版本信息 |

---

## install

Oh-My-OpenCode 初始设置的交互式安装工具。提供基于 `@clack/prompts` 的 TUI。

### 用法

```bash
bunx oh-my-opencode install
```

### 安装过程

1. **提供程序选择**：选择您的 AI 提供程序（Claude、ChatGPT 或 Gemini）
2. **API 密钥输入**：输入您选择的提供程序的 API 密钥
3. **配置文件创建**：生成 `opencode.json` 或 `oh-my-opencode.json` 文件
4. **插件注册**：在 OpenCode 设置中自动注册 oh-my-opencode 插件

### 选项

| 选项 | 描述 |
| --- | --- |
| `--no-tui` | 在非交互式模式下运行而不使用 TUI（适用于 CI/CD 环境） |
| `--verbose` | 显示详细日志 |

---

## doctor

诊断您的环境以确保 Oh-My-OpenCode 正确工作。执行 17+ 次健康检查。

### 用法

```bash
bunx oh-my-opencode doctor
```

### 诊断类别

| 类别 | 检查项 |
| --- | --- |
| **安装** | OpenCode 版本（>= 1.0.150）、插件注册状态 |
| **配置** | 配置文件有效性、JSONC 解析 |
| **身份验证** | Anthropic、OpenAI、Google API 密钥有效性 |
| **依赖项** | Bun、Node.js、Git 安装状态 |
| **工具** | LSP 服务器状态、MCP 服务器状态 |
| **更新** | 最新版本检查 |

### 选项

| 选项 | 描述 |
| --- | --- |
| `--category <name>` | 仅检查特定类别（例如，`--category authentication`） |
| `--json` | 以 JSON 格式输出结果 |
| `--verbose` | 包括详细信息 |

### 示例输出

```
oh-my-opencode doctor

┌──────────────────────────────────────────────────┐
│  Oh-My-OpenCode Doctor                           │
└──────────────────────────────────────────────────┘

安装
  ✓ OpenCode 版本：1.0.155 (>= 1.0.150)
  ✓ 插件已在 opencode.json 中注册

配置
  ✓ oh-my-opencode.json 有效
  ⚠ categories.visual-engineering：使用默认模型

身份验证
  ✓ Anthropic API 密钥已配置
  ✓ OpenAI API 密钥已配置
  ✗ 未找到 Google API 密钥

依赖项
  ✓ Bun 1.2.5 已安装
  ✓ Node.js 22.0.0 已安装
  ✓ Git 2.45.0 已安装

摘要：10 通过，1 警告，1 失败
```

---

## run

执行 OpenCode 会话并监控任务完成情况。

### 用法

```bash
bunx oh-my-opencode run [提示词]
```

### 选项

| 选项 | 描述 |
| --- | --- |
| `--enforce-completion` | 保持会话活动，直到所有待办事项完成 |
| `--timeout <秒>` | 设置最大执行时间 |
| `--agent <名称>` | 指定要使用的代理 |
| `--directory <路径>` | 设置工作目录 |
| `--port <编号>` | 设置会话端口 |
| `--attach` | 附加到现有会话 |
| `--json` | 以 JSON 格式输出 |
| `--no-timestamp` | 禁用带时间戳的输出 |
| `--session-id <id>` | 恢复现有会话 |
| `--on-complete <操作>` | 完成时的操作 |
| `--verbose` | 启用详细日志记录 |

---

## mcp oauth

管理远程 MCP 服务器的 OAuth 2.1 身份验证。

### 用法

```bash
# 登录到 OAuth 保护的 MCP 服务器
bunx oh-my-opencode mcp oauth login <服务器名称> --server-url https://api.example.com

# 使用明确的客户端 ID 和范围登录
bunx oh-my-opencode mcp oauth login my-api --server-url https://api.example.com --client-id my-client --scopes "read,write"

# 删除存储的 OAuth 令牌
bunx oh-my-opencode mcp oauth logout <服务器名称>

# 检查 OAuth 令牌状态
bunx oh-my-opencode mcp oauth status [服务器名称]
```

### 选项

| 选项 | 描述 |
| --- | --- |
| `--server-url <url>` | MCP 服务器 URL（登录时需要） |
| `--client-id <id>` | OAuth 客户端 ID（如果服务器支持动态客户端注册，则可选） |
| `--scopes <范围>` | 逗号分隔的 OAuth 范围 |

### 令牌存储

令牌以 `0600` 权限（仅所有者读写）存储在 `~/.config/opencode/mcp-oauth.json` 中。密钥格式：`{服务器主机}/{资源}`。

---

## 配置文件

CLI 在以下位置（按优先级顺序）搜索配置文件：

1. **项目级别**：`.opencode/oh-my-opencode.json`
2. **用户级别**：`~/.config/opencode/oh-my-opencode.json`

### JSONC 支持

配置文件支持**JSONC（带注释的 JSON）** 格式。您可以使用注释和尾随逗号。

```jsonc
{
  // 代理配置
  "sisyphus_agent": {
    "disabled": false,
    "planner_enabled": true,
  },

  /* 类别自定义 */
  "categories": {
    "visual-engineering": {
      "model": "google/gemini-3.1-pro",
    },
  },
}
```

---

## 故障排除

### "OpenCode 版本太旧"错误

```bash
# 更新 OpenCode
npm install -g opencode@latest
# 或
bun install -g opencode@latest
```

### "插件未注册"错误

```bash
# 重新安装插件
bunx oh-my-opencode install
```

### Doctor 检查失败

```bash
# 详细诊断
bunx oh-my-opencode doctor --verbose

# 仅检查特定类别
bunx oh-my-opencode doctor --category authentication
```

---

## 非交互式模式

使用 `--no-tui` 选项进行 CI/CD 环境。

```bash
# 在 CI 环境中运行 doctor
bunx oh-my-opencode doctor --no-tui --json

# 将结果保存到文件
bunx oh-my-opencode doctor --json > doctor-report.json
```

---

## 开发者信息

### CLI 结构

```
src/cli/
├── cli-program.ts        # 基于 Commander.js 的主入口
├── install.ts            # 基于 @clack/prompts 的 TUI 安装程序
├── config-manager/       # JSONC 解析、多源配置管理
│   └── *.ts
├── doctor/               # 健康检查系统
│   ├── index.ts          # doctor 命令入口
│   └── checks/           # 17+ 个单独检查模块
├── run/                  # 会话运行器
│   └── *.ts
└── mcp-oauth/            # OAuth 管理命令
    └── *.ts
```

### 添加新的 Doctor 检查

创建 `src/cli/doctor/checks/my-check.ts`：

```typescript
import type { DoctorCheck } from "../types";

export const myCheck: DoctorCheck = {
  name: "my-check",
  category: "environment",
  check: async () => {
    // 检查逻辑
    const isOk = await someValidation();

    return {
      status: isOk ? "pass" : "fail",
      message: isOk ? "一切看起来都很好" : "出问题了",
    };
  },
};
```

在 `src/cli/doctor/checks/index.ts` 中注册：

```typescript
export { myCheck } from "./my-check";
```
