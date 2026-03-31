# Claude Code 开发者指南

> 基于还原的 `@anthropic-ai/claude-code` 源码编写，帮助你理解和魔改这个强大的 AI 编程助手

---

## 目录

1. [项目概述](#项目概述)
2. [快速开始](#快速开始)
3. [架构总览](#架构总览)
4. [核心模块详解](#核心模块详解)
5. [工具系统](#工具系统)
6. [命令系统](#命令系统)
7. [状态管理](#状态管理)
8. [UI 组件](#ui-组件)
9. [功能开关系统](#功能开关系统)
10. [修改指南](#修改指南)
11. [调试技巧](#调试技巧)
12. [高级主题](#高级主题)
13. [完整示例](#完整示例)

---

## 项目概述

Claude Code 是一个基于终端的 AI 编程助手，使用 React + Ink 构建 TUI（终端用户界面），支持复杂的工具调用系统、多 Agent 协作、MCP 扩展等功能。

### 技术栈

- **运行时**: Bun (>= 1.3.5) / Node.js (>= 24)
- **语言**: TypeScript (ESM)
- **UI 框架**: React + Ink (终端渲染)
- **状态管理**: 自定义 Store 系统
- **构建**: Bun bundle

### 源码统计

- **TypeScript 文件**: ~1,900 个
- **核心工具**: 53 个
- **斜杠命令**: 87 个
- **UI 组件**: 148 个
- **自定义 Hooks**: 87 个

---

## 快速开始

### 环境准备

```bash
# 安装 Bun (如果尚未安装)
curl -fsSL https://bun.sh/install | bash

# 克隆并进入项目
cd claude-code

# 安装依赖
bun install

# 启动开发版本
bun run dev

# 验证版本
bun run version
```

### 开发工作流

```bash
# 1. 启动交互式 CLI
bun run dev

# 2. 在另一个终端运行特定测试
bun test src/tools/BashTool/

# 3. 构建检查
bun run build

# 4. 代码格式化
bun run format
```

---

## 架构总览

### 目录结构

```
src/
├── main.tsx              # 应用入口 & CLI 参数解析
├── dev-entry.ts          # 开发环境入口
├── QueryEngine.ts        # 核心查询引擎
├── query.ts              # 查询主循环
├── Tool.ts               # 工具类型定义
├── tools.ts              # 工具注册
├── commands.ts           # 命令注册
├── context.ts            # 全局上下文
├── bootstrap/            # 启动状态管理
├── commands/             # 斜杠命令实现
├── components/           # React UI 组件
├── constants/            # 常量定义
├── hooks/                # 自定义 React Hooks
├── ink/                  # Ink TUI 框架核心
├── services/             # 业务服务层
│   ├── api/              # API 调用
│   ├── mcp/              # MCP 客户端
│   ├── analytics/        # 分析遥测
│   └── compact/          # 自动压缩
├── state/                # 应用状态管理
├── tools/                # 工具实现
├── types/                # TypeScript 类型
└── utils/                # 工具函数
```

### 核心数据流

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              用户交互层                                       │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────────────────────────┐│
│  │  用户输入    │     │  斜杠命令    │     │  快捷键/交互事件                  ││
│  │  (自然语言)  │────▶│  (/command) │     │  (Vim/Emacs模式)                 ││
│  └─────────────┘     └─────────────┘     └─────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           QueryEngine 核心层                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                     submitMessage() 方法                                ││
│  │  1. 处理用户输入                                                         ││
│  │  2. 构建系统提示词                                                        ││
│  │  3. 调用 query() 主循环                                                   ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                             query.ts 主循环                                  │
│                                                                              │
│  while (true) {                                                              │
│    ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌─────────────┐ │
│    │  上下文压缩   │──▶│  API 调用    │──▶│  流式响应    │──▶│  工具执行    │ │
│    │  (snip/MC)   │   │  (claude.ts) │   │  处理        │   │  (runTools)  │ │
│    └──────────────┘   └──────────────┘   └──────────────┘   └──────┬──────┘ │
│                                                                     │        │
│                              ◀──────────────────────────────────────┘        │
│                              (如果需要更多工具调用，继续循环)                   │
│  }                                                                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
        ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
        │   内置工具     │   │   MCP 工具    │   │   子 Agent    │
        │  (tools/)     │   │  (外部服务)   │   │  (AgentTool)  │
        │               │   │               │   │               │
        │ • BashTool    │   │ • 数据库查询   │   │ • 代码审查    │
        │ • FileEdit    │   │ • API 调用    │   │ • 测试生成    │
        │ • WebSearch   │   │ • 自定义工具   │   │ • 文档生成    │
        └───────────────┘   └───────────────┘   └───────────────┘
```

### 启动流程

```
┌─────────────────────────────────────────────────────────────────┐
│                         启动流程                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. dev-entry.ts                                                │
│     ├── 检查缺失的模块依赖                                       │
│     ├── 处理 --version/--help                                    │
│     └── 转发到 entrypoints/cli.tsx 或 main.tsx                   │
│                                                                 │
│  2. main.tsx                                                    │
│     ├── 启动性能分析 (profileCheckpoint)                         │
│     ├── 预取系统上下文 (keychain/MDM)                            │
│     ├── 解析 CLI 参数 (--print, --model, etc.)                   │
│     ├── 初始化 Commander.js                                      │
│     └── 执行 preAction 钩子                                      │
│                                                                 │
│  3. entrypoints/init.ts                                         │
│     ├── 初始化配置系统                                           │
│     ├── 加载设置                                                 │
│     ├── 初始化遥测/分析                                          │
│     └── 运行迁移脚本                                             │
│                                                                 │
│  4. screens/REPL.tsx                                            │
│     ├── 初始化 Ink 渲染器                                        │
│     ├── 加载插件/MCP                                             │
│     ├── 恢复会话（如果 --resume）                                │
│     └── 启动主输入循环                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 核心模块详解

### 1. QueryEngine - 查询引擎

**文件**: `src/QueryEngine.ts`

QueryEngine 是 Claude Code 的核心，管理对话生命周期和会话状态。

#### 主要职责

- 管理对话状态（消息、文件缓存、使用量等）
- 处理用户输入并生成响应
- 协调工具调用
- 处理会话持久化

#### 关键方法

```typescript
export class QueryEngine {
  constructor(config: QueryEngineConfig)
  
  // 提交消息并获取响应流
  async *submitMessage(
    prompt: string | ContentBlockParam[],
    options?: { uuid?: string; isMeta?: boolean }
  ): AsyncGenerator<SDKMessage, void, unknown>
}
```

#### 配置选项

```typescript
type QueryEngineConfig = {
  cwd: string                    // 工作目录
  tools: Tools                   // 可用工具集
  commands: Command[]            // 可用命令
  mcpClients: MCPServerConnection[]  // MCP 连接
  agents: AgentDefinition[]      // Agent 定义
  canUseTool: CanUseToolFn       // 权限检查函数
  getAppState: () => AppState    // 获取应用状态
  setAppState: (f: (prev: AppState) => AppState) => void  // 更新状态
  initialMessages?: Message[]    // 初始消息
  readFileCache: FileStateCache  // 文件读取缓存
  customSystemPrompt?: string    // 自定义系统提示
  appendSystemPrompt?: string    // 追加系统提示
  maxTurns?: number              // 最大轮数限制
  maxBudgetUsd?: number          // 预算限制
}
```

#### 使用示例

```typescript
import { QueryEngine } from './QueryEngine.js'

const engine = new QueryEngine({
  cwd: process.cwd(),
  tools: getTools(permissionContext),
  commands: await getCommands(cwd),
  mcpClients: [],
  agents: [],
  canUseTool: async (tool, input, context) => {
    // 权限检查逻辑
    return { behavior: 'allow', updatedInput: input }
  },
  getAppState: () => appState,
  setAppState: (updater) => { /* ... */ },
  readFileCache: new Map(),
})

// 使用生成器获取响应
for await (const message of engine.submitMessage('Hello')) {
  console.log(message)
}
```

### 2. query.ts - 查询主循环

**文件**: `src/query.ts`

实现了完整的对话循环逻辑，包括 API 调用、工具执行、自动压缩等。

#### 核心流程

```typescript
async function* queryLoop(params: QueryParams, consumedCommandUuids: string[]) {
  // 1. 初始化状态
  let state: State = {
    messages: params.messages,
    toolUseContext: params.toolUseContext,
    // ...
  }
  
  // 2. 主循环
  while (true) {
    // 3. 应用 snip 压缩（如果启用）
    if (feature('HISTORY_SNIP')) {
      const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
      // ...
    }
    
    // 4. 应用微压缩
    const microcompactResult = await deps.microcompact(messagesForQuery, ...)
    
    // 5. 应用上下文折叠（如果启用）
    if (feature('CONTEXT_COLLAPSE')) {
      const collapseResult = await contextCollapse.applyCollapsesIfNeeded(...)
    }
    
    // 6. 自动压缩（如果需要）
    const { compactionResult } = await deps.autocompact(...)
    
    // 7. 调用模型
    for await (const message of deps.callModel({...})) {
      // 8. 处理流式响应
      // 9. 执行工具调用
    }
    
    // 10. 执行工具
    if (needsFollowUp) {
      // 运行工具并获取结果
    }
  }
}
```

#### 关键决策点

| 检查点 | 目的 | 文件 |
|--------|------|------|
| Snip Compact | 截断早期历史 | `services/compact/snipCompact.ts` |
| Micro Compact | 合并相邻工具调用 | `services/compact/microCompact.ts` |
| Context Collapse | 折叠非关键上下文 | `services/contextCollapse/` |
| Auto Compact | 当接近 Token 限制时自动压缩 | `services/compact/autoCompact.ts` |

### 3. 工具系统核心

**文件**: `src/Tool.ts`

定义了工具的接口和类型。

#### Tool 接口详解

```typescript
type Tool<Input extends AnyObject, Output, P extends ToolProgressData> = {
  // ============ 基础信息 ============
  name: string                    // 工具名称（API 中使用）
  aliases?: string[]              // 别名（向后兼容）
  searchHint?: string             // 工具搜索提示词
  
  // ============ 执行相关 ============
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>
  
  // ============ 描述与模式 ============
  description(
    input: z.infer<Input>,
    options: {
      isNonInteractiveSession: boolean
      toolPermissionContext: ToolPermissionContext
      tools: Tools
    },
  ): Promise<string>              // 返回给模型的描述
  
  readonly inputSchema: Input      // Zod 输入验证模式
  readonly inputJSONSchema?: ToolInputJSONSchema  // JSON Schema 替代
  outputSchema?: z.ZodType<unknown>  // 输出模式
  
  // ============ 权限与安全 ============
  isEnabled(): boolean             // 是否启用
  isReadOnly(input): boolean       // 是否只读（不修改文件系统）
  isDestructive?(input): boolean   // 是否具有破坏性（删除/覆盖）
  checkPermissions(
    input: z.infer<Input>,
    context: ToolUseContext,
  ): Promise<PermissionResult>
  
  // ============ 并发控制 ============
  isConcurrencySafe(input): boolean  // 是否支持并发执行
  interruptBehavior?(): 'cancel' | 'block'  // 中断时的行为
  
  // ============ 渲染（UI） ============
  renderToolUseMessage(
    input: Partial<z.infer<Input>>,
    options: { theme: ThemeName; verbose: boolean; commands?: Command[] },
  ): React.ReactNode
  
  renderToolResultMessage(
    content: Output,
    progressMessagesForMessage: ProgressMessage<P>[],
    options: {
      style?: 'condensed'
      theme: ThemeName
      tools: Tools
      verbose: boolean
      isTranscriptMode?: boolean
      isBriefOnly?: boolean
      input?: unknown
    },
  ): React.ReactNode
  
  renderToolUseProgressMessage?(
    progressMessagesForMessage: ProgressMessage<P>[],
    options: {
      tools: Tools
      verbose: boolean
      terminalSize?: { columns: number; rows: number }
      inProgressToolCallCount?: number
      isTranscriptMode?: boolean
    },
  ): React.ReactNode
  
  // ============ 序列化 ============
  mapToolResultToToolResultBlockParam(
    content: Output,
    toolUseID: string,
  ): ToolResultBlockParam  // 转换为 API 格式
  
  // ============ 其他 ============
  userFacingName(input): string    // 用户可见名称
  userFacingNameBackgroundColor?(input): keyof Theme | undefined
  toAutoClassifierInput(input): unknown  // 自动模式分类器输入
  getToolUseSummary?(input): string | null  // 工具使用摘要
  getActivityDescription?(input): string | null  // 活动描述（用于 Spinner）
}
```

#### buildTool 辅助函数

```typescript
import { buildTool } from './Tool.js'

// buildTool 会自动填充默认值
const myTool = buildTool({
  name: 'my_tool',
  description: async () => '工具描述',
  inputSchema: z.object({ path: z.string() }),
  
  async call(input, context, canUseTool, parentMessage) {
    // 实现
    return { data: result }
  },
  
  // 以下会自动填充默认值：
  // isEnabled: () => true
  // isConcurrencySafe: () => false
  // isReadOnly: () => false
  // checkPermissions: () => Promise.resolve({ behavior: 'allow' })
  // userFacingName: () => name
})
```

---

## 工具系统

### 内置工具列表

位于 `src/tools/` 目录：

| 工具 | 文件 | 功能 | 并发安全 |
|------|------|------|----------|
| BashTool | `BashTool/` | 执行 Bash 命令 | 是 |
| FileReadTool | `FileReadTool/` | 读取文件 | 是 |
| FileEditTool | `FileEditTool/` | 编辑文件（diff 格式） | 否 |
| FileWriteTool | `FileWriteTool/` | 写入文件 | 否 |
| GlobTool | `GlobTool/` | 文件搜索 | 是 |
| GrepTool | `GrepTool/` | 文本搜索 | 是 |
| AgentTool | `AgentTool/` | 创建子 Agent | 否 |
| WebSearchTool | `WebSearchTool/` | 网页搜索 | 是 |
| WebFetchTool | `WebFetchTool/` | 获取网页内容 | 是 |
| TodoWriteTool | `TodoWriteTool/` | 任务列表管理 | 否 |
| AskUserQuestionTool | `AskUserQuestionTool/` | 询问用户 | 否 |
| NotebookEditTool | `NotebookEditTool/` | 编辑 Jupyter Notebook | 否 |
| EnterPlanModeTool | `EnterPlanModeTool/` | 进入计划模式 | 否 |
| ExitPlanModeTool | `ExitPlanModeTool/` | 退出计划模式 | 否 |
| TaskCreateTool | `TaskCreateTool/` | 创建后台任务 | 否 |
| TaskStopTool | `TaskStopTool/` | 停止后台任务 | 否 |
| ... | ... | ... | ... |

### 工具目录结构

每个工具通常包含以下文件：

```
src/tools/ToolName/
├── ToolName.ts        # 主实现
├── UI.tsx             # UI 组件（可选）
├── prompt.ts          # 工具提示词
├── constants.ts       # 常量定义
├── types.ts           # 类型定义（可选）
└── utils.ts           # 工具函数（可选）
```

### 创建自定义工具 - 完整示例

#### 1. 创建工具目录

```bash
mkdir -p src/tools/MyCustomTool
touch src/tools/MyCustomTool/MyCustomTool.ts
touch src/tools/MyCustomTool/prompt.ts
touch src/tools/MyCustomTool/UI.tsx
```

#### 2. 编写提示词 (prompt.ts)

```typescript
// src/tools/MyCustomTool/prompt.ts
export const MY_CUSTOM_TOOL_PROMPT = `
## my_custom_tool

Perform custom analysis on the provided data. This tool can process files, extract insights, and generate reports.

### When to use
- When you need to analyze code patterns
- When generating project statistics
- When extracting documentation from source files

### Parameters
- path: (string, required) Path to the file or directory to analyze
- format: ("json" | "markdown" | "text", optional) Output format. Defaults to "markdown"
- depth: (number, optional) Analysis depth (1-5). Higher values are more thorough but slower. Defaults to 3

### Example Usage
<example>
<tool>my_custom_tool</tool>
<path>./src</path>
<format>markdown</format>
<depth>3</depth>
</example>

### Output
Returns a structured analysis report in the requested format.

### Notes
- Large directories may take time to analyze
- Results are cached for 5 minutes
`.trim()
```

#### 3. 实现工具 (MyCustomTool.ts)

```typescript
// src/tools/MyCustomTool/MyCustomTool.ts
import { z } from 'zod/v4'
import { buildTool, type ToolUseContext, type ToolResult } from '../../Tool.js'
import { MY_CUSTOM_TOOL_PROMPT } from './prompt.js'
import { MyCustomToolUI } from './UI.js'
import { readFile, readdir } from 'fs/promises'
import { join } from 'path'

// 定义输入模式
const MyCustomToolInputSchema = z.object({
  path: z.string().describe('Path to the file or directory to analyze'),
  format: z.enum(['json', 'markdown', 'text']).optional().describe('Output format'),
  depth: z.number().min(1).max(5).optional().describe('Analysis depth (1-5)'),
})

type MyCustomToolInput = z.infer<typeof MyCustomToolInputSchema>

type AnalysisResult = {
  filesAnalyzed: number
  totalLines: number
  codeLines: number
  commentLines: number
  blankLines: number
  complexity: number
  findings: Array<{
    file: string
    line: number
    message: string
    severity: 'info' | 'warning' | 'error'
  }>
}

async function analyzeFile(filePath: string, depth: number): Promise<Partial<AnalysisResult>> {
  const content = await readFile(filePath, 'utf-8')
  const lines = content.split('\n')
  
  let codeLines = 0
  let commentLines = 0
  let blankLines = 0
  const findings: AnalysisResult['findings'] = []
  
  for (let i = 0; i < lines.length; i++) {
    const line = lines[i]!.trim()
    
    if (line === '') {
      blankLines++
    } else if (line.startsWith('//') || line.startsWith('/*') || line.startsWith('*')) {
      commentLines++
    } else {
      codeLines++
    }
    
    // 深度分析
    if (depth >= 3 && line.length > 100) {
      findings.push({
        file: filePath,
        line: i + 1,
        message: 'Line exceeds 100 characters',
        severity: 'warning',
      })
    }
  }
  
  return {
    filesAnalyzed: 1,
    totalLines: lines.length,
    codeLines,
    commentLines,
    blankLines,
    findings,
  }
}

async function analyzeDirectory(
  dirPath: string, 
  depth: number,
  onProgress?: (message: string) => void
): Promise<AnalysisResult> {
  const entries = await readdir(dirPath, { withFileTypes: true })
  const result: AnalysisResult = {
    filesAnalyzed: 0,
    totalLines: 0,
    codeLines: 0,
    commentLines: 0,
    blankLines: 0,
    complexity: 0,
    findings: [],
  }
  
  for (const entry of entries) {
    const fullPath = join(dirPath, entry.name)
    
    if (entry.isDirectory() && !entry.name.startsWith('.') && entry.name !== 'node_modules') {
      if (depth >= 2) {
        onProgress?.(`Analyzing directory: ${fullPath}`)
        const subResult = await analyzeDirectory(fullPath, depth, onProgress)
        mergeResults(result, subResult)
      }
    } else if (entry.isFile() && entry.name.endsWith('.ts')) {
      onProgress?.(`Analyzing file: ${fullPath}`)
      const fileResult = await analyzeFile(fullPath, depth)
      mergeResults(result, fileResult)
    }
  }
  
  return result
}

function mergeResults(target: AnalysisResult, source: Partial<AnalysisResult>) {
  target.filesAnalyzed += source.filesAnalyzed ?? 0
  target.totalLines += source.totalLines ?? 0
  target.codeLines += source.codeLines ?? 0
  target.commentLines += source.commentLines ?? 0
  target.blankLines += source.blankLines ?? 0
  if (source.findings) {
    target.findings.push(...source.findings)
  }
}

function formatResult(result: AnalysisResult, format: string): string {
  switch (format) {
    case 'json':
      return JSON.stringify(result, null, 2)
      
    case 'markdown':
      return `# Analysis Report

## Summary
- **Files Analyzed**: ${result.filesAnalyzed}
- **Total Lines**: ${result.totalLines}
- **Code Lines**: ${result.codeLines}
- **Comment Lines**: ${result.commentLines}
- **Blank Lines**: ${result.blankLines}

## Statistics
- **Code/Comment Ratio**: ${(result.codeLines / (result.commentLines || 1)).toFixed(2)}
- **Average File Size**: ${(result.totalLines / result.filesAnalyzed).toFixed(0)} lines

## Findings (${result.findings.length})
${result.findings.map(f => `- [${f.severity.toUpperCase()}] ${f.file}:${f.line} - ${f.message}`).join('\n') || 'No issues found.'}
`
      
    default:
      return `Analyzed ${result.filesAnalyzed} files, ${result.totalLines} lines total`
  }
}

export const MyCustomTool = buildTool({
  name: 'my_custom_tool',
  description: async () => MY_CUSTOM_TOOL_PROMPT,
  searchHint: 'analyze code statistics and patterns',
  
  inputSchema: MyCustomToolInputSchema,
  
  // 执行逻辑
  async call(
    input, 
    context, 
    canUseTool, 
    parentMessage, 
    onProgress
  ): Promise<ToolResult<AnalysisResult>> {
    const { path: targetPath, format = 'markdown', depth = 3 } = input
    
    // 1. 权限检查
    const permission = await canUseTool(
      MyCustomTool,
      input,
      context,
      parentMessage,
      context.toolUseId ?? 'unknown'
    )
    
    if (permission.behavior !== 'allow') {
      return { 
        data: {
          filesAnalyzed: 0,
          totalLines: 0,
          codeLines: 0,
          commentLines: 0,
          blankLines: 0,
          complexity: 0,
          findings: [{
            file: targetPath,
            line: 0,
            message: 'Permission denied',
            severity: 'error'
          }],
        }
      }
    }
    
    // 2. 执行分析
    const stats = await context.getAppState().fs.stat(targetPath)
    
    let result: AnalysisResult
    if (stats.isFile()) {
      const fileResult = await analyzeFile(targetPath, depth)
      result = {
        filesAnalyzed: 1,
        totalLines: fileResult.totalLines ?? 0,
        codeLines: fileResult.codeLines ?? 0,
        commentLines: fileResult.commentLines ?? 0,
        blankLines: fileResult.blankLines ?? 0,
        complexity: 0,
        findings: fileResult.findings ?? [],
      }
    } else {
      result = await analyzeDirectory(targetPath, depth, (message) => {
        onProgress?.({
          toolUseID: context.toolUseId ?? 'unknown',
          data: { type: 'my_custom_tool_progress', message }
        })
      })
    }
    
    // 3. 返回结果
    return { data: result }
  },
  
  // 是否支持并发
  isConcurrencySafe: () => true,
  
  // 是否只读
  isReadOnly: () => true,
  
  // 权限检查
  async checkPermissions(input, context) {
    // 检查路径是否在允许范围内
    const { path: targetPath } = input
    const fs = context.getAppState().fs
    
    try {
      const resolved = await fs.resolvePath(targetPath)
      return { behavior: 'allow', updatedInput: { ...input, path: resolved } }
    } catch (error) {
      return { 
        behavior: 'deny', 
        updatedInput: input,
        message: `Cannot access path: ${targetPath}`
      }
    }
  },
  
  // 渲染工具使用消息
  renderToolUseMessage(input, { theme, verbose }) {
    return `Analyzing ${input.path}${input.format ? ` (${input.format})` : ''}`
  },
  
  // 渲染结果消息
  renderToolResultMessage(content, progressMessages, { theme, verbose, input }) {
    // 使用单独的 UI 组件
    return MyCustomToolUI({ content, format: input?.format ?? 'markdown' })
  },
  
  // 映射到 API 格式
  mapToolResultToToolResultBlockParam(content, toolUseID) {
    return {
      type: 'tool_result' as const,
      tool_use_id: toolUseID,
      content: JSON.stringify(content),
    }
  },
  
  // 用户可见名称
  userFacingName: (input) => `分析: ${input?.path ?? ''}`,
  
  // 用户可见名称颜色
  userFacingNameBackgroundColor: () => 'cyan',
  
  // 自动分类器输入
  toAutoClassifierInput: (input) => input.path ?? '',
  
  // 工具使用摘要
  getToolUseSummary: (input) => `Analyzed ${input?.path ?? 'unknown path'}`,
  
  // 活动描述
  getActivityDescription: (input) => `Analyzing ${input?.path ?? ''}`,
})
```

#### 4. 创建 UI 组件 (UI.tsx)

```typescript
// src/tools/MyCustomTool/UI.tsx
import React from 'react'
import { Box, Text } from '../../ink.js'
import type { AnalysisResult } from './MyCustomTool.js'

export function MyCustomToolUI({ 
  content, 
  format 
}: { 
  content: AnalysisResult 
  format: string 
}) {
  if (content.findings.some(f => f.severity === 'error')) {
    return (
      <Box flexDirection="column">
        <Text color="red">❌ Analysis completed with errors</Text>
        <Text>Files: {content.filesAnalyzed} | Lines: {content.totalLines}</Text>
      </Box>
    )
  }
  
  return (
    <Box flexDirection="column">
      <Text color="green">✓ Analysis complete</Text>
      <Text>Files analyzed: {content.filesAnalyzed}</Text>
      <Text>Total lines: {content.totalLines}</Text>
      <Text>Code lines: {content.codeLines}</Text>
      <Text>Comments: {content.commentLines}</Text>
      {content.findings.length > 0 && (
        <Box marginTop={1}>
          <Text color="yellow">⚠ {content.findings.length} findings</Text>
        </Box>
      )}
    </Box>
  )
}
```

#### 5. 注册工具

在 `src/tools.ts` 中添加：

```typescript
import { MyCustomTool } from './tools/MyCustomTool/MyCustomTool.js'

export function getAllBaseTools(): Tools {
  return [
    // ... 现有工具
    MyCustomTool,  // 添加你的工具
  ]
}
```

---

## 命令系统

### 命令类型详解

```typescript
// src/types/command.ts

// ============ Prompt 命令 ============
// 展开为提示词发送给模型
// 示例: /help -> "请帮我了解 Claude Code 的功能"
type PromptCommand = {
  type: 'prompt'
  name: string
  aliases?: string[]
  description: string
  source: 'builtin' | 'plugin' | 'mcp' | 'skills' | 'bundled'
  
  // 展开为发送给模型的提示词
  getPromptForCommand(
    args: string,  // 命令后的参数字符串
    context: CommandContext
  ): Promise<string | { content: string; attachments?: Attachment[] }>
  
  // 可选：自定义是否启用
  isEnabled?(): boolean
  
  // 可选：限制可用性（需要 claude-ai 订阅或 console API）
  availability?: Array<'claude-ai' | 'console'>
  
  // 可选：模型调用限制
  disableModelInvocation?: boolean
  
  // 可选：何时使用说明
  whenToUse?: string
}

// ============ Local 命令 ============
// 本地执行，返回文本结果
// 示例: /cost -> "当前会话已使用 $0.12"
type LocalCommand = {
  type: 'local'
  name: string
  description: string
  
  // 执行命令
  execute(
    args: string,
    context: CommandContext
  ): Promise<string>
}

// ============ Local-JSX 命令 ============
// 本地执行，渲染 React 组件
// 示例: /model -> 打开模型选择器对话框
type LocalJSXCommand = {
  type: 'local-jsx'
  name: string
  description: string
  
  // React 组件
  component: React.FC<{
    args: string
    context: CommandContext
    onDone: () => void  // 完成后调用
  }>
}
```

### CommandContext

```typescript
type CommandContext = {
  // 当前工作目录
  cwd: string
  
  // 应用状态
  getAppState(): AppState
  setAppState(updater: (prev: AppState) => AppState): void
  
  // 工具上下文（prompt 命令需要）
  toolUseContext?: ToolUseContext
  
  // 其他工具函数
  clearScreen(): void
  exit(code?: number): never
  // ...
}
```

### 创建自定义命令 - 完整示例

#### Prompt 命令示例

```typescript
// src/commands/code-review/index.ts
import type { Command } from '../../types/command.js'

const codeReviewCommand: Command = {
  type: 'prompt',
  name: 'code-review',
  aliases: ['cr'],
  description: 'Request a code review for the current changes',
  source: 'builtin',
  whenToUse: 'Use when you want feedback on code changes before committing',
  
  async getPromptForCommand(args, context) {
    const { cwd } = context
    
    // 获取 git diff
    const { execSync } = await import('child_process')
    let diff = ''
    try {
      diff = execSync('git diff --cached', { cwd, encoding: 'utf-8' })
      if (!diff) {
        diff = execSync('git diff', { cwd, encoding: 'utf-8' })
      }
    } catch {
      return 'Please run this command in a git repository with changes.'
    }
    
    if (!diff) {
      return 'No changes to review. Stage some changes with git add first.'
    }
    
    return `Please review the following code changes. Focus on:
1. Code quality and best practices
2. Potential bugs or issues
3. Performance implications
4. Security concerns
5. Suggestions for improvement

Here is the diff:

\`\`\`diff
${diff.slice(0, 50000)}  // 限制大小
\`\`\`

${args ? `Additional context: ${args}` : ''}`
  },
}

export default codeReviewCommand
```

#### Local 命令示例

```typescript
// src/commands/session-info/index.ts
import type { Command } from '../../types/command.js'
import { getSessionId } from '../../bootstrap/state.js'
import { getTotalCost } from '../../cost-tracker.js'

const sessionInfoCommand: Command = {
  type: 'local',
  name: 'session-info',
  aliases: ['info'],
  description: 'Display current session information',
  
  async execute(args, context) {
    const { getAppState } = context
    const appState = getAppState()
    const sessionId = getSessionId()
    const cost = getTotalCost()
    const messageCount = appState.messages.length
    
    const lines = [
      '═══════════════════════════════════',
      '         Session Information        ',
      '═══════════════════════════════════',
      `Session ID: ${sessionId}`,
      `Messages: ${messageCount}`,
      `Total Cost: $${cost.toFixed(4)}`,
      `Model: ${appState.mainLoopModel}`,
      `Permission Mode: ${appState.toolPermissionContext.mode}`,
      '',
      '═══════════════════════════════════',
    ]
    
    return lines.join('\n')
  },
}

export default sessionInfoCommand
```

#### Local-JSX 命令示例

```typescript
// src/commands/theme-picker/index.ts
import type { Command } from '../../types/command.js'
import React, { useState } from 'react'
import { Box, Text, useInput } from '../../ink.js'
import { getGlobalConfig, saveGlobalConfig } from '../../utils/config.js'

const THEME_OPTIONS = [
  { name: 'Dark', value: 'dark' },
  { name: 'Light', value: 'light' },
  { name: 'High Contrast', value: 'high-contrast' },
  { name: 'Dracula', value: 'dracula' },
  { name: 'Solarized', value: 'solarized' },
] as const

const ThemePickerComponent: Command['component'] = ({ args, context, onDone }) => {
  const [selectedIndex, setSelectedIndex] = useState(() => {
    const current = getGlobalConfig().theme
    const index = THEME_OPTIONS.findIndex(t => t.value === current)
    return index >= 0 ? index : 0
  })
  
  useInput((input, key) => {
    if (key.upArrow) {
      setSelectedIndex(i => (i > 0 ? i - 1 : THEME_OPTIONS.length - 1))
    } else if (key.downArrow) {
      setSelectedIndex(i => (i < THEME_OPTIONS.length - 1 ? i + 1 : 0))
    } else if (key.return) {
      const theme = THEME_OPTIONS[selectedIndex]!.value
      saveGlobalConfig(prev => ({ ...prev, theme }))
      onDone()
    } else if (key.escape) {
      onDone()
    }
  })
  
  return (
    <Box flexDirection="column" padding={1}>
      <Text bold>Select Theme:</Text>
      <Box flexDirection="column" marginTop={1}>
        {THEME_OPTIONS.map((theme, index) => (
          <Box key={theme.value}>
            <Text color={index === selectedIndex ? 'green' : undefined}>
              {index === selectedIndex ? '▶ ' : '  '}
              {theme.name}
            </Text>
          </Box>
        ))}
      </Box>
      <Box marginTop={1}>
        <Text dimColor>↑↓ to select, Enter to confirm, Esc to cancel</Text>
      </Box>
    </Box>
  )
}

const themePickerCommand: Command = {
  type: 'local-jsx',
  name: 'theme-picker',
  aliases: ['theme'],
  description: 'Interactively select a color theme',
  component: ThemePickerComponent,
}

export default themePickerCommand
```

### 注册命令

在 `src/commands.ts` 中：

```typescript
// ... 现有导入
import codeReviewCommand from './commands/code-review/index.js'
import sessionInfoCommand from './commands/session-info/index.js'
import themePickerCommand from './commands/theme-picker/index.js'

const COMMANDS = memoize((): Command[] => [
  // ... 现有命令
  codeReviewCommand,
  sessionInfoCommand,
  themePickerCommand,
])
```

---

## 状态管理

### 应用状态结构

**文件**: `src/state/AppState.tsx`

```typescript
type AppState = {
  // ═══════════════════════════════════════════════════════════
  // 核心消息状态
  // ═══════════════════════════════════════════════════════════
  messages: Message[]              // 对话消息历史
  pendingTools: Set<string>        // 正在执行的工具 ID
  inProgressToolUseIDs: Set<string>  // 进行中的工具调用
  
  // ═══════════════════════════════════════════════════════════
  // MCP 相关状态
  // ═══════════════════════════════════════════════════════════
  mcp: {
    clients: MCPServerConnection[]    // MCP 服务器连接
    tools: Tools                       // MCP 提供的工具
    commands: Command[]                // MCP 提供的命令
    resources: Record<string, ServerResource[]>  // MCP 资源
  }
  
  // ═══════════════════════════════════════════════════════════
  // 权限状态
  // ═══════════════════════════════════════════════════════════
  toolPermissionContext: ToolPermissionContext
  pendingPermissions: Map<string, PendingPermission>
  
  // ═══════════════════════════════════════════════════════════
  // 输入/队列状态
  // ═══════════════════════════════════════════════════════════
  inputQueue: InputQueueItem[]     // 待处理的输入
  isInputFocused: boolean          // 输入框是否聚焦
  inputBuffer: string              // 当前输入缓冲
  
  // ═══════════════════════════════════════════════════════════
  // UI 状态
  // ═══════════════════════════════════════════════════════════
  isGlobalSearchOpen: boolean      // 全局搜索是否打开
  isHistorySearchOpen: boolean     // 历史搜索是否打开
  showVerboseOutput: boolean       // 是否显示详细输出
  
  // ═══════════════════════════════════════════════════════════
  // 会话状态
  // ═══════════════════════════════════════════════════════════
  mainLoopModel: string            // 当前使用的模型
  fastMode: boolean                // 快速模式是否开启
  effortValue: number              // 努力值 (0-3)
  
  // ═══════════════════════════════════════════════════════════
  // Agent 状态
  // ═══════════════════════════════════════════════════════════
  activeAgents: AgentInstance[]    // 活跃的子 Agent
  
  // ═══════════════════════════════════════════════════════════
  // 文件历史
  // ═══════════════════════════════════════════════════════════
  fileHistory: FileHistoryState     // 文件修改历史
  
  // ... 更多状态
}
```

### 使用状态

```typescript
import { useAppState } from './state/AppState.js'

function MyComponent() {
  const [appState, setAppState] = useAppState()
  
  // 读取状态
  const messages = appState.messages
  const isInputFocused = appState.isInputFocused
  
  // 更新状态（函数式更新）
  const handleSomething = () => {
    setAppState(prev => ({
      ...prev,
      someField: newValue
    }))
  }
  
  // 批量更新
  const handleMultipleChanges = () => {
    setAppState(prev => {
      const newState = { ...prev }
      newState.messages = [...prev.messages, newMessage]
      newState.pendingTools = new Set(prev.pendingTools)
      newState.pendingTools.add(toolId)
      return newState
    })
  }
}
```

### 状态选择器

为了避免不必要的重渲染，使用选择器模式：

```typescript
import { useAppStateSelector } from './state/AppState.js'

function MessageCount() {
  // 只在 messageCount 变化时重渲染
  const messageCount = useAppStateSelector(state => state.messages.length)
  return <Text>Messages: {messageCount}</Text>
}

function CurrentModel() {
  // 只在 mainLoopModel 变化时重渲染
  const model = useAppStateSelector(state => state.mainLoopModel)
  return <Text>Model: {model}</Text>
}
```

---

## UI 组件

### Ink 基础

Ink 是 React 的终端渲染器。组件与 React 类似，但使用终端特定的布局系统。

#### 核心组件

```typescript
import { Box, Text, useInput, useApp } from '../ink.js'

function MyComponent() {
  const { exit } = useApp()  // 控制应用退出
  
  // 处理键盘输入
  useInput((input, key) => {
    if (key.escape) {
      exit()
    }
    if (input === 'q') {
      exit()
    }
  })
  
  return (
    <Box 
      flexDirection="column"      // 或 "row"
      justifyContent="center"      // flex 布局属性
      alignItems="center"
      padding={1}                  // 内边距
      margin={1}                   // 外边距
      borderStyle="round"          // single, double, round
      borderColor="green"          // 颜色
    >
      <Text 
        color="blue"
        backgroundColor="white"
        bold
        italic
        underline
        strikethrough
        wrap="wrap"                // 文本换行
      >
        Hello, World!
      </Text>
    </Box>
  )
}
```

#### 颜色系统

```typescript
import { Text } from '../ink.js'
import { useTheme } from '../utils/theme.js'

function ColoredText() {
  const theme = useTheme()
  
  return (
    <>
      {/* 主题颜色 */}
      <Text color={theme.primary}>Primary</Text>
      <Text color={theme.secondary}>Secondary</Text>
      <Text color={theme.success}>Success</Text>
      <Text color={theme.warning}>Warning</Text>
      <Text color={theme.error}>Error</Text>
      
      {/* 标准颜色 */}
      <Text color="black">Black</Text>
      <Text color="red">Red</Text>
      <Text color="green">Green</Text>
      <Text color="yellow">Yellow</Text>
      <Text color="blue">Blue</Text>
      <Text color="magenta">Magenta</Text>
      <Text color="cyan">Cyan</Text>
      <Text color="white">White</Text>
      <Text color="gray">Gray</Text>
      
      {/* 亮色系 */}
      <Text color="redBright">Bright Red</Text>
      {/* ... */}
    </>
  )
}
```

#### 布局技巧

```typescript
import { Box, Text, Spacer } from '../ink.js'

// 左右布局
function Header() {
  return (
    <Box flexDirection="row" justifyContent="space-between">
      <Text>Left</Text>
      <Text>Right</Text>
    </Box>
  )
}

// 使用 Spacer
function HeaderWithSpacer() {
  return (
    <Box flexDirection="row">
      <Text>Left</Text>
      <Spacer />  {/* 占据剩余空间 */}
      <Text>Right</Text>
    </Box>
  )
}

// 固定宽度列
function Columns() {
  return (
    <Box flexDirection="row">
      <Box width={20}><Text>Column 1</Text></Box>
      <Box width={30}><Text>Column 2</Text></Box>
      <Box><Text>Flexible</Text></Box>
    </Box>
  )
}

// 百分比宽度
function PercentageWidth() {
  const { columns } = useTerminalSize()
  
  return (
    <Box flexDirection="row">
      <Box width="30%"><Text>30%</Text></Box>
      <Box width="70%"><Text>70%</Text></Box>
    </Box>
  )
}
```

### 常用 Hooks

```typescript
import { 
  useInput,           // 键盘输入
  useTerminalSize,    // 终端尺寸
  useApp,             // 应用控制
  useStdin,           // stdin 流
  useStdout,          // stdout 流
} from '../hooks/index.js'

// ═══════════════════════════════════════════════════════════
// useInput - 键盘处理
// ═══════════════════════════════════════════════════════════
function InteractiveComponent() {
  const [count, setCount] = useState(0)
  
  useInput((input, key) => {
    if (key.upArrow) setCount(c => c + 1)
    if (key.downArrow) setCount(c => c - 1)
    if (key.return) console.log('Selected:', count)
    if (key.escape) exit()
    
    // 处理普通字符
    if (input === 'r') setCount(0)
  })
  
  return <Text>Count: {count}</Text>
}

// ═══════════════════════════════════════════════════════════
// useTerminalSize - 响应式布局
// ═══════════════════════════════════════════════════════════
function ResponsiveComponent() {
  const { columns, rows } = useTerminalSize()
  
  return (
    <Box flexDirection="column">
      <Text>Terminal: {columns} x {rows}</Text>
      <Box width={columns - 4}>
        {/* 自适应内容 */}
      </Box>
    </Box>
  )
}

// ═══════════════════════════════════════════════════════════
// useApp - 应用控制
// ═══════════════════════════════════════════════════════════
function AppController() {
  const { exit, focusPrevious, focusNext } = useApp()
  
  useInput((_, key) => {
    if (key.tab) {
      key.shift ? focusPrevious() : focusNext()
    }
  })
  
  return <Text>Press Tab to switch focus, Ctrl+C to exit</Text>
}
```

### 组件示例 - 完整对话框

```typescript
// src/components/ConfirmDialog.tsx
import React, { useState } from 'react'
import { Box, Text, useInput } from '../ink.js'

export type ConfirmDialogProps = {
  title: string
  message: string
  onConfirm: () => void
  onCancel: () => void
  confirmLabel?: string
  cancelLabel?: string
  danger?: boolean  // 危险操作（红色）
}

export function ConfirmDialog({
  title,
  message,
  onConfirm,
  onCancel,
  confirmLabel = 'Yes',
  cancelLabel = 'No',
  danger = false,
}: ConfirmDialogProps) {
  const [selected, setSelected] = useState<'confirm' | 'cancel'>('cancel')
  
  useInput((_, key) => {
    if (key.leftArrow || key.rightArrow) {
      setSelected(s => s === 'confirm' ? 'cancel' : 'confirm')
    }
    if (key.return) {
      selected === 'confirm' ? onConfirm() : onCancel()
    }
    if (key.escape) {
      onCancel()
    }
  })
  
  const confirmColor = danger ? 'red' : 'green'
  
  return (
    <Box 
      flexDirection="column" 
      borderStyle="round"
      borderColor={danger ? 'red' : 'blue'}
      padding={1}
    >
      <Text bold color={danger ? 'red' : 'blue'}>{title}</Text>
      <Box marginY={1}>
        <Text wrap="wrap">{message}</Text>
      </Box>
      
      <Box flexDirection="row" gap={2}>
        <Text 
          color={selected === 'confirm' ? confirmColor : undefined}
          backgroundColor={selected === 'confirm' ? 'gray' : undefined}
        >
          {selected === 'confirm' ? '▶ ' : '  '}
          {confirmLabel}
        </Text>
        <Text 
          color={selected === 'cancel' ? 'blue' : undefined}
          backgroundColor={selected === 'cancel' ? 'gray' : undefined}
        >
          {selected === 'cancel' ? '▶ ' : '  '}
          {cancelLabel}
        </Text>
      </Box>
      
      <Box marginTop={1}>
        <Text dimColor>← → to select, Enter to confirm, Esc to cancel</Text>
      </Box>
    </Box>
  )
}
```

---

## 功能开关系统

### 编译时开关 (feature)

**使用**: `feature('FLAG_NAME')`

```typescript
import { feature } from 'bun:bundle'

// 条件代码
if (feature('KAIROS')) {
  // KAIROS 功能代码
}

// 条件导入
const assistantModule = feature('KAIROS') 
  ? require('./assistant/index.js') 
  : null
```

### 常用功能开关

| 开关 | 说明 | 相关命令/功能 |
|------|------|---------------|
| `BUDDY` | 宠物伴侣系统 | `/buddy` |
| `KAIROS` | 持久助手模式 | `/assistant`, `/brief`, 自动做梦 |
| `KAIROS_BRIEF` | 简报模式 | `/brief` |
| `COORDINATOR_MODE` | 多 Agent 编排 | Coordinator + Worker 模式 |
| `BRIDGE_MODE` | 远程控制桥接 | `/bridge`, 远程控制 |
| `PROACTIVE` | 主动自主模式 | `/proactive` |
| `ULTRAPLAN` | 云端深度规划 | `/ultraplan` |
| `MCP_SKILLS` | MCP 技能系统 | MCP 提供的技能 |
| `HISTORY_SNIP` | 历史截断 | 自动截断早期历史 |
| `CONTEXT_COLLAPSE` | 上下文折叠 | 智能上下文管理 |
| `REACTIVE_COMPACT` | 响应式压缩 | 实时上下文压缩 |
| `CACHED_MICROCOMPACT` | 缓存微压缩 | 微压缩缓存优化 |
| `VOICE_MODE` | 语音交互 | `/voice` |
| `WORKFLOW_SCRIPTS` | 工作流脚本 | `/workflows` |
| `FORK_SUBAGENT` | 子代理分叉 | `/fork` |
| `AGENT_TRIGGERS` | Agent 触发器 | Cron 任务创建 |
| `UDS_INBOX` | Unix Socket 收件箱 | `/peers` |
| `TORCH` | Torch 功能 | `/torch` |
| `MONITOR_TOOL` | 监控工具 | 性能监控 |
| `EXPERIMENTAL_SKILL_SEARCH` | 实验性技能搜索 | 智能技能发现 |

### 运行时开关 (GrowthBook)

```typescript
import { 
  getFeatureValue,
  getFeatureValue_CACHED_MAY_BE_STALE 
} from './services/analytics/growthbook.js'

// 获取功能值（带默认值）
const value = getFeatureValue('tengu_feature_name', defaultValue)

// 获取缓存值（性能更好，但可能不是最新的）
const cachedValue = getFeatureValue_CACHED_MAY_BE_STALE('tengu_feature_name', defaultValue)

// 常用 GrowthBook 开关
const config = {
  kairosEnabled: getFeatureValue('tengu_kairos', false),
  kairosThreshold: getFeatureValue('tengu_onyx_plover', { interval: 24, sessions: 5 }),
  voiceEnabled: getFeatureValue('tengu_cobalt_frost', false),
  ultraplanModel: getFeatureValue('tengu_ultraplan_model', 'opus'),
  maxVersionConfig: getFeatureValue('tengu_max_version_config', null),
}
```

### 用户类型检查

```typescript
// 检查是否为内部用户
const isAnt = process.env.USER_TYPE === 'ant'

// 检查是否为外部用户
const isExternal = process.env.USER_TYPE === 'external'

// 条件导入（Ant 专用功能）
const antOnlyTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/AntOnlyTool.js').AntOnlyTool
  : null
```

---

## 修改指南

### 场景 1: 修改现有工具

以修改 `BashTool` 添加新的安全规则为例：

1. **找到工具文件**: `src/tools/BashTool/BashTool.ts`

2. **修改执行逻辑**:

```typescript
// 在 BashTool.ts 中找到 call 方法
async call(input, context, canUseTool, parentMessage, onProgress) {
  // ... 现有代码
  
  // 添加新的安全检查
  if (isDangerousCommand(input.command)) {
    return {
      data: {
        output: '',
        exitCode: 1,
        error: 'This command is not allowed for security reasons'
      }
    }
  }
  
  // ... 继续执行
}

// 添加新的安全函数
function isDangerousCommand(command: string): boolean {
  const dangerousPatterns = [
    /rm\s+-rf\s+\//,           // 删除根目录
    /mkfs\.\w+\s+\/dev\/sd/,   // 格式化磁盘
    /:(){ :|:& };:/,           // Fork bomb
    /dd\s+if=.*of=\/dev\/sd/,  // 直接写磁盘
  ]
  
  return dangerousPatterns.some(pattern => pattern.test(command))
}
```

3. **修改提示词**: 编辑 `src/tools/BashTool/prompt.ts`

4. **测试**:
   ```bash
   bun run dev
   # 测试: "请帮我删除 / 目录下的所有文件" - 应该被拒绝
   ```

### 场景 2: 添加新命令

完整流程见 [命令系统](#命令系统) 章节

### 场景 3: 修改系统提示词

1. **找到系统提示词文件**:
   - 主要: `src/constants/prompts.ts`
   - 工具特定: `src/tools/*/prompt.ts`

2. **修改示例**:

```typescript
// src/constants/prompts.ts

// 添加新的系统提示词段落
export const MY_CUSTOM_SYSTEM_PROMPT = `
## Custom Guidelines

When working with this project:
1. Always follow the existing code style
2. Add comments for complex logic
3. Write tests for new features
4. Update documentation when changing APIs
`

// 在 fetchSystemPromptParts 中使用
export async function fetchSystemPromptParts(options: {
  tools: Tools
  mainLoopModel: string
  // ...
}) {
  const defaultSystemPrompt = [
    // ... 现有提示词
    MY_CUSTOM_SYSTEM_PROMPT,
  ]
  
  return { defaultSystemPrompt, /* ... */ }
}
```

### 场景 4: 启用隐藏功能

某些功能需要设置环境变量：

```bash
# 启用内部功能
export USER_TYPE=ant

# 启用特定功能
export CLAUDE_CODE_PROACTIVE=1
export CLAUDE_CODE_COORDINATOR_MODE=1
export CLAUDE_CODE_BRIEF=1

bun run dev
```

### 场景 5: 修改 UI 样式

```typescript
// src/components/MyComponent.tsx

import { useTheme } from '../utils/theme.js'

function MyComponent() {
  const theme = useTheme()
  
  return (
    <Box 
      borderStyle="round"
      borderColor={theme.primary}  // 使用主题色
      padding={2}                  // 增加内边距
    >
      <Text 
        bold
        color={theme.success}
        backgroundColor="black"
      >
        自定义样式文本
      </Text>
    </Box>
  )
}
```

### 场景 6: 添加新的事件 Hook

```typescript
// src/constants/hooks.ts

// 1. 添加新的事件类型
export const HOOK_EVENTS = [
  // ... 现有事件
  'my_custom_event',
] as const

// 2. 定义事件参数
export type MyCustomEventParams = {
  data: string
  timestamp: number
}

// src/utils/hooks/myCustomHooks.ts
import { logEvent } from '../../services/analytics/index.js'

export async function executeMyCustomHooks(
  params: MyCustomEventParams,
  context: HookContext,
): Promise<HookResult> {
  const hooks = getHooksForEvent('my_custom_event')
  
  for (const hook of hooks) {
    const result = await executeHook(hook, params, context)
    if (result.error) {
      logEvent('tengu_my_custom_hook_error', { error: result.error })
    }
  }
  
  return { success: true }
}

// 3. 在适当位置触发
// src/someFeature.ts
import { executeMyCustomHooks } from './utils/hooks/myCustomHooks.js'

async function someFunction() {
  // ... 执行某些操作
  
  await executeMyCustomHooks({
    data: 'some data',
    timestamp: Date.now(),
  }, hookContext)
}
```

### 场景 7: 修改 API 调用行为

```typescript
// src/services/api/claude.ts

// 找到 callModel 函数
export async function* callModel(params: CallModelParams) {
  // ...
  
  // 修改请求头
  const headers = {
    ...getDefaultHeaders(),
    'X-Custom-Header': 'my-value',
  }
  
  // 修改请求体
  const requestBody = {
    ...buildRequestBody(params),
    // 添加自定义参数
    custom_field: customValue,
  }
  
  // 添加请求前处理
  console.log('API Request:', JSON.stringify(requestBody, null, 2))
  
  // 执行请求
  const response = await fetch(API_URL, {
    method: 'POST',
    headers,
    body: JSON.stringify(requestBody),
  })
  
  // 添加响应后处理
  console.log('API Response:', response.status)
  
  // ...
}
```

---

## 调试技巧

### 1. 启用调试日志

```bash
# 所有调试输出
bun run dev --debug

# 特定类别（逗号分隔）
bun run dev --debug=api,tools,hooks

# 排除类别（! 前缀）
bun run dev --debug='!1p,!file,!telemetry'

# 输出到文件
bun run dev --debug-file=/tmp/debug.log

# 详细模式（更多输出）
bun run dev --verbose
```

### 2. 使用断点

```typescript
// 在代码中添加 debugger
debugger

// 或使用 console.log
console.log('调试信息:', variable)
console.dir(object, { depth: null })

// 使用专门的调试函数
import { logForDebugging } from './utils/debug.js'
logForDebugging('关键变量值:', value)
```

### 3. 检查状态

```bash
# 查看当前会话状态
cat ~/.claude/projects/$(pwd | tr '/' '_')/sessions/latest.json

# 查看日志
ls -la ~/.claude/logs/
tail -f ~/.claude/logs/latest.log

# 查看全局配置
cat ~/.claude/config.json

# 查看项目配置
cat .claude/CLAUDE.md
```

### 4. 性能分析

```bash
# 启动性能分析
CLAUDE_CODE_PROFILE=1 bun run dev

# 查看分析结果
cat /tmp/claude-profile.json

# 使用分析工具
bun run analyze-profile /tmp/claude-profile.json
```

### 5. API 请求调试

在 `src/services/api/claude.ts` 中添加：

```typescript
// 在 callModel 函数中添加日志
console.log('API Request Body:', JSON.stringify(requestBody, null, 2))

// 使用环境变量控制
if (process.env.CLAUDE_CODE_DUMP_PROMPTS) {
  await writeFile(
    `/tmp/prompt-${Date.now()}.json`,
    JSON.stringify(requestBody, null, 2)
  )
}
```

### 6. 追踪函数调用

```typescript
// 使用装饰器模式
function trace<T extends (...args: any[]) => any>(fn: T, name: string): T {
  return ((...args: any[]) => {
    console.log(`[TRACE] ${name} called with:`, args)
    const start = performance.now()
    const result = fn(...args)
    const end = performance.now()
    console.log(`[TRACE] ${name} completed in ${end - start}ms`)
    return result
  }) as T
}

// 使用
const tracedFunction = trace(originalFunction, 'originalFunction')
```

### 7. 内存调试

```bash
# 启用内存跟踪
CLAUDE_CODE_HEAPDUMP=1 bun run dev

# 生成堆快照（在运行时）
kill -USR2 <pid>

# 分析堆快照
ls *.heapsnapshot
```

---

## 高级主题

### 1. MCP 扩展开发

MCP (Model Context Protocol) 允许 Claude 调用外部工具。

#### 简单的 MCP 服务器

```typescript
// my-mcp-server.ts
#!/usr/bin/env bun
import { Server } from '@modelcontextprotocol/sdk/server/index.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'

const server = new Server(
  { name: 'my-server', version: '1.0.0' },
  {
    capabilities: {
      tools: {
        my_tool: {
          description: 'My custom tool',
          inputSchema: {
            type: 'object',
            properties: {
              input: { type: 'string' },
            },
            required: ['input'],
          },
        },
      },
    },
  }
)

server.setRequestHandler('tools/call', async (request) => {
  if (request.params.name === 'my_tool') {
    const { input } = request.params.arguments
    return {
      content: [{ type: 'text', text: `Result: ${input.toUpperCase()}` }],
    }
  }
  throw new Error('Unknown tool')
})

const transport = new StdioServerTransport()
await server.connect(transport)
```

#### 配置 MCP 服务器

在 `~/.claude/CLAUDE.md` 或项目 `.claude/CLAUDE.md` 中：

```json
{
  "mcpServers": {
    "my-server": {
      "command": "bun",
      "args": ["/path/to/my-mcp-server.ts"],
      "env": {
        "API_KEY": "your-api-key"
      }
    }
  }
}
```

### 2. 子 Agent 系统

AgentTool 允许创建专门的子 Agent：

```typescript
// src/tools/AgentTool/built-in/myAgent.ts
import type { AgentDefinition } from '../loadAgentsDir.js'

export const codeReviewerAgent: AgentDefinition = {
  name: 'code-reviewer',
  description: 'A specialized agent for code review',
  systemPrompt: `You are an expert code reviewer. Focus on:
- Code quality and best practices
- Potential bugs and edge cases
- Performance implications
- Security vulnerabilities
- Maintainability

Provide specific, actionable feedback with code examples.`,
  
  // 允许的工具（子集）
  tools: ['BashTool', 'FileReadTool', 'GlobTool', 'GrepTool'],
  
  // 指定模型
  model: 'claude-sonnet-4-6',
  
  // 自定义设置
  maxTurns: 10,
  maxBudgetUsd: 0.50,
  
  // 提示词后缀
  promptAddendum: 'Remember to check for proper error handling.',
}

// 在 loadAgentsDir.ts 中注册
const BUILTIN_AGENTS: Record<string, AgentDefinition> = {
  // ... 现有 agents
  codeReviewerAgent,
}
```

### 3. 自动压缩系统

上下文压缩在 Token 不足时自动执行：

```typescript
// src/services/compact/autoCompact.ts

// 修改压缩策略
const AUTO_COMPACT_CONFIG = {
  // 触发压缩的阈值（85% 上下文窗口）
  threshold: 0.85,
  // 压缩后的目标（70% 上下文窗口）
  target: 0.70,
  // 最小保留消息数
  minMessages: 5,
  // 最大压缩次数
  maxCompactions: 3,
}

// 自定义压缩策略
export function customCompactStrategy(messages: Message[]): Message[] {
  // 保留系统消息和最近的对话
  const systemMessages = messages.filter(m => m.type === 'system')
  const recentMessages = messages.slice(-10)
  
  // 压缩中间的消息
  const middleMessages = messages.slice(
    systemMessages.length, 
    -recentMessages.length
  )
  const summary = createSummary(middleMessages)
  
  return [
    ...systemMessages,
    summary,
    ...recentMessages,
  ]
}
```

### 4. 添加新的功能开关

1. **在代码中使用**:
   ```typescript
   if (feature('MY_FEATURE')) {
     // 功能代码
   }
   ```

2. **构建配置**: 修改 `bunfig.toml` 或构建脚本

```toml
[define]
"BUN_FEATURES_MY_FEATURE" = "true"
```

3. **条件导入**:
```typescript
const myFeature = feature('MY_FEATURE')
  ? require('./features/myFeature.js').default
  : null
```

### 5. 自定义主题

```typescript
// src/utils/theme.ts

export const CUSTOM_THEMES = {
  myTheme: {
    primary: '#FF6B6B',
    secondary: '#4ECDC4',
    success: '#95E1D3',
    warning: '#F38181',
    error: '#AA96DA',
    background: '#2C3E50',
    foreground: '#ECF0F1',
    muted: '#95A5A6',
    // ... 更多颜色
  },
}

// 在设置中添加主题选项
export type ThemeName = keyof typeof THEMES | keyof typeof CUSTOM_THEMES

// 使用自定义主题
export function useTheme() {
  const themeName = getGlobalConfig().theme
  return CUSTOM_THEMES[themeName] ?? THEMES[themeName] ?? THEMES.dark
}
```

### 6. 扩展分析遥测

```typescript
// src/services/analytics/customEvents.ts

import { logEvent } from './index.js'

// 定义自定义事件类型
type CustomEvents = {
  'my_custom_event': {
    action: string
    duration_ms: number
    success: boolean
  }
  'my_feature_used': {
    feature: string
    context: string
  }
}

// 记录自定义事件
export function logMyCustomEvent(
  action: string,
  duration: number,
  success: boolean
) {
  logEvent('my_custom_event', {
    action,
    duration_ms: duration,
    success,
  })
}
```

### 7. 自定义权限系统

```typescript
// src/utils/permissions/customRules.ts

import type { PermissionResult, ToolPermissionContext } from '../../types/permissions.js'

// 自定义权限规则
export async function checkCustomPermission(
  toolName: string,
  input: unknown,
  context: ToolPermissionContext,
): Promise<PermissionResult> {
  // 检查是否在工作时间内
  const hour = new Date().getHours()
  const isWorkHours = hour >= 9 && hour < 18
  
  // 非工作时间限制某些操作
  if (!isWorkHours && isDestructiveTool(toolName)) {
    return {
      behavior: 'ask',
      updatedInput: input,
      message: 'This operation requires confirmation outside work hours.',
    }
  }
  
  // 检查路径规则
  if (hasPath(input)) {
    const isInAllowedPath = context.additionalWorkingDirectories
      .has(getBasePath(input.path))
    
    if (!isInAllowedPath) {
      return {
        behavior: 'deny',
        updatedInput: input,
        message: 'Path not in allowed working directories.',
      }
    }
  }
  
  return { behavior: 'allow', updatedInput: input }
}
```

---

## 完整示例

### 示例 1: 创建一个完整的工具 + 命令组合

这个示例创建一个代码统计工具和相关命令：

```typescript
// ═══════════════════════════════════════════════════════════
// 1. 工具实现
// ═══════════════════════════════════════════════════════════
// src/tools/CodeStatsTool/CodeStatsTool.ts

import { z } from 'zod/v4'
import { buildTool } from '../../Tool.js'
import { CODE_STATS_PROMPT } from './prompt.js'
import { glob } from 'glob'
import { readFile } from 'fs/promises'
import { extname } from 'path'

const CodeStatsInputSchema = z.object({
  pattern: z.string().describe('Glob pattern for files to analyze'),
  includeTests: z.boolean().optional(),
})

async function analyzeCodeStats(pattern: string, includeTests: boolean) {
  const files = await glob(pattern, { 
    ignore: includeTests ? [] : ['**/*.test.*', '**/*.spec.*'] 
  })
  
  const stats = {
    totalFiles: files.length,
    totalLines: 0,
    codeLines: 0,
    commentLines: 0,
    blankLines: 0,
    byExtension: {} as Record<string, { files: number; lines: number }>,
  }
  
  for (const file of files) {
    const content = await readFile(file, 'utf-8')
    const lines = content.split('\n')
    const ext = extname(file) || 'no-ext'
    
    stats.totalLines += lines.length
    
    if (!stats.byExtension[ext]) {
      stats.byExtension[ext] = { files: 0, lines: 0 }
    }
    stats.byExtension[ext].files++
    stats.byExtension[ext].lines += lines.length
    
    for (const line of lines) {
      const trimmed = line.trim()
      if (trimmed === '') {
        stats.blankLines++
      } else if (trimmed.startsWith('//') || trimmed.startsWith('/*')) {
        stats.commentLines++
      } else {
        stats.codeLines++
      }
    }
  }
  
  return stats
}

export const CodeStatsTool = buildTool({
  name: 'code_stats',
  description: async () => CODE_STATS_PROMPT,
  inputSchema: CodeStatsInputSchema,
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
  
  async call(input, context) {
    const stats = await analyzeCodeStats(input.pattern, input.includeTests ?? false)
    return { data: stats }
  },
  
  renderToolUseMessage(input) {
    return `Analyzing code statistics for: ${input.pattern}`
  },
  
  renderToolResultMessage(content) {
    return `Found ${content.totalFiles} files, ${content.totalLines} total lines`
  },
  
  mapToolResultToToolResultBlockParam(content, toolUseID) {
    return {
      type: 'tool_result',
      tool_use_id: toolUseID,
      content: JSON.stringify(content),
    }
  },
})

// ═══════════════════════════════════════════════════════════
// 2. 提示词
// ═══════════════════════════════════════════════════════════
// src/tools/CodeStatsTool/prompt.ts

export const CODE_STATS_PROMPT = `
## code_stats

Analyze code statistics for files matching a glob pattern.

### Parameters
- pattern: Glob pattern (e.g., "src/**/*.ts")
- includeTests: Whether to include test files

### Returns
Statistics including file count, line counts by type, and breakdown by file extension.
`

// ═══════════════════════════════════════════════════════════
// 3. 命令
// ═══════════════════════════════════════════════════════════
// src/commands/code-stats/index.ts

import type { Command } from '../../types/command.js'

const codeStatsCommand: Command = {
  type: 'prompt',
  name: 'code-stats',
  aliases: ['stats'],
  description: 'Show code statistics for the project',
  
  async getPromptForCommand(args, context) {
    const pattern = args || 'src/**/*.{ts,tsx,js,jsx}'
    
    return `Please analyze the code statistics for this project.

Use the code_stats tool with pattern: "${pattern}"

Then provide a summary including:
1. Total files and lines of code
2. Breakdown by file type
3. Code to comment ratio
4. Any observations about the codebase structure`
  },
}

export default codeStatsCommand

// ═══════════════════════════════════════════════════════════
// 4. 注册
// ═══════════════════════════════════════════════════════════

// 在 src/tools.ts 中添加:
// import { CodeStatsTool } from './tools/CodeStatsTool/CodeStatsTool.js'
// getAllBaseTools: [...existingTools, CodeStatsTool]

// 在 src/commands.ts 中添加:
// import codeStatsCommand from './commands/code-stats/index.js'
// COMMANDS: [...existingCommands, codeStatsCommand]
```

---

## 常见问题

### Q: 修改没有生效？

A: 确保：
1. 保存了文件
2. 重启了 `bun run dev`
3. 检查 TypeScript 错误: `bun tsc --noEmit`
4. 清除缓存: `rm -rf ~/.claude/cache`

### Q: 如何添加新的依赖？

A:
```bash
bun add package-name
# 或开发依赖
bun add -d package-name
```

### Q: 如何调试类型错误？

A:
```bash
# 运行类型检查
bun tsc --noEmit

# 查看详细错误
bun tsc --noEmit --pretty

# 监视模式
bun tsc --noEmit --watch
```

### Q: 如何贡献代码？

A:
1. 创建分支
2. 进行修改
3. 测试 (`bun test`)
4. 提交 PR

### Q: 构建失败？

A: 检查：
1. Bun 版本: `bun --version` (需要 >= 1.3.5)
2. Node 版本: `node --version` (需要 >= 24)
3. 依赖安装: `bun install`
4. 磁盘空间

### Q: 如何查看日志？

A:
```bash
# 实时查看
tail -f ~/.claude/logs/latest.log

# 查看特定会话
cat ~/.claude/logs/$(date +%Y-%m-%d).log
```

---

## 资源链接

- [Ink 文档](https://github.com/vadimdemedes/ink)
- [Anthropic API 文档](https://docs.anthropic.com/)
- [MCP 规范](https://modelcontextprotocol.io/)
- [Bun 文档](https://bun.sh/docs)
- [Zod 文档](https://zod.dev/)

---

## 结语

这份指南涵盖了 Claude Code 源码的主要方面。要深入理解，建议：

1. **从入口开始**: 阅读 `src/main.tsx` 了解启动流程
2. **跟踪一次完整对话**: 从用户输入到 API 响应再到工具执行
3. **选择感兴趣的模块深入**: 工具、命令、UI 或状态管理
4. **动手修改并测试**: 实践是最好的学习方式
5. **阅读现有代码**: 学习 Anthropic 工程师的编码模式

祝魔改愉快！🚀
