# 实践课3、4、5 Issue 提交内容

## Issue 1：实践课3：Pi Agent源码安装与编译 liuwenjia

**标题：**
```
实践课3：Pi Agent源码安装与编译 liuwenjia
```

**正文：**
```
## 基本信息
- 姓名：刘文佳
- 学号：202430154070
- GitHub 登录名：FaGaoSao

## 作业链接
- 个人报告：https://github.com/FaGaoSao/pi-agent-practice/blob/main/practice-3/README.md
- 小组代码仓库：https://gitee.com/pinhaodui/pi-hunnu-2026

## 完成情况
- 环境检查：Node.js v24.14.0、npm 11.9.0、Git 2.53.0.windows.1，满足根项目 engines 要求 >=22.19.0
- 源码确认：org 远端指向 https://gitee.com/pinhaodui/pi-hunnu-2026.git，分支 course-main，提交编号 d981de1229ef899957bbe968bc8dcda02a21f477
- 依赖安装：npm install --ignore-scripts，added 313 packages
- 本地编译：首次 npm run build 因联网更新模型目录在 ai 包失败（TS2307 kimi-coding.models.ts 缺失、TS2322 opencode-go 类型不匹配）；回退受版本控制的生成文件并从 npm 同版本包补齐被 gitignore 的数据文件后，改用 npm run build:offline 构建成功，退出码 0，产物 Built packages\coding-agent\dist\bundle (51 files, 7.5 MiB)
- 产物验证：Test-Path .\packages\coding-agent\dist\bundle\cli.js 为 True，运行得到版本号 0.85.1
- 本地源码版会话：.\pi-test.bat 启动并完成一次问答

报告含完整的困难描述与解决过程、截图与自己的理解。
```

---

## Issue 2：实践课4：Pi Agent模型调用与运行流程 liuwenjia

**标题：**
```
实践课4：Pi Agent模型调用与运行流程 liuwenjia
```

**正文：**
```
## 基本信息
- 姓名：刘文佳
- 学号：202430154070
- GitHub 登录名：FaGaoSao

## 作业链接
- 个人报告：https://github.com/FaGaoSao/pi-agent-practice/blob/main/practice-4/README.md
- 小组代码仓库：https://gitee.com/pinhaodui/pi-hunnu-2026

## 完成情况
### 源码修改
在 packages/agent/src/agent-loop.ts 加入两处插桩（本版本行号与指导书不同，通过搜索语句定位）：
- 第 307 行 console.error("[P4] Agent 即将请求模型"); 插在 const response = await streamFunction(...) 上方
- 第 239 行 console.error("[P4] 工具结果已加入上下文"); 插在 currentContext.messages.push(result); 之后

### 运行证据
- 重新编译：npm run build:offline 成功，$LASTEXITCODE = 0
- 启动：node .\packages\coding-agent\dist\cli.js

### 两次会话对比（practice4.txt 代号为 river-731）
| 会话 | 指令 | 是否调用工具 | [P4] 日志顺序 | 模型调用轮数 | 最终回复 |
|---|---|---|---|---|---|
| A 直接问答 | 请直接用一句话解释什么是 Git 分支。 | 否 | 即将请求模型（1条） | 1 轮 | Git 分支是 Git 中从主开发线分离出来的一条独立提交线… |
| B 读取文件 | 请用 read 工具读取当前目录的 practice4.txt，并告诉我文件中的实验代号。 | 是，调用 read | 即将请求模型 → 工具结果已加入上下文 → 即将请求模型（3条） | 2 轮 | 文件中的实验代号是：river-731 |

结合 B 的日志：实际读文件的是 read 工具（packages/coding-agent/src/core/tools/read.ts，用 fs/promises 的 readFile 缓冲读取，非流式）；下一轮模型请求多带了 createToolResultMessage() 构造的 role: "toolResult" 消息（含 toolCallId、toolName、content、isError、timestamp）。

报告含代码差异、日志记录、四者关系简图（coding-agent、agent、ai、远程模型服务）。
```

---

## Issue 3：实践课5：Pi Agent工具调用与会话恢复 liuwenjia

**标题：**
```
实践课5：Pi Agent工具调用与会话恢复 liuwenjia
```

**正文：**
```
## 基本信息
- 姓名：刘文佳
- 学号：202430154070
- GitHub 登录名：FaGaoSao

## 作业链接
- 个人报告：https://github.com/FaGaoSao/pi-agent-practice/blob/main/practice-5/README.md
- 小组代码仓库：https://gitee.com/pinhaodui/pi-hunnu-2026

## 完成情况
工作目录创建 handoff.txt（本组随机标识 river-731，含待办与启动命令 npm run demo -- --profile river-731）。

### 图1：单轮工具调用的代码级时序图
五条生命线（用户 / 交互与会话层 / Agent 执行层 / 模型请求层 / read 工具），箭头标注真实函数与语句：
- 输入入口 interactive-mode.ts:2965 onSubmit → 主循环 interactive-mode.ts:1135 session.prompt()
- Agent 执行层 agent.ts:350 Agent.prompt() → agent-loop.ts:156 runLoop() → agent-loop.ts:212 streamAssistantResponse()
- 模型请求层 agent-loop.ts:308 streamFunction → sdk.ts:314 → model-runtime.ts:636 → compat.ts:275 → openai-completions.ts:311，真正发出 HTTP 请求在 openai-completions.ts:367
- read 工具 read.ts:79 execute() → read.ts:104 ops.access → read.ts:129 ops.readFile（核对结论：缓冲区读取 fs/promises readFile，非流式，无 createReadStream）→ read.ts:151 truncateHead
- 回复显示 assistant-message.ts:91 updateContent() → assistant-message.ts:114 new Markdown()
已用 Note 标出同一层内的跨文件调用。

### 图2：恢复旧会话与新建会话对照图
- 恢复旧会话（-c）：回复包含完整启动命令 npm run demo -- --profile river-731，[P4] 日志仅 1 条，未调用工具 → 证明使用了恢复出的会话历史
- 新建会话（/new，等价 --no-session）：相同提问，回复明确说明"本会话中并不存在上次的待办或交接单的内容"，答不出命令

两次对照：提问相同、模型相同、工具配置相同，唯一差异是是否加载 A 阶段会话历史。

报告含两张图的可编辑图源（.mmd）与渲染图，以及适合修改的源码位置说明。
```

---

## 提交地址

https://github.com/hunnuOpenSourceSoftware2026/assignments/issues/new

三个 Issue 用同一个报告仓库的不同路径，按顺序提交即可。
