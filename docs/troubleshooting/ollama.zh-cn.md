# Ollama 故障排除

## 流式传输问题：JSON 解析错误

### 问题

当将 Ollama 作为提供程序与 oh-my-opencode 代理一起使用时，您可能会遇到：

```
JSON 解析错误：意外的 EOF
```

当代理尝试工具调用（例如，`explore` 代理使用 `mcp_grep_search`）时，会发生这种情况。

### 根本原因

Ollama 在 API 请求中使用 `stream: true` 时返回 **NDJSON**（换行符分隔的 JSON）：

```json
{"message":{"tool_calls":[{"function":{"name":"read","arguments":{"filePath":"README.md"}}}]}, "done":false}
{"message":{"content":""}, "done":true}
```

Claude Code SDK 期望单个 JSON 对象，而不是多个 NDJSON 行，导致解析错误。

**为什么会发生这种情况：**
- **Ollama API**：设计上将流式响应作为 NDJSON 返回
- **Claude Code SDK**：不能正确处理工具调用的 NDJSON 响应
- **oh-my-opencode**：传递 SDK 的行为（无法在此层修复）

## 解决方案

### 方案 1：禁用流式传输（推荐）

将您的 Ollama 提供程序配置为使用 `stream: false`：

```json
{
  "provider": "ollama",
  "model": "qwen3-coder",
  "stream": false
}
```

**优点：**
- 立即工作
- 无需代码更改
- 简单配置

**缺点：**
- 响应时间稍慢（无流式传输）
- 交互性反馈较少

### 方案 2：仅使用非工具代理

如果您需要流式传输，避免使用工具的代理：

- **安全**：简单的文本生成、非工具任务
- **有问题**：任何有工具调用的代理（explore、librarian 等）

### 方案 3：等待 SDK 修复

正确的修复需要 Claude Code SDK：

1. 检测 NDJSON 响应
2. 单独解析每一行
3. 从多行合并 `tool_calls`
4. 返回单个合并的响应

**跟踪**：https://github.com/code-yeongyu/oh-my-opencode/issues/1124

## 变通实施

在 SDK 修复之前，以下是实施 NDJSON 解析的方法（适用于 SDK 维护者）：

```typescript
async function parseOllamaStreamResponse(response: string): Promise<object> {
  const lines = response.split('\n').filter(line => line.trim());
  const mergedMessage = { tool_calls: [] };

  for (const line of lines) {
    try {
      const json = JSON.parse(line);
      if (json.message?.tool_calls) {
        mergedMessage.tool_calls.push(...json.message.tool_calls);
      }
      if (json.message?.content) {
        mergedMessage.content = json.message.content;
      }
    } catch (e) {
      // 跳过格式错误的行
      console.warn('Skipping malformed NDJSON line:', line);
    }
  }

  return mergedMessage;
}
```

## 测试

要验证修复有效：

```bash
# 使用 curl 测试（stream: false 应该工作）
curl -s http://localhost:11434/api/chat \
  -d '{
    "model": "qwen3-coder",
    "messages": [{"role": "user", "content": "读取文件 README.md"}],
    "stream": false,
    "tools": [{"type": "function", "function": {"name": "read", "description": "读取文件", "parameters": {"type": "object", "properties": {"filePath": {"type": "string"}}, "required": ["filePath"]}}]
  }'
```

## 相关问题

- **oh-my-opencode**：https://github.com/code-yeongyu/oh-my-opencode/issues/1124
- **Ollama API 文档**：https://github.com/ollama/ollama/blob/main/docs/api.md

## 获取帮助

如果您遇到此问题：

1. 检查您的 Ollama 提供程序配置
2. 将 `stream: false` 设置为变通方案
3. 向问题跟踪器报告任何其他错误
4. 提供您的配置（不带秘密）以进行调试
