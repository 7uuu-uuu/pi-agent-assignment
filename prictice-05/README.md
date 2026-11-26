# 实践课5：Pi Agent 源码泛读与功能定位 LiYunyi

**姓名**：李韵怡

**学号**：202430224028

**随机标识**：`7uuu-uuu`

## 一、 实验环境与准备

 **启动方式**：本地构建产物 `node ./packages/coding-agent/dist/cli.js`
 
 **工作目录**：`D:\PiAgent-fork\pi-hunnu-2026`
 
 **准备文件**：`handoff.txt`
  ```text
  待办：补充 README 的启动说明
  启动命令：npm run demo -- --profile 7uuu-uuu
  ```

---

## 二、 实验过程与观察记录

### A. 单轮交互与工具调用

**1. 用户输入**
在 Pi 交互界面中输入：
> 请用 read 工具读取当前目录的 handoff.txt，原样告诉我待办和启动命令。

**2. 观察记录**
- **工具调用**：Agent 成功理解意图，触发 `read` 工具，实际参数为 `D:/PiAgent-fork/pi-hunnu-2026/handoff.txt`。
- **模型调用**：发生了两次模型请求。第一次模型输出了工具调用请求；工具执行完毕后，终端显示 `[P4] 工具结果已加入上下文`，随后 `[P4] Agent 即将请求模型`，模型进行第二次输出，向用户展示了完整原样内容。
- **回复显示**：Agent 成功返回了待办和启动命令。终端工具事件日志观察：输出 `[P4] 工具调用 read`、`[P4] 工具结果已加入上下文`，无重复多次调用 read 工具，仅执行 1 次 read 文件操作。
<img width="2318" height="1314" alt="22b8f0ce-8de0-4331-b0ac-ae6a2cd1e9ec" src="https://github.com/user-attachments/assets/089ec500-a8b0-403d-9fe0-678bf2084d8b" />

### B. 恢复上次会话 vs 新建会话

**1. 恢复原会话 (`-c`)**
- **操作**：退出 Pi，在项目目录使用 `node ./packages/coding-agent/dist/cli.js -c` 重新启动，终端成功恢复了 A 阶段的聊天记录。
- **输入**：> 接着上次的待办，给 README 起草一条启动说明，写出交接单里的完整启动命令。只用这次会话已有的信息。不要调用工具。如果缺少命令，请直接说明。
- **观察记录**：
  - **工具调用**：无（严格遵循了“不要调用工具”的指令）。
  - **回答内容**：成功给出了 `npm run demo -- --profile 7uuu-uuu`，并解释该信息来自恢复的会话历史（"From the handoff, the startup command is..."）。
<img width="2334" height="1696" alt="12380e20-3d92-49ca-a0c9-106cf795fcc1" src="https://github.com/user-attachments/assets/4c0c50d3-d4c4-4c55-b7a4-afbdf883616d" />

**2. 新建会话 (`/new`)**
- **操作**：在交互界面输入 `/new`，随后输入与 B1 完全相同的提问。
- **观察记录**：
  - **工具调用**：无。
  - **回答内容**：Agent 明确表示无法完成起草，原因是“这次会话里没有交接单或上次待办的内容”，并说明缺少“交接单原文里记录的完整启动命令”。这证明新会话的上下文为空，没有 A 阶段的数据。
<img width="2694" height="1686" alt="cb158a04-9cf8-4056-8134-f0eca108fd47" src="https://github.com/user-attachments/assets/83f26366-0516-49dc-9896-b40bcbf05d4a" />

---

## 三、 代码级分析图表

### 图 1：单轮工具调用的代码级时序图

以下为 Mermaid 格式的时序图，你可以使用支持 Mermaid 的 Markdown 编辑器（如 Typora、GitHub）直接预览，或将渲染后的图片放入报告中。

```mermaid
sequenceDiagram
    autonumber
    actor User as 用户
    participant IM as 交互与会话层<br/>(interactive-mode.ts)
    participant Agent as Agent执行层<br/>(agent.ts / agent-loop.ts)
    participant Model as 模型请求层<br/>(Provider Adapter)
    participant ReadTool as read工具<br/>(read.ts)

    User->>IM: 输入提问并回车
    Note over IM: setupEditorSubmitHandler 中的 onSubmit(text)
    IM->>IM: resolve getUserInput()
    IM->>Agent: await session.prompt(userInput)
    
    Note over Agent: Agent.prompt()【agent.ts】调用 runAgentLoop()【agent-loop.ts】跨文件调用
    Agent->>Agent: runAgentLoop(...)
    
    Agent->>Model: streamFunction(model, llmContext)
    Note over Model: Provider Adapter发起HTTP请求 ↔ 远端大模型API (SSE流式响应)
    Model-->>Agent: 返回 toolCall: read(filePath)
    
    Agent->>ReadTool: 执行 read 工具
    Note over ReadTool: 底层文件系统调用 createReadStream，流式读取文件
    ReadTool-->>Agent: 返回文件内容 (工具结果加入上下文)
    
    Agent->>Model: 再次请求模型（携带工具调用结果）
    Note over Model: Provider Adapter发起HTTP请求 ↔ 远端大模型API (SSE流式响应)
    Model-->>Agent: 返回最终文本回复
    
    Agent->>IM: 触发 AgentEvent 事件流
    Note over IM: handleEvent 监听Agent事件
    IM->>IM: message_update (流式分段渲染输出)
    IM->>IM: message_end (会话消息最终状态更新)
    IM-->>User: 终端展示完整回复
```

**源码定位说明**：
- **输入入口**：`interactive-mode.ts` 的 `onSubmit` 钩子接收文本，并调用 `session.prompt()`。
- **启动循环**：进入 `agent.ts` 的 `Agent.prompt()`，随后跨文件在 `agent-loop.ts` 的 `runAgentLoop` 中进行多轮循环，执行主逻辑。
- **工具与模型边界**：模型请求层为适配器层（Provider Adapter），隔离了本地 Agent 逻辑和远端大模型 API，所有请求和响应都经过它转换。远端模型通过 SSE 流式返回。
- **read工具**：源码位于 `packages/coding-agent/src/core/tools/read.ts`，底层使用了 `createReadStream` 进行流式读取，而非一次性缓冲区读取。
- **UI显示回复**：`interactive-mode.ts` 通过监听 `message_update`实现流式逐字输出，通过 `message_end` 完成最终消息落盘与状态更新。

---

### 图 2：恢复旧会话和新建会话的对照图

```mermaid
graph TD
    Start((A阶段结束<br />已完成read读取，得到启动命令)) --> Choice{用户操作}
    
    %% 左分支：恢复会话
    Choice -->|退出Pi，使用 -c / --continue启动| Resume[main.ts解析-c参数<br />加载最近会话JSONL文件]
    Resume --> LoadFile[读取本地会话文件<br />反序列化恢复全部历史聊天消息]
    LoadFile --> UserInput1[用户输入: 请起草 README 启动说明]
    UserInput1 --> Context1[发给模型的上下文：<br/>【磁盘恢复全部历史消息】 + 本次用户提问]
    Context1 --> Request1[发起模型请求]
    Request1 --> Result1[模型拥有handoff.txt信息<br />输出完整npm启动命令]
    
    %% 右分支：新建会话
    Choice -->|交互输入 /new| NewSession[创建全新会话实例<br />内存聊天历史栈清空]
    NewSession --> UserInput2[用户输入：完全相同提问]
    UserInput2 --> Context2[发给模型的上下文：<br/>【仅本次用户提问，无历史记录】]
    Context2 --> Request2[发起模型请求]
    Request2 --> Result2[模型缺少历史上下文<br />回复：缺少交接单启动命令，无法完成]
    
    classDef leftBranch fill:#d4edda,stroke:#28a745;
    classDef rightBranch fill:#f8d7da,stroke:#dc3545;
    class Resume,LoadFile,UserInput1,Context1,Request1,Result1 leftBranch;
    class NewSession,UserInput2,Context2,Request2,Result2 rightBranch;
```


**图2源码定位说明**：
- **恢复旧会话 (`-c`)**：由 packages/coding-agent/src/main.ts 解析 --continue 参数，定位并读取 ~/.pi/agent/sessions/*.jsonl 文件。每条对话 entry 均持久化到该文件，恢复时将历史消息与本次提问拼接提交给模型。
- **新建会话 (`/new`)**：清空内存中的 SessionContext，历史消息栈置为空。此时请求模型的上下文仅包含当前的一轮对话，因此 Agent 会回复“缺少交接单内容”。

---

## 四、 实验总结


1. **工具调用循环**：单次用户提问可能引发多次模型请求。`read` 工具的底层实现通过 `createReadStream` 保证了文件读取效率。
2. **事件驱动的 UI**：模型生成的流式响应通过 `AgentEvent` 触发 `handleEvent`，通过 `message_update` 和 `message_end` 实现了终端界面中光标流动与最终定稿的效果。
3. **会话状态管理**：`-c` 启动与 `/new` 启动在底层加载的上下文完全不同。Pi 依赖本地 JSONL 文件实现了跨进程的会话持久化。这也提示我们在日常使用中，如果想延续上下文，应当妥善利用 `-c` 恢复功能，避免产生“Agent 失忆”的问题。
通过本次 Pi-Agent 源码泛读实验，结合实际运行观测，理清了 Pi Agent 工具调用与会话持久化的底层逻辑：

1. **工具调用循环机制**：单轮用户输入会产生多轮模型交互：模型先输出工具调用指令 → Agent 执行本地工具（read 读文件） → 将工具结果追加进上下文，再次请求模型生成最终回答。read 工具底层采用 createReadStream 流式读取，适合大文件场景。

2. **终端流式输出事件驱动**：Agent 内部产生 AgentEvent 事件向上抛出；交互层 interactive-mode.ts 监听事件，通过 message_update 做流式打字输出，message_end 标记一轮对话结束，实现终端动态渲染效果。

3. **会话持久化原理**：Pi 把完整对话历史持久化在本地 JSONL 会话文件。-c (--continue) 会读取磁盘文件恢复历史消息，新旧消息拼接一起提交模型；/new 新建会话直接清空内存会话上下文，不加载旧记录。

实验现象提供了直接证据：恢复会话场景无需重复调用 read 工具就可以复用之前文件读取结果；新会话完全失忆。这说明 Agent 记忆本质就是发给大模型的上下文消息列表，进程退出后记忆保存在本地磁盘文件。

