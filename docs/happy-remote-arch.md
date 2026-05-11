# Happy 远程操控 Claude Code 架构设计文档

## 概述

Happy 是一个 Claude Code 的远程控制平台，允许用户通过移动端或网页端远程操控运行在远程服务器上的 Claude Code 实例。其核心设计理念是将 Claude Code 作为后端服务运行，而用户界面则运行在前端客户端。

## 系统架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              用户层                                       │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐             │
│  │   iOS App    │    │  Android App │    │   Web App    │             │
│  │  (Expo App)  │    │  (Expo App)  │    │   (Web)      │             │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘             │
│         │                    │                    │                      │
└─────────┼────────────────────┼────────────────────┼──────────────────────┘
          │                    │                    │
          └────────────────────┼────────────────────┘
                              │ WebSocket (Socket.IO)
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            Happy Server                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      Socket.IO Server                             │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────────┐  │   │
│  │  │ Message Handler │  │ Session Handler│  │ Push Notifications │  │   │
│  │  └────────────────┘  └────────────────┘  └────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                         Database (PostgreSQL)                    │   │
│  │  Sessions, Messages, User Accounts, Encryption Keys               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              │ Socket.IO (认证后)
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          happy-cli (Remote Mode)                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                         Main Loop                                │   │
│  │              ┌───────────────────────────────┐                   │   │
│  │              │     Session Manager           │                   │   │
│  │              │   - Message Queue            │                   │   │
│  │              │   - State Management         │                   │   │
│  │              └───────────────────────────────┘                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Claude SDK Integration                         │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐    │   │
│  │  │ ClaudeRemote   │  │ Permission     │  │ MCP Server     │    │   │
│  │  │ Launcher       │  │ Handler        │  │ Manager        │    │   │
│  │  └────────────────┘  └────────────────┘  └────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │ Anthropic API
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Anthropic Claude API                             │
│                    Claude Code with MCP Support                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 核心组件详解

### 1. 主循环 (Main Loop)

**文件**: `packages/happy-cli/src/claude/loop.ts`

主循环是整个 CLI 的控制中心，负责在 local 模式和 remote 模式之间切换。

```typescript
export async function loop(opts: LoopOptions): Promise<number> {
    let mode: 'local' | 'remote' = opts.startingMode ?? 'local';
    while (true) {
        switch (mode) {
            case 'local':
                const result = await claudeLocalLauncher(session);
                if (result.type === 'switch') mode = 'remote';
                break;
            case 'remote':
                const reason = await claudeRemoteLauncher(session);
                if (reason === 'switch') mode = 'local';
                break;
        }
    }
}
```

**设计意图**: 这种设计允许用户在任何时候通过双击空格键在本地模式和远程模式之间切换，实现"dogfooding"——开发者自己也在使用自己的产品。

### 2. 远程启动器 (Claude Remote Launcher)

**文件**: `packages/happy-cli/src/claude/claudeRemoteLauncher.ts`

这是远程模式的核心，负责管理 Claude SDK 进程的整个生命周期。

#### 2.1 组件初始化

```typescript
// 1. 权限处理器 - 管理工具调用权限
const permissionHandler = new PermissionHandler(session);

// 2. 消息队列 - 延迟发送工具调用消息
const messageQueue = new OutgoingMessageQueue(
    (logMessage) => session.client.sendClaudeSessionMessage(logMessage)
);

// 3. SDK 到日志转换器 - 转换消息格式
const sdkToLogConverter = new SDKToLogConverter({}, permissionHandler.getResponses());
```

#### 2.2 消息处理管道

```
Claude SDK 消息
    │
    ▼
formatClaudeMessageForInk()     // 格式化显示
    │
    ▼
sdkToLogConverter.convert()     // 转换为统一格式
    │
    ▼
messageQueue.enqueue()          // 进入队列（tool calls 有 250ms 延迟）
    │
    ▼
session.client.sendClaudeSessionMessage()  // 通过 Socket 发送
    │
    ▼
Happy Server → 数据库 → 广播给所有客户端
```

### 3. Claude Remote (SDK 封装)

**文件**: `packages/happy-cli/src/claude/claudeRemote.ts`

这是对 Anthropic Claude SDK 的封装，处理与 SDK 的直接交互。

#### 3.1 SDK 调用

```typescript
const response = query({
    prompt: messages,        // 用户消息流
    options: {
        cwd: opts.path,      // 工作目录
        mcpServers: opts.mcpServers,  // MCP 服务器配置
        permissionMode: mapToClaudeMode(initial.mode.permissionMode),
        canCallTool: (toolName, input, options) =>
            opts.canCallTool(toolName, input, mode, options),
    },
});

// 迭代响应
for await (const message of response) {
    // 处理各类消息
}
```

#### 3.2 消息类型

| 类型 | 说明 |
|------|------|
| `system` | 系统初始化消息，包含工具列表、会话 ID |
| `assistant` | Claude 助手的响应 |
| `user` | 用户消息（包括工具结果） |
| `result` | 最终结果标记 |
| `error` | 错误消息 |

### 4. 会话管理 (Session)

**文件**: `packages/happy-cli/src/claude/session.ts`

Session 类是客户端状态的容器，管理会话相关的一切数据。

```typescript
export class Session {
    readonly path: string;           // 项目路径
    readonly api: ApiClient;          // API 客户端
    readonly client: ApiSessionClient; // 会话客户端
    readonly queue: MessageQueue2;    // 消息队列

    sessionId: string | null;         // Claude 会话 ID
    mode: 'local' | 'remote';        // 当前模式
    thinking: boolean;                // 是否正在思考

    // Keep Alive - 每 2 秒发送心跳
    private keepAliveInterval = setInterval(() => {
        this.client.keepAlive(this.thinking, this.mode);
    }, 2000);
}
```

### 5. 权限系统 (Permission Handler)

**文件**: `packages/happy-cli/src/claude/utils/permissionHandler.ts`

#### 5.1 权限模式

| 模式 | 描述 | 允许的操作 |
|------|------|-----------|
| `default` | 默认模式 | 所有操作需审批 |
| `acceptEdits` | 接受编辑 | 文件编辑自动批准 |
| `bypassPermissions` | 绕过权限 | 所有操作自动批准 |
| `plan` | 计划模式 | 只读操作自动批准 |
| `haiku` | Haiku 模式 | 限制性操作 |
| `sonnet` | Sonnet 模式 | 中等权限 |
| `opus` | Opus 模式 | 完整权限 |

#### 5.2 权限流程

```
Claude SDK 调用工具
    │
    ▼
PermissionHandler.handleToolCall()
    │
    ├─── 检查显式允许列表
    │
    ├─── 检查 Bash 命令白名单（前缀/精确匹配）
    │
    ├─── 检查权限模式
    │
    └─── 需要审批 → 发送 permission_request 通知
                      │
                      ▼
                 客户端 UI 显示审批对话框
                      │
                      ▼
                 用户批准/拒绝
                      │
                      ▼
                 RPC 'permission' 响应
                      │
                      ▼
                 resolve Promise<PermissionResult>
```

### 6. Happy Server 会话处理

**文件**: `packages/happy-server/sources/app/api/socket/sessionUpdateHandler.ts`

#### 6.1 核心 Socket 事件

| 事件 | 方向 | 说明 |
|------|------|------|
| `message` | Client → Server | 发送消息 |
| `session-alive` | Client → Server | 心跳 |
| `session-end` | Client → Server | 会话结束 |
| `update-metadata` | Client → Server | 更新元数据 |
| `update-state` | Client → Server | 更新代理状态 |

#### 6.2 消息处理流程

```typescript
socket.on('message', async (data) => {
    // 1. 验证会话
    const session = await db.session.findUnique({
        where: { id: sid, accountId: userId }
    });

    // 2. 加密消息内容
    const msgContent = {
        t: 'encrypted',
        c: message
    };

    // 3. 分配序列号
    const updSeq = await allocateUserSeq(userId);
    const msgSeq = await allocateSessionSeq(sid);

    // 4. 存储消息
    const msg = await db.sessionMessage.create({
        data: { sessionId: sid, seq: msgSeq, content: msgContent }
    });

    // 5. 广播给所有相关客户端
    eventRouter.emitUpdate({
        payload: buildNewMessageUpdate(msg, sid, updSeq),
        recipientFilter: { type: 'all-interested-in-session', sessionId: sid }
    });
});
```

## 消息协议设计

### 原始消息格式 (RawJSONLines)

Claude SDK 输出的原始格式，经过 `sessionProtocolMapper` 转换：

```typescript
// Assistant 消息
{
    type: 'assistant',
    message: {
        role: 'assistant',
        content: [
            { type: 'text', text: '分析中...' },
            { type: 'tool_use', id: 'tool_123', name: 'Read', input: {...} }
        ]
    }
}

// Tool Result 消息
{
    type: 'user',
    message: {
        role: 'user',
        content: [
            {
                type: 'tool_result',
                tool_use_id: 'tool_123',
                content: '文件内容...',
                permissions: { date: 1234567890, result: 'approved' }
            }
        ]
    }
}
```

### 协议信封 (Session Envelope)

转换后的统一格式：

```typescript
interface SessionEnvelope {
    role: 'user' | 'agent' | 'session';
    ev: {
        t: 'text' | 'tool_use' | 'tool_result' | 'thinking' | 'usage';
        id: string;
        parentId?: string;
        data: any;
    };
}
```

## 核心流程详解

### 流程 1: 用户发送消息

```
1. 用户在 App 输入消息
       │
       ▼
2. App 通过 Socket 发送: socket.emit('message', { sid, message, localId })
       │
       ▼
3. Server sessionUpdateHandler 接收
       │
       ▼
4. 加密并存储到数据库
       │
       ▼
5. 广播给所有客户端（包括 CLI）
       │
       ▼
6. CLI MessageQueue2 接收并推送
       │
       ▼
7. claudeRemote.nextMessage() 返回消息
       │
       ▼
8. SDK 处理消息，生成响应
```

### 流程 2: Claude 调用工具

```
1. Claude 决定调用工具 (e.g., Read file)
       │
       ▼
2. SDK canCallTool 回调触发
       │
       ▼
3. PermissionHandler 检查权限
       │
       ├─── 允许 → 直接执行
       │
       └─── 需要审批
              │
              ▼
           发送 'permission' notification
              │
              ▼
           App 显示审批对话框
              │
              ▼
           用户选择审批/拒绝
              │
              ▼
           RPC 响应返回
              │
              ▼
           工具执行，结果返回 SDK
```

### 流程 3: 消息显示延迟机制

```
1. Claude 发送 assistant 消息（包含 tool_use）
       │
       ▼
2. 消息进入 OutgoingMessageQueue
       │
       ▼
3. 设置 250ms 延迟
       │
       ▼
4. 250ms 后发送消息
       │
       ▼
5. 用户看到消息，有机会点击"中断"
       │
       ├─── 未中断 → 正常执行工具
       │
       └─── 中断 → 发送 abort 信号
                    │
                    ▼
                 isAborted(toolCallId) 返回 true
                    │
                    ▼
                 跳过工具执行
```

### 流程 4: 远程模式启动

```
1. 用户在 App 点击"连接远程"
       │
       ▼
2. Server 创建/恢复会话
       │
       ▼
3. CLI 收到会话事件
       │
       ▼
4. loop() 进入 remote 模式
       │
       ▼
5. claudeRemoteLauncher 初始化
       │
       ▼
6. 连接 Anthropic API
       │
       ▼
7. 等待消息循环开始
```

## 安全性设计

### 1. 消息加密

所有消息内容在传输前都会被加密：

```typescript
const msgContent: SessionMessageContent = {
    t: 'encrypted',
    c: encrypt(message)  // 加密后的内容
};
```

### 2. 权限隔离

- 每个工具调用都需要明确的权限
- Bash 命令支持前缀匹配和精确匹配
- 权限模式控制操作范围

### 3. 会话隔离

- 用户只能访问自己的会话
- 会话 ID 使用 UUID 防止猜测
- 所有操作都验证用户身份

## 性能优化

### 1. Keep Alive 心跳

```typescript
// 每 2 秒发送一次心跳
this.keepAliveInterval = setInterval(() => {
    this.client.keepAlive(this.thinking, this.mode);
}, 2000);
```

### 2. 消息队列

- Tool calls 有 250ms 延迟，允许用户中断
- 其他消息立即发送
- 队列支持批量处理

### 3. 版本控制

使用乐观锁防止并发冲突：

```typescript
if (session.metadataVersion !== expectedVersion) {
    return { result: 'version-mismatch', version: session.metadataVersion };
}
```

## 扩展点

### MCP 服务器支持

```typescript
// 通过配置传递 MCP 服务器
const sdkOptions: QueryOptions = {
    mcpServers: opts.mcpServers,  // e.g., { mcp-server: { command: 'npx', args: [...] } }
};
```

### 自定义工具

可以通过 MCP 协议添加自定义工具，Happy 会自动发现并集成。

### 多模型支持

架构支持多种 AI 模型（Claude、Codex、Gemini），通过统一的 `loop.ts` 接口切换。

## 总结

Happy 的远程控制架构是一个精心设计的分布式系统：

1. **分层设计**: 清晰的分层，从 UI 到 SDK 到 API
2. **实时通信**: 基于 WebSocket 的双向通信
3. **权限安全**: 多层权限控制和审批流程
4. **消息可靠**: 序列号、版本控制确保消息顺序和一致性
5. **用户体验**: 250ms 延迟机制允许用户随时中断

这个架构使得用户可以在任何设备上远程操控强大的 Claude Code 实例，同时保持了安全性和可用性。
