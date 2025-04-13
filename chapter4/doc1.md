# Day 2：模型IO与数据连接
## 一、 Model I/O
LangChain的Model I/O模块提供了标准化、可扩展的接口，用于实现与大语言模型(LLM)的外部集成。该模块通过抽象化的设计简化了与各类大模型的交互流程，主要包含三个核心组件：
1. 模型输入(Prompts) - 处理与模型的交互输入
2. 模型输出(Outputs) - 处理模型的响应输出
3. 模型本身(Models) - 提供统一的模型调用接口

![图片描述](./images/modelIO.png)

对于这个模块，LangChain抽象出了"Chain"的概念，通过链式处理简化和增强交互流程。一个完整的Chain包含三个关键处理阶段：


### 1. Format（格式化阶段）
- 功能：使用Prompt Templates（提示模板）管理模型输入
- 实现方式：
    - 提供模板化输入管理
    - 支持动态变量插入
    - 可定制不同场景的提示模板
- 优势：使提示工程更加系统化和可复用

#### 1.1 什么是 Prompt Template？
Prompt Template 是一种预定义的提示生成方法，其核心是一个模板字符串。该模板字符串中包含了固定的文本和占位符变量，通过传入用户提供的参数，这些变量会被动态替换，最终生成一个完整的提示。

#### 1.2 Prompt Template 的主要组成部分：

- 指令部分：对语言模型的基本指令，例如“扮演命名顾问”等。
- 变量占位符：用花括号 {} 标记，用于动态插入用户输入的参数，如 {product}、{adjective} 等。
- Few-Shot Examples（可选）：一组示例对话或示例问题与答案，作为上下文帮助模型理解任务要求，从而生成更准确的响应。

#### 1.3  为什么需要使用 Prompt Template？
- 简化提示设计：对于同类型的任务，不需要每次都从头编写完整提示，通过模板即可复用已有结构。
- 灵活性和动态性：支持传入任意数量的变量，便于根据不同场景和输入动态生成提示内容。
- 提升模型效果：通过插入 Few-Shot Examples，模型可以借助示例学习如何回答问题，从而提高生成响应的质量。

#### 1.4 如何创建 Prompt Template？
**1. 使用 PromptTemplate 类创建模板**
你可以直接使用 PromptTemplate 类定义一个硬编码提示，并指定需要的输入变量。例如：

```python
from langchain import PromptTemplate

# 定义一个模板，包含一个变量 {product}
template = """
I want you to act as a naming consultant for new companies.
What is a good name for a company that makes {product}?
"""

prompt = PromptTemplate(
    input_variables=["product"],
    template=template,
)
result = prompt.format(product="colorful socks")
print(result)
```

输出结果为：
```text
I want you to act as a naming consultant for new companies.
What is a good name for a company that makes colorful socks?
```

**2. 自动推断输入变量**
如果不想手动指定 input_variables，可以使用 from_template 类方法。LangChain 会根据模板自动推断需要的变量：

```python
template = "Tell me a {adjective} joke about {content}."
prompt_template = PromptTemplate.from_template(template)
print(prompt_template.input_variables)
# 输出: ['adjective', 'content']
print(prompt_template.format(adjective="funny", content="chickens"))
# 输出: "Tell me a funny joke about chickens."
```

**3. 从文件中读取 from_file 方法**

该方法用于从文件加载提示模板。你只需传入模板文件的路径以及模板中包含的输入变量列表。
- **input_variables 参数：**
尽管 from_file 方法现在会调用内部的 from_template 方法，但你仍然需要传入一个包含模板中占位符名称的列表。
- **格式化提示：**
调用 format 方法时，传入的关键字参数将替换模板中对应的占位符，从而生成最终的提示文本。

```python
from langchain import PromptTemplate

# 使用 from_file 方法加载模板文件，注意这里指定了输入变量列表
prompt = PromptTemplate.from_file("template.txt", input_variables=["name", "place"])

# 使用 format 方法传入变量，生成最终提示
formatted_prompt = prompt.format(name="Alice", place="Wonderland")
```

**4. 模板验证**

默认情况下，PromptTemplate 会检查传入的 input_variables 是否与模板中的占位符一致。你可以通过设置 validate_template=False 来禁用此验证（但建议谨慎使用）：

```python
template = "I am learning langchain because {reason}."
# 如果指定了额外的变量，会报错
# prompt_template = PromptTemplate(template=template, input_variables=["reason", "foo"])

# 关闭验证后，则不会报错
prompt_template = PromptTemplate(template=template, input_variables=["reason", "foo"], validate_template=False)
```

**5. Few-Shot Examples 的使用**

Few-Shot Examples 是一组示例对话、问题与答案，用于帮助语言模型理解任务要求。通过提供少量示例，模型可以模仿示例中的回答风格，从而生成更为合理的输出。

FewShotPromptTemplate 类结合了 PromptTemplate 与示例集合。其主要参数包括

- examples：示例列表，每个示例通常为字典形式（包含问题与答案）。
- example_prompt：用于格式化单个示例的 PromptTemplate。
- prefix：在示例之前的说明文本，通常用来描述任务要求。
- suffix：在示例之后的提示文本，通常用作用户输入的占位符。
- input_variables：最终生成的提示中需要传入的变量名称。
- example_separator：示例之间的连接符，常用 "\n\n"。

以生成单词反义词为例，完整示例如下：
```python
from langchain import PromptTemplate, FewShotPromptTemplate

# 定义 few-shot 示例
examples = [
    {"word": "happy", "antonym": "sad"},
    {"word": "tall", "antonym": "short"},
]

# 定义格式化单个示例的模板
example_formatter_template = """
Word: {word}
Antonym: {antonym}
"""
example_prompt = PromptTemplate(
    input_variables=["word", "antonym"],
    template=example_formatter_template,
)

# 创建 FewShotPromptTemplate
few_shot_prompt = FewShotPromptTemplate(
    examples=examples,
    example_prompt=example_prompt,
    prefix="Give the antonym of every input",
    suffix="Word: {input}\nAntonym:",
    input_variables=["input"],
    example_separator="\n\n",
)

# 生成最终提示
final_prompt = few_shot_prompt.format(input="big")
print(final_prompt)
```

生成的提示会包含前缀说明、格式化后的示例和后缀提示，帮助模型理解任务，从而生成反义词。
通过输入一些类似问题和答案，让模型参考学习，并在同一个prompt的末尾提出新的问题，以此来提升模型的推理能力
在LangChain中，需要使用 PromptTemplate 创建字符串提示模板。模板可以包括说明、少量示例以及适合给定任务的特定上下文和问题。因此，需要创建一个少量示例的列表。每个示例都是一个字典，其中键是输入变量，值是这些输入变量的值。

```python
from langchain_community.llms.tongyi import Tongyi  
from langchain_core.prompts import PromptTemplate, FewShotPromptTemplate  
  
examples = [  
    {  
        "question": "罗杰有五个网球，他又买了两盒网球，每盒有3个网球，请问他现在总共有多少个网球？",  
        "answer": "罗杰一开始有五个网球，又购买了两盒网球，每盒3个，共购买了6个网球，因此现在总共由5+6=11个网球。因此答案是11。"  
    },  
    {  
        "question": "食堂总共有23个苹果，如果他们用掉20个苹果，然后又买了6个苹果，请问现在食堂总共有多少个苹果？",  
        "answer": "食堂最初有23个苹果，用掉20个，然后又买了6个，总共有23-20+6=9个苹果，答案是9。"  
    },  
    {  
        "question": "杂耍者可以杂耍16个球。其中一半的球是高尔夫球，其中一半的高尔夫球是蓝色的。请问总共有多少个蓝色高尔夫球？",  
        "answer": "总共有16个球，其中一半是高尔夫球，也就是8个，其中一半是蓝色的，也就是4个，答案是4个。"  
    },  
]  
example_prompt = PromptTemplate(  
    input_variables=["question", "answer"], template="Question: {question}\n{answer}"  
)  
few_shot_prompt = FewShotPromptTemplate(  
    examples=examples,  
    example_prompt=example_prompt,  
    suffix="Question: {input}",  # 后缀模板，其中 {input} 会被替换为实际输入  
    input_variables=["input"]  # 定义输入变量的列表  
)  
question = '艾米需要4分钟才能爬到滑梯顶部，她花了1分钟才滑下来，水滑梯将在15分钟后关闭，请问在关闭之前她能滑多少次？'  
prompt_format = few_shot_prompt.format(input=question)  
llm = Tongyi()  
result = llm.invoke(prompt_format)  
print(result)
```

### 2. Predict（预测阶段）

- 功能：通过统一接口调用不同的大语言模型
- 特点：
    - 提供模型无关的通用接口
    - 支持多种LLM提供商（如OpenAI、Anthropic等）
    - 可灵活切换不同模型而不需修改业务逻辑
- 价值：实现了模型调用的抽象层，提高代码的可移植性

下面是对“LangChain Runnable介绍”这一部分的深入讲解，详细说明了其设计理念、主要功能和在构建 LLM 应用中的作用。

#### 2.1 LangChain Runnable 介绍（深入讲解）

##### 2.1.1 设计理念
Runnable 协议 是 LangChain 中的核心抽象，旨在为构建复杂的语言模型应用程序提供一个标准化、模块化的接口。其主要思想包括：

- 统一接口
无论是简单的文本转换、LLM 调用，还是复杂的链式处理任务，所有组件都可以遵循相同的协议。这样在设计应用时，就可以通过相同的方式调用不同的组件。
- 模块化与组合性
每个 Runnable 对象都专注于完成单一任务，而多个 Runnable 之间可以通过管道操作符（|）或其它组合方式串联起来，形成一个端到端的处理流程。这样的设计使得系统更易扩展、调试和维护。
- 支持同步与异步
在现代应用中，尤其是涉及 API 调用、网络请求等 I/O 密集型任务时，异步处理显得尤为重要。Runnable 协议设计了对应的同步（如 invoke、batch、stream）和异步方法（如 ainvoke、abatch、astream），使得同一组件能适用于不同的调用场景。
- 灵活的输入与输出验证
每个 Runnable 对象通过定义输入和输出的 schema，使得开发者在组合组件时可以明确知道每个步骤期望接收的数据格式和最终输出结果。这不仅帮助调试，也提高了代码的可读性和鲁棒性。


##### 2.1.2 主要功能

Runnable 对象主要包含以下几个核心功能：
-  **方法接口**
    - invoke / ainvoke
用于处理单个输入，分别对应同步和异步调用。例如，当你只需要处理一段文本时，可以使用这两个方法。
    - batch / abatch
用于并行处理多个输入。对于批量数据处理，通过这种方式可以显著提高效率，特别是利用线程池或异步协程的优势。
    - stream / astream
用于流式处理数据，适合处理需要逐步输出或实时展示结果的场景。流式接口可以一次处理一个数据块，并立即返回结果，适合大数据集的增量生成或实时响应。
-  **输入与输出 Schema**
  

每个 Runnable 对象都定义了：

 - 输入 schema：描述期望的输入数据格式和类型。
 - 输出 schema：说明生成的结果数据格式。
 - 配置 schema：有时还会有额外的配置信息，方便开发者在调用前进行检查和验证。


通过这些 schema 信息，开发者可以在组合链式流程时更容易进行调试和数据验证，确保每个组件之间的数据格式匹配。 


- **组合性**


    由于所有 Runnable 对象遵循相同的接口，开发者可以通过简单的管道符号（|）将不同的组件拼接起来，形成一个复杂的工作流程。这个组合过程类似于 Unix 系统中的管道命令，每个组件的输出直接作为下一个组件的输入。

- **并发支持**

    Runnable 对象支持异步执行（使用 asyncio 的 await 语法），使得可以同时运行多个任务。这对于调用外部 API、并发数据处理等场景非常关键，可以大幅度提升程序性能。

##### 2.1.3 在 LLM 应用中的作用
在构建大型语言模型（LLM）应用时，通常需要处理以下几个环节：
- 提示生成
根据用户输入生成适合 LLM 调用的提示（prompt）。
- 模型调用
调用语言模型，获取响应。
- 输出解析
对模型返回的结果进行解析或格式化，以便后续使用。
使用 Runnable 协议，可以将这些环节模块化，每个环节都用一个 Runnable 对象来实现。随后，通过管道操作符将这些组件组合起来，形成一个完整的处理链。例如：

```python
# 示例：构建一个简单的 LLM 调用链
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnableLambda, RunnableSequence
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

# 创建提示模板
prompt = ChatPromptTemplate.from_template("Tell me a joke about {topic}")

# 创建一个简单的链式流程
chain = prompt | ChatOpenAI() | StrOutputParser()

# 运行链，获取笑话
result = chain.invoke({"topic": "bears"})
print(result)
```

在这个例子中，每个组件（提示模板、模型调用、输出解析）都封装成一个 Runnable 对象，通过 | 操作符组合在一起。这样不仅代码简洁，而且每个模块都可以独立测试和维护。

##### 2.1.4 小结
- 统一与标准化： 
  Runnable 提供了一套标准接口，使得不同类型的任务可以用统一方式调用，无论是单个输入、批量处理还是流式处理。
- 模块化与灵活组合： 
  每个 Runnable 组件都专注于单一任务，通过组合操作可以构建复杂的处理流程，非常适合 LLM 应用中的提示生成、模型调用、输出解析等多步流程。
- 并发与异步支持：
  内置的异步方法和批处理接口，使得在处理 I/O 密集型任务时能够显著提高性能和响应速度。


通过对 Runnable 的深入理解，开发者可以更高效地构建、调试和扩展 LLM 应用程序，同时保持代码的清晰和模块化。这也是 LangChain 在处理复杂语言模型应用时备受推崇的重要原因之一。

### 3. Parse（解析阶段）
- 功能：处理和规范化模型输出
- 主要任务：
    - 从模型原始响应中提取关键信息
    - 按照预定格式规范化输出
    - 处理可能的异常响应
- 应用场景：输出结构化、标准化，便于后续处理或展示

在 LangChain 中，Output Parser 主要负责对大语言模型（LLM）生成的原始输出进行解析和转换，将其转化为更结构化、易于后续处理的数据格式。下面我们详细介绍一下它的作用、使用场景、内置解析器以及如何自定义输出解析器。

#### 3.1. Output Parser 的作用
- 结构化处理
LLM 通常返回自由文本输出，但很多应用需要将这些输出转换为结构化数据（例如字典、列表或 Pydantic 模型），以便进一步处理和验证。Output Parser 就是完成这种转换工作的组件。
- 数据校验与错误处理
通过使用输出解析器，可以对 LLM 的输出进行数据格式校验。比如使用 Pydantic 解析器时，会校验字段类型、必填项等，保证数据符合预期格式。如果格式不正确，可以及时发现问题并采取相应的处理措施。
- 简化后续逻辑
当输出解析器把输出转换为结构化数据后，后续的应用逻辑可以直接使用这些数据，而无需再手动处理字符串分割、正则匹配等操作，从而简化代码和流程设计。

#### 3.2. 内置的 Output Parser 类型
LangChain 提供了多种内置的输出解析器，满足不同的解析需求

##### 3.2.1 StrOutputParser
- 作用：直接返回原始字符串，不做额外解析。
- 适用场景：当你只需要获取 LLM 的文本输出，而无需进一步结构化时使用。
- 示例：
  
```python
from langchain_core.output_parsers import StrOutputParser

parser = StrOutputParser()
output = parser.parse("Hello, this is a response from LLM!")
print(output)  # 直接输出原文本
```

##### 3.2.2 JsonOutputParser
- 作用：将 LLM 输出的 JSON 格式文本解析为 Python 的字典（dict）。
- 适用场景：当 LLM 输出符合 JSON 标准格式时，使用该解析器可直接获取结构化数据。
- 示例：

```python
from langchain_core.output_parsers import JsonOutputParser

parser = JsonOutputParser()
llm_output = '{"name": "Alice", "age": 25}'
output = parser.parse(llm_output)
print(output)  # 输出: {'name': 'Alice', 'age': 25}
```

##### 3.2.3 PydanticOutputParser
- 作用：将 LLM 输出的 JSON 数据解析并转换为预先定义的 Pydantic 模型对象，同时进行数据校验。
- 适用场景：当你希望严格控制输出格式，并利用 Pydantic 提供的数据校验和默认值功能时使用。
- 示例：

```python
from langchain_core.output_parsers import PydanticOutputParser
from pydantic import BaseModel

class Person(BaseModel):
    name: str
    age: int
    city: str = "Unknown"  # 提供默认值

parser = PydanticOutputParser(pydantic_object=Person)
llm_output = '{"name": "Alice", "age": 25}'
output = parser.parse(llm_output)
print(output)
# 输出: Person(name='Alice', age=25, city='Unknown')
```

##### 3.2.4 CommaSeparatedListOutputParser

- 作用：将逗号分隔的字符串解析成 Python 列表。
- 适用场景：当 LLM 输出一个以逗号分隔的列表时使用。
- 示例：

```python
from langchain_core.output_parsers import CommaSeparatedListOutputParser

parser = CommaSeparatedListOutputParser()
output = parser.parse("apple, banana, orange")
print(output)  # 输出: ['apple', 'banana', 'orange']
```
##### 3.2.5 RegexParser
- 作用：利用正则表达式从 LLM 输出中提取出特定格式的数据。
- 适用场景：当输出格式比较复杂或不规则，但包含可以通过正则匹配的关键信息时使用。
- 示例：

```python
from langchain_core.output_parsers import RegexParser

# 例如 LLM 输出 "Name: Alice, Age: 25"
parser = RegexParser(
    regex=r"Name: (.*), Age: (\d+)",
    output_keys=["name", "age"]
)
output = parser.parse("Name: Alice, Age: 25")
print(output)  
# 输出: {'name': 'Alice', 'age': '25'}---
```
##### 3.2.6. 自定义 Output Parser
如果内置的解析器无法满足你的特定需求，你可以自定义解析器。通常的方法是继承 BaseOutputParser 并实现 parse 方法。
示例


```python
from langchain_core.output_parsers import BaseOutputParser

class CustomOutputParser(BaseOutputParser):
    def parse(self, text: str):
        # 自定义解析逻辑：例如将文本全部转为大写
        return text.upper()

parser = CustomOutputParser()
output = parser.parse("hello world")
print(output)  
# 输出: "HELLO WORLD"---
```
##### 3.2.7. Output Parser 在 Chain 中的应用


在构建 LangChain 的完整流程时，Output Parser 常与 Prompt、LLM 调用等组件结合，形成一个完整的数据流。例如，构建一个提取信息的链式流程时，你可能会这样组合：


```python
from langchain_core.prompts import PromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import JsonOutputParser

# 定义提示模板
prompt = PromptTemplate.from_template("Extract the following information in JSON: {input}")

# 定义 LLM（假设返回 JSON 格式字符串）
llm = ChatOpenAI(model="gpt-4", temperature=0)

# 定义输出解析器
output_parser = JsonOutputParser()

# 将它们通过管道组合成一个完整的 Chain
chain = prompt | llm | output_parser

# 运行链
result = chain.invoke({"input": "Name: Alice, Age: 25"})
print(result)  # 输出: {'name': 'Alice', 'age': 25}
```
在这个示例中：
- PromptTemplate 生成 LLM 调用的输入提示；
- ChatOpenAI 调用 LLM 生成输出；
- JsonOutputParser 将 LLM 输出解析为结构化 JSON 数据（Python 字典）。
---
##### 3.2.8. 小结
- Output Parser 的核心目标 是将 LLM 的自由文本输出转化为结构化数据，方便后续处理。
- 内置解析器涵盖了直接返回文本、解析 JSON、利用 Pydantic 进行数据校验、逗号分隔列表解析、正则表达式匹配等多种场景。
- 如果内置的解析器不满足需求，可以自定义解析逻辑，继承 BaseOutputParser 即可。
- 在 Chain 流程中，Output Parser 是连接 LLM 输出和应用逻辑的重要环节，确保数据格式一致和数据有效性。
通过合理使用 Output Parser，可以大大简化 LLM 应用的数据处理流程，提高代码的健壮性和可维护性。



## **Message**

LangChain抽象出来的消息类型有 AIMessage 、 HumanMessage 、 SystemMessage、 FunctionMessage 和 ChatMessage

- SystemMessage ：用于启动 AI 行为，作为输入消息序列中的第一个传入。

- HumanMessage ：表示来自与聊天模型交互的用户消息。

- AIMessage：表示来自聊天模型的消息。这可以是文本，也可以是调用工具的请求

下面是对 HumanMessage 类的中文笔记说明：

### HumanMessage 

HumanMessage 类（位于 langchain_core.messages.human 模块中）继承自
BaseMessage，用于表示来自人类的消息。这类消息通常作为输入传递给模型，让模型能基于人类的提问或指令进行回复。

#### 主要说明

- **用途**：

用于封装人类发送给模型的文本消息，通常用于对话场景中与模型交互。比如在对话系统中，HumanMessage 会包含用户的提问或指令内容。

- **示例**：

```python
from langchain_core.messages import HumanMessage, SystemMessage

messages = [
    SystemMessage(content="You are a helpful assistant! Your name is Bob."),
    HumanMessage(content="What is your name?")
]

### 实例化聊天模型，并使用 messages 进行调用
model = ...  
print(model.invoke(messages))
```


在这个示例中，系统消息（SystemMessage）用于设定对话背景（如助手的角色及名称），而
HumanMessage 则包含了用户实际提问的内容。

- **参数说明**：

  - **content**（必选）：

    消息的文本内容，可以是字符串或者列表（列表中的元素可以是字符串或字典），表示消息的具体内容。

  - **additional_kwargs**（可选）：
    额外的字典数据，用于传递与消息相关的额外负载数据。举例来说，对于 AI 消息，可能包含工具调用信息等。

  - **example**（可选，默认为 False）：

    用于标记该消息是否为示例对话的一部分，目前大部分模型对此参数忽略，不建议使用。

  - **id**（可选）：

    消息的唯一标识符，通常由生成该消息的模型或提供者提供。

  - **name**（可选）：

    可选的消息名称，用于提供更友好的名称说明。该字段的使用取决于具体模型的实现。

  - **response_metadata**（可选）：

    响应元数据，如响应头、logprobs、token 计数等信息。

  - **type**：

    消息类型，固定为 \'human\'，用于序列化时区分不同类型的消息。

- **方法**：

  - pretty_print()：

    打印消息的美化表示，返回值为 None。

  - pretty_repr(html: bool = False)：

    返回消息的美化表示字符串。参数 html 为 True 时，会使用 HTML标签格式化输出；默认值为 False。

  - text()：
    返回消息的纯文本内容。

### SystemMessage、AIMessage、FunctionMessage 和 ChatMessage

参数与内容大致与HumanMessage模块相同，只是type是system或ai或function或chat



## 数据连接

在构建检索增强生成（RAG）或知识增强型应用时，数据连接模块扮演了极其重要的角色。它负责从各种来源加载文档、将长文本拆分为适当的片段、并利用向量化技术将文本映射到高维空间，进而通过相似度搜索来增强大语言模型的回答。本文将从以下几个层面进行详细解析：

1. 文档加载器（Document Loaders）
2. 文本分块（Text Splitters）
3. 向量存储（Vector Stores）及各向量库详解（FAISS、Chroma、Milvus 等）
4. 综合应用与未来发展方向
---
### 1. 文档加载器（Document Loaders）
#### 1.1 作用与基本概念
文档加载器负责从不同格式的数据源中提取文本内容，转换为统一的内部表示（通常为 Document 对象）。这一步骤是所有后续处理（例如文本分块、向量化）的前提。
- 主要目标：从 PDF、网页、CSV、JSON、Docx 等格式中提取文本。
- 统一输出：所有加载的内容最终被格式化为标准的 Document 对象，包含文本内容、元数据（如页码、文件名、来源 URL 等）。
1.2 常见的加载器与示例
- PDF 加载器：
常用的有 PyMuPDFLoader 和 PDFMinerLoader。例如，使用 PyMuPDFLoader 加载 PDF：


```python
from langchain.document_loaders import PyMuPDFLoader
loader = PyMuPDFLoader("document.pdf")
docs = loader.load()
print(docs[0].page_content)
```

注意：针对不同 PDF 文件格式和内容，选择合适的解析库能显著提高准确性。

- **网页加载器**：
- PDF 加载器：

WebBaseLoader 支持从 URL 中抓取网页数据，适合加载实时新闻、博客文章等。
```python
from langchain.document_loaders import WebBaseLoader
loader = WebBaseLoader("https://example.com/article")
docs = loader.load()
```

- **其他格式加载器**：

  - **CSVLoader/JSONLoader**：直接解析结构化数据文件，适用于财务报表、调查问卷等。

  - **Docx2txtLoader**：专门用于加载 Word 文档，保持格式和段落信息。

#### 1.3 后续研究方向

- **多语言支持**：不同语言的文档可能需要定制化加载器。

- **高质量 OCR**：对于图片中的文本，可以结合 OCR 技术（如 Tesseract
  或商业 API）实现自动文本提取。

- **动态内容捕捉**：网页加载器可扩展支持 JavaScript 渲染页面（如使用
  Selenium 或 Playwright），确保抓取动态网页内容。

#### 1.4 PDF解析深入理解

##### 1.4.1 PDF解析概述

**PDF文档的特点**：PDF（Portable Document
Format）是一种广泛使用的文档格式，具有固定的布局和格式，能够在不同平台上保持一致的显示效果。然而，这种固定性也使得解析和提取其中的内容变得复杂。

**解析PDF的必要性**：在自然语言处理和信息检索等应用中，需要从PDF文档中提取文本、表格、图像等信息，以便进行进一步的分析和处理。因此，深入理解PDF解析的流程和技术对于构建高效的文档处理系统至关重要。

##### 1.4.2 PDF解析的关键环节和流程

1.  **文档预处理**：对PDF文档进行去噪、校正倾斜、二值化和增强对比度等处理，以提升文档质量，确保后续分析的准确性。

2.  **物理版面分析**：使用深度学习模型（如Faster
    R-CNN、YOLO等）检测文档的物理布局元素，识别标题、段落、图片和表格等区域。

3.  **文本区域分析**：进一步分析检测到的文本区域，识别单词、行和段落，可能涉及文本行提取和字符分割等任务。

4.  **内容识别**：应用OCR技术和表格解析等方法，提取文本、表格、公式等内容。

5.  **逻辑版面分析**：通过语义分析理解文档的结构和层次关系，将文本块组织成段落、列表等语义单元。

6.  **数据输出**：将分析结果以HTML、JSON等格式输出，便于后续处理和应用。

##### 1.4.3 PDF解析的关键技术

- **OCR技术**：光学字符识别（Optical Character
  Recognition）用于将图像中的文字内容转换为可编辑的文本格式，是PDF解析中的核心技术之一。

- **表格解析**：解析PDF中的表格结构，识别单元格的位置、行列关系等信息，将视觉信息转化为结构化数据。

- **公式识别**：将图像中的数学公式转换为LaTeX、MathML等格式，确保公式的正确显示和编辑

#### 1.5 PDF解析器分类

##### ✅ 1.5.1**按照功能分类整理开源框架**

 **📊 按照功能分类**

| 分类 | 框架名称 | 简要特点 |
|------|---------|----------|
| 传统型文档解析 | 1. MinerU | 完善的 PDF/Web 文档解析（结构保留、公式识别、表格转 HTML、OCR 多语支持），但不支持多级标题、垂直文字、特殊图书解析等。 |
| | 2. PaddleOCR | 功能全、生态强、模型丰富（如 UVDoc 矫正、LatexOCR、表格识别、布局检测），但更专注 OCR。 |
| | 3. Marker | 使用 Surya 多语言 OCR 支持，注重 Markdown/JSON 输出格式和结构保持。 |
| | 4. Unstructured | 专注数据预处理 ETL 流程，为训练/生产提供结构化输出，可作为处理管道中的一环。 |


**📊 多模态模型驱动**

| 框架名称  | 简要特点                                                                 |
|----------|--------------------------------------------------------------------------|
| gptpdf   | 首个尝试用 GPT-4o 处理 PDF 的项目，设计巧妙，但已停止维护                  |
| Zerox    | 多模态+图像转换流程，支持多格式文档，输出结构化 Markdown，已商业化为 OmniAI |



**📊 混合型/结合方案**

| 框架名称         | 简要特点                                                                 |
|------------------|--------------------------------------------------------------------------|
| Chunkr          | 多格式文档支持 + 语义标签布局分析，输出 HTML/Markdown，开源公司 Lumina 支持 |
| pdf-extract-api  | OCR 引擎 + 多模态模型 + LLM 校正，输出高质量 Markdown/JSON，OCR 模型灵活可切换 |
| Sparrow         | 模块化处理多种非结构化数据（如发票、银行单），适合结构化表单类型文档解析       |



**📊 RAG 系列文档组件**

| 框架名称               | 简要特点                                                                 |
|------------------------|--------------------------------------------------------------------------|
| DeepDoc (RAGFlow)      | 来自 RAGFlow，支持分块+解析，集成到 RAG 流程中使用                         |
| MegaParse (Quivr)      | Quivr 项目下的文档解析器，结构化输出，支持多文档聚合处理                    |
| 系列首篇 RAG 工具       | 文档未列名，指代首篇中介绍的通用 RAG 构建组件工具链                        |



**📊 核心能力对比表**

| 框架名               | 文档格式支持       | OCR支持 | 表格结构识别 | 多模态支持       | Markdown 输出 | 结构化保留 | 数学公式识别       | 商业化支持 |
|----------------------|--------------------|---------|--------------|------------------|---------------|------------|--------------------|------------|
| MinerU               | PDF/Web/ePub      | ✅      | ✅ (HTML)    | ❌               | ✅            | ✅ (段落、标题) | ✅ (LaTeX)        | ❌         |
| PaddleOCR            | 图片/PDF          | ✅      | ✅ (SLANet)  | ❌               | ✅            | ✅ (高精布局) | ✅ (LatexOCR)     | ✅         |
| Marker + Surya       | PDF               | ✅      | ✅           | ❌               | ✅            | ✅          | ✅                 | ✅         |
| Unstructured         | PDF/图像等        | ✅      | ✅           | ❌               | ✅            | ✅          | ❌                 | ✅         |
| gptpdf              | PDF               | ✅      | ❌           | ✅ (GPT-4o)      | ✅            | ✅ (语言感知) | ✅ ($$形式)       | ❌         |
| Zerox               | PDF/Docx          | ✅      | ❌           | ✅ (GPT)         | ✅            | ✅          | ✅                 | ✅         |
| Chunkr              | PDF/PPT等         | ✅      | ✅           | ✅ (语义标注)     | ✅            | ✅          | ❌                 | ✅ (Lumina) |
| pdf-extract-api      | PDF/图片          | ✅ (多种) | ✅           | ✅ (llama)       | ✅            | ✅          | ❌                 | ✅         |
| Sparrow             | 发票、收据等       | ✅      | ✅           | ✅ (模块化)       | ✅            | ✅          | ❌                 | ✅         |
| DeepDoc (RAGFlow)   | 文档合集           | ✅      | ✅           | ✅               | ✅            | ✅          | ✅                 | ✅         |
| MegaParse (Quivr)    | 多种格式           | ✅      | ✅           | ✅               | ✅            | ✅          | ✅                 | ✅         |
| RAG 工具链          | 不详               | ✅      | ✅           | ✅               | ✅            | ✅          | ✅                 | ✅         |


##### 📄 PDF解析工具对比分析：MinerU vs Marker

###### 🔍 对比分析

###### ✅ 总体结论

- **MinerU** 和 **Marker** 都是目前能满足 RAG 场景需求的开源 PDF
  解析工具；

- 两者在段落解析方面表现出色，能较好保持原文语义；

- **Marker** 能将表格结构输出为 Markdown 格式；

- **MinerU** 的版面分析较为精准，表格定位也较为准确；

- 两者都存在不同程度的问题，但整体效果尚可接受。

###### 🧯 问题与差异细节对比

###### ❌ 问题1：图片识别误差（PDF-Extract-Kit）

- **原始PDF（首页）**：包含正常文本块。

- **Marker**：✅ 正确识别为文本段落。

- **PDF-Extract-Kit**：❌ 将文本块识别为了图片，严重影响可提取性。

- **MinerU**：✅ 表现良好，无图片误识。

###### ❌ 问题2：表格识别错误

- **原始PDF**：包含标题行和数据表格。

- **Marker**：✅ 将表格转为 Markdown 格式，但**标题行解析错误**。

- **PDF-Extract-Kit**：❌ 无法识别表格，直接保存为图片。

- **MinerU**：✅ 表格定位准确，版面分析良好，但结构化输出仍待完善。

###### ❌ 问题3：目录识别问题

- **原始PDF**：具备清晰目录结构。

- **Marker**：❌ 将目录识别成了表格（结构错位）。

- **PDF-Extract-Kit**：✅ 正确解析出目录结构。

- **MinerU**：✅ 目录解析良好，基本保留结构与层级。

###### ❌ 问题4：标题识别遗漏

- **原始PDF**：存在多级标题。

- **Marker**：❌ 部分标题未能正确识别（如一级标题缺失）。

- **PDF-Extract-Kit**：✅ 标题识别准确，层级分明。

- **MinerU**：✅ 表现稳定，标题结构清晰。

###### ❌ 问题5：表格内容错乱

- **原始PDF**：存在嵌套或不规整表格。

- **Marker**：❌ 结构受损，部分行列错乱。

- **MinerU**：✅ 定位准确，但仍存在数据错行情况。

- **PDF-Extract-Kit**：❌ 表格仍被保存为图片，无法提取。


###### ✅ 总结对比表

 
  | **功能项**   |   **MinerU**|     **Marker**        |  **PDF-Extract-Kit**|
  |----------------------|--------------------|---------|--------------|
  | 段落提取      |  ✅ 优秀  |      ✅ 优秀           |  ❌ 经常误识为图像|
  | 表格定位      |  ✅ 准确  |      ✅ 可转md格式     |  ❌ 无法处理表格|
  | 表格结构解析    |⚠️ 有待完善  |  ⚠️ 标题行解析错误 |  ❌ 不支持|
  | 目录识别      |  ✅ 正确  |      ❌ 误识成表格     |  ✅ 准确|
  | 标题识别      |  ✅ 完整  |      ❌ 部分遗漏       |  ✅ 正确|
  | 图片识别误判    |✅ 准确  |      ✅ 准确           |  ❌ 误将文本识别为图像|
  | RAG 适配能力  |  ✅ 强  |        ✅ 强             |  ❌ 较差|
 
 
 ##### **基于MinerU/Marker文档解析**

Marker开源地址：<https://github.com/VikParuchuri/marker>

MinerU开源地址:
<https://github.com/opendatalab/MinerU/blob/master/README_zh-CN.md>

---

需要添加内容，关于MinerU/Marker文档解析使用方法研究
- 先列简单的方式，需要包含整个流程
- minerU的使用：https://mineru.readthedocs.io/en/latest/user_guide/usage/command_line.html
https://github.com/opendatalab/MinerU/blob/master/README_zh-CN.md
- marker的使用：https://github.com/VikParuchuri/marker


---


###### 🧠 1.5.3 **总结建议**

- **泛用性最强**：MinerU、PaddleOCR、Marker，适合研发人员做全流程解析。

- **AI 结合好**：Zerox、pdf-extract-api、Chunkr，把传统 OCR 与多模态/LLM
  有效融合。

- **RAG推荐首选**：DeepDoc、MegaParse，结构化+分块是 RAG 关键需求。

- **探索性项目**：gptpdf 是很有启发性的项目，虽然小但可学习其 prompt
  工程设计。

### 2. 文本分块（Text Splitters）

#### 2.1 为什么需要文本分块

长文档直接传给 LLM
处理可能导致上下文丢失、响应超长或令计算资源耗尽。文本分块（Chunking）解决了这一问题，通过拆分文本来：

- 保证单个块的长度适中，适合 LLM 生成或检索。

- 保持信息连贯，利用重叠机制减少分割带来的信息丢失。

#### 2.2 主要文本切分器介绍

- **RecursiveCharacterTextSplitter**：

这是最常用的分块器，它按层级顺序尝试先以段落、句子、最后退化到字符级别进行切分，同时允许块之间有一定重叠（例如
20 个字符），保证上下文连续性。

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
chunks = splitter.split_text(long_text)
```

- **CharacterTextSplitter**：

按固定字符数拆分，不考虑语义边界，适合简单场景。

- **NLTKTextSplitter**：

利用 NLTK 库根据标点和语法进行切分，适合需要语言学支持的场景。

- **MarkdownTextSplitter**：

针对 Markdown 格式文本进行优化，保留标题、列表等结构信息。

---
**需要添加内容，关于markdown+递归分块 结合 进行分块的研究**
MarkdownHeaderTextSplitter()+RecursiveCharacterTextSplitter()

好的例子：组织比较好的pdf内容

差的例子：组织比较差的pdf内容

---


#### 2.3 深入研究方向

- **自适应切分策略**：根据文档内容动态调整切分粒度，例如在专业文本中保持技术术语、公式不被拆分。

- **跨语言分块**：不同语言对标点和句子结构的处理不同，设计更智能的跨语言文本分块工具。

- **语义感知切分**：结合预训练模型，在切分时考虑语义边界，进一步提高后续向量检索的准确性。

### 3. 向量存储（Vector Stores）与向量库详解

向量存储将文本块通过嵌入函数转换为向量表示，并在高维空间中存储它们，以便进行相似度搜索。常见向量库包括
FAISS、Chroma、Milvus、Pinecone
等。下面详细介绍各个系统的特点与适用场景。

#### 3.1 向量化过程

- **嵌入模型**：常用的有 OpenAIEmbeddings、Sentence Transformers、CLIP
  等。

- **向量库目标**：快速检索最相似的文本块，常见的相似度计算有余弦相似度、内积、欧几里得距离等。

#### 3.2 FAISS

- **特点**：

  - Facebook 开源的向量搜索库，针对大规模、低延迟检索做了高度优化。

  - 支持 CPU 和 GPU 加速。

  - 适用于单机环境以及批量数据处理。

- **使用示例**：
  
```python
from langchain.vectorstores import FAISS
from langchain.embeddings.openai import OpenAIEmbeddings

texts = ["巴黎是法国的首都", "北京是中国的首都", "华盛顿是美国的首都"]
embeddings = OpenAIEmbeddings()
vectorstore = FAISS.from_texts(texts, embeddings)

# 检索示例
query = "美国的首都是什么？"
results = vectorstore.similarity_search(query)
print(results[0].page_content)
```

 **适用场景**：

  - 本地部署、单机批处理，快速原型设计和数据量中等的应用。

#### 3.3 Chroma

- **特点**：

  - 开源且轻量，专为快速原型和小规模应用设计。

  - 内存优先，易于集成在本地或小型服务器中。

- **使用示例**：

```python
from langchain.vectorstores import Chroma
from langchain.embeddings.openai import OpenAIEmbeddings

vectorstore = Chroma(collection_name="my_collection", embedding_function=OpenAIEmbeddings())
vectorstore.add_texts(["纽约是美国的城市", "伦敦是英国的城市"])
results = vectorstore.similarity_search("美国的城市")
print(results[0].page_content)
```

- **适用场景**：

  - 快速原型，数据量较小的检索任务，或需要简单部署的场景。

#### 3.4 Milvus

- **特点**：

  - 开源、分布式向量数据库，专为海量数据设计。

  - 支持水平扩展、分布式部署以及 GPU 加速。

  - 丰富的查询接口和数据管理能力，适合大规模应用场景。

- **使用示例**（假设已安装并启动 Milvus 服务）：

```python
from langchain.vectorstores import Milvus
from langchain.embeddings.openai import OpenAIEmbeddings

# 配置 Milvus 参数
vectorstore = Milvus(
    collection_name="my_collection",
    embedding_function=OpenAIEmbeddings(),
    connection_args={"host": "localhost", "port": "19530"}
)

# 添加数据
docs = ["东京是日本的首都", "首尔是韩国的首都"]
vectorstore.add_texts(docs)

# 相似度搜索
results = vectorstore.similarity_search("日本的首都是什么？")
print(results[0].page_content)  # 应输出："东京是日本的首都"
```


- **适用场景**：

  - 海量数据检索、跨节点分布式部署、大规模应用。

  - 适用于企业级搜索、推荐系统以及实时在线服务。

- **深入研究方向**：

  - 集成更高效的更新和删除机制，支持实时数据流入。

  - 利用 Milvus 的 GPU
    加速和分布式架构，研究高并发检索场景下的性能调优。

  - 探索与多模态数据（如图像、视频）融合的混合检索方案。

#### 3.5 Pinecone、Weaviate 等云服务

- **Pinecone**：

  - 云托管向量数据库，自动扩展、高可用，适合无运维负担的场景。

  - 与 LangChain 集成简单，但成本和隐私需要考虑。

- **Weaviate**：

  - 不仅支持向量检索，还结合了知识图谱等功能，适用于需要关系查询的场景。

- **发展方向**：

  - 研究混合部署方案（本地与云端结合），优化成本与性能平衡。

  - 探索针对特定领域（如医疗、金融）的定制化向量检索系统。

### 4. 综合应用与未来发展方向

#### 4.1 端到端数据处理流程

一个完整的数据连接流程通常包括：

1.  **文档加载**：用不同加载器获取文本和元数据。

2.  **文本切分**：利用智能切分器生成适当长度的文本块，保留语义信息。

3.  **向量化**：调用嵌入模型（如 OpenAIEmbeddings）将文本转为向量。

4.  **向量存储**：将向量存储在 FAISS、Chroma、Milvus 等数据库中。

5.  **语义检索**：基于查询文本生成向量，进行相似度搜索，检索相关文档块。

6.  **后续处理**：检索到的文档可以用于上下文增强、问答生成或其他应用。

#### 4.2 未来研究和优化方向

- **自适应文本切分策略**

研究如何根据文档内容自动调整切分策略，利用深度模型判断语义边界，提升上下文连续性和检索效果。

- **混合向量数据库方案**

探索在不同场景下（例如高并发与大数据量）如何选择或混合使用 FAISS、Milvus
与云服务，实现性能与成本的最佳平衡。

- **高效嵌入模型**

随着模型不断升级，研究如何结合最新的嵌入模型（例如多模态嵌入、领域特定嵌入）提高检索相关性。

- **数据预处理与清洗**

在文档加载阶段结合
OCR、多语言处理、噪声过滤等方法，提高后续流程的准确性。

- **实时数据更新**

研究如何在向量数据库中实现高效的增量更新、删除和在线重构，以支持实时应用场景。

- **端到端调优与评估**

设计标准评测指标（例如检索准确率、响应时间），对整个数据处理管道进行系统评估，确保整体性能符合预期。
