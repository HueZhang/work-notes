## 模块一：LangChain 基础组件与调用方式

### 🎯 主问题：LangChain 中的核心消息类型（System、Human、AI、Tool）如何区分？

* **答案摘要**：
* **SystemMessage**：系统设定/人设，规定最高行为准则。
* **HumanMessage**：用户输入的提问或指令。
* **AIMessage**：大模型的回复，可能包含文本或工具调用请求（`tool_calls`）。
* **ToolMessage**：外部工具执行完后的真实返回结果，需带上 `tool_call_id` 反哺给大模型。

#### 📌 次要/补充问题 1：LangChain 中的 `invoke`、`batch`、`stream` 有什么区别？

* **答案摘要**：
* `invoke`：单次阻塞调用，返回单个完整结果。
* `batch`：批量并行调用，接收列表并返回列表，适合高吞吐处理。
* `stream`：流式分片返回，返回迭代器，适合实现逐字打印的打字机效果。

#### 📌 次要/补充问题 2：如何使用 Prompt Template 组织带变量的提示词？

* **答案摘要**：
* 纯文本用 `PromptTemplate`，聊天模型用 `ChatPromptTemplate`（支持定义多角色消息）。
* 在实际 LCEL 中，可以直接将模板与大模型用管道符 `|` 串联，在 `.invoke()` 时直接传入字典变量自动填充。

---

## 模块二：模型响应与 API 调用

### 🎯 主问题：使用 DeepSeek 或 GPT 返回的 response 结构是否一样？

* **答案摘要**：
* **在 LangChain 封装下**：结构完全一样。两者都返回标准 `AIMessage`，可通过 `.content` 和 `.tool_calls` 无缝通用。
* **在底层 Raw API 中**：略有不同，例如 DeepSeek 的推理模型（如 `deepseek-reasoner`）会有专属的深度思考字段（`reasoning_content`）。

#### 📌 次要/补充问题：如果调用 DeepSeek 的 API，如何直接获取 `content` 内容？

* **答案摘要**：
* **LangChain 方式**：调用 `response = llm.invoke(...)` 后，直接访问 `response.content`。
* **原生 OpenAI SDK 方式**：通过 `response.choices[0].message.content` 获取。

---

## 模块三：结构化输出、Pydantic 约束与兼容性

### 🎯 主问题：Prompt 约束与程序校验有什么区别？如何用 Pydantic 定义请求与响应模型？

* **答案摘要**：
* **Prompt 约束（软约束）**：靠大模型的理解和概率执行，灵活性高但可能失效。
* **程序校验（硬约束）**：靠代码（如 Pydantic）进行确定性拦截，100% 安全。
* **定义方式**：继承 `BaseModel` 并使用 `Field(description=...)` 定义字段，配合 `.with_structured_output()` 可直接让模型输出强类型实例。

#### 📌 次要/补充问题 1：字段的必填、默认值、枚举、长度和数值约束怎么写？模型大概率会遵守吗？

* **答案摘要**：
* **必填/默认值**：直接声明或用 `default=`。
* **枚举**：推荐用 `Literal["选项1", "选项2"]`。
* **数值/长度**：用 `ge`、`le`、`min_length` 等。
* **遵守率**：枚举和基本类型遵守率极高（>95%）；字符长度约束大模型表现较差，需靠 Pydantic 代码兜底。

#### 📌 次要/补充问题 2：为什么用 Pydantic 配合 DeepSeek 调用 `with_structured_output` 会报错？修改 `tool_choice="auto"` 为什么能解决？

* **答案摘要**：
* **报错原因**：LangChain 默认的强制单工具调用协议与 DeepSeek 底层某些参数解析存在细微的不兼容。
* **解决原理**：将 `tool_choice` 设为 `"auto"` 后，解除了强行锁死特定内部函数名的限制，恢复了 API 的兼容性。

#### 📌 次要/补充问题 3：当模型或服务商不支持特定结构化输出模式时，有什么兼容性降级方案？

* **答案摘要**：
* 降级为 `json_mode`。
* 退化为“Prompt 手写 Schema 描述 + `PydanticOutputParser`”的经典解析路线。
* 开启 `RetryWithErrorOutputParser` 自纠错重试机制。

---

## 模块四：LCEL 流水线设计

### 🎯 主问题：LCEL 管道中 `prompt | llm | parser` 的先后顺序可以改变吗？

* **答案摘要**：
* **绝对不能改变**。这由数据流转的类型决定：字典输入经 `prompt` 变为消息列表，大模型消费消息列表吐出 `AIMessage`，最后由 `parser` 将 `AIMessage` 解析为结构化对象。顺序错乱会导致类型不匹配报错。

---
