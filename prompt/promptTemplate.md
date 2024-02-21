# f-string
f-string 是 Python 中用于格式化字符串的一种方式。它是在 Python 3.6 版本中引入的。f-string 的名称来自于在字符串前加上 'f' 字母，表示这是一个格式化字符串。

使用 f-string，你可以在字符串中嵌入变量、表达式和函数调用，而无需使用传统的字符串格式化方法（如 `%` 操作符或 `str.format()` 方法）。这使得代码更简洁、可读性更强，并提供了更直观的语法。

以下是一个简单的示例，演示了使用 f-string 格式化字符串的方式：

```python
name = "Alice"
age = 30

# 使用 f-string 格式化字符串
message = f"Hello, my name is {name} and I am {age} years old."

# 打印结果
print(message)
```

在上面的例子中，`{name}` 和 `{age}` 是 f-string 的表达式部分，它们会被相应的变量值替代。f-string 还支持在表达式中执行任意的 Python 代码，使得字符串格式化更加灵活。

需要注意的是，f-string 只在 Python 3.6 及以上的版本中可用。如果你使用的是较旧的 Python 版本，你可能需要考虑其他字符串格式化的方法。

在langchainjs中，f-string则指能够变量（使用{}包裹）的字符串。

我们并不需要知道它具体是如何解析的，只要知道是什么就行。

# 模板解析
在langchains中解析模板是使用parseTemplate方法，而渲染模板则是使用renderTemplate，其定义如下：

```typescript
// 渲染模板
export const renderTemplate = (
  template: string,
  templateFormat: TemplateFormat,
  inputValues: InputValues
) => DEFAULT_FORMATTER_MAPPING[templateFormat](template, inputValues);

// 解析模板
export const parseTemplate = (
  template: string,
  templateFormat: TemplateFormat
) => DEFAULT_PARSER_MAPPING[templateFormat](template);

// 内置的只有一种模板引擎f-string
export const DEFAULT_PARSER_MAPPING: Record<TemplateFormat, Parser> = {
  "f-string": parseFString,
};

// 如果是变量的节点则使用values中对应的变量替换
export const interpolateFString = (template: string, values: InputValues) =>
  parseFString(template).reduce((res, node) => {
    if (node.type === "variable") {
      if (node.name in values) {
        return res + values[node.name];
      }
      throw new Error(`Missing value for input ${node.name}`);
    }

    return res + node.text;
  }, "");

// 具体的解析变量或者是文本节点的实现
export const parseFString = (template: string): ParsedFStringNode[] => {}
// 该模板引擎只有两种节点，变量或者文本
export type ParsedFStringNode =
  | { type: "literal"; text: string }
  | { type: "variable"; name: string };
```

在开发过程中，我们经常遇见到`throw new Error(`Missing value for input ${node.name}`);`这个缺少变量错误，实际上就是在这里报错的。

## fromTemplate
我们可以直接从字符串创建PromptTemplate，利用的就是promptTemplate.fromTemplate。

其原理就是使用上面的parseTemplate获取到所有的变量节点，然后初始化PromptTemplate即可。

# promptValue
现在的大模型有两种形式，一种是直接一段prompt，一种是按照对话的形式messages。

在langchains中对这两种做了区分：

llm：只接受一段字符串的prompt
chatModel：接受对话形式的prompt(message)

另外，openai对这两种则是以另外的形式区分。比如llm叫做instruct，chatModel叫做chat。

实际上两种是一样的，只是接收的prompt的参数不一样而已。

## promptValue
在langchains中，因为区分为llm和chatModel，因此prompt也有不同的类型。

> 多模态的prompt则不止字符串，还有图片、视频、文件等等，这里我们只谈论文生文模型。

```typescript
export interface BasePromptValueInterface extends Serializable {
  toString(): string;

  toChatMessages(): BaseMessage[];
}
// 对应llm
export interface StringPromptValueInterface extends BasePromptValueInterface {
  value: string;
}
对应chat model
export interface ChatPromptValueInterface extends BasePromptValueInterface {
  messages: BaseMessage[];
}
```

从另外一个角度来说，messages实际上可以看出是StringPromptValue的数组形式。

# PromptTemplate

PrompteTemplate实际上也是一个runable，也就是说拥有invoke方法。

其类的定义如下：

```typescript
export class PromptTemplate<
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    RunInput extends InputValues = any,
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    PartialVariableName extends string = any
  >
  extends BaseStringPromptTemplate<RunInput, PartialVariableName>
  implements PromptTemplateInput<RunInput, PartialVariableName>
{}
```

继承了`BaseStringPromptTemplate`：

```typescript
export abstract class BaseStringPromptTemplate<
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  RunInput extends InputValues = any,
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  PartialVariableName extends string = any
> extends BasePromptTemplate<
  RunInput,
  StringPromptValueInterface,
  PartialVariableName
> {
  /**
   * Formats the prompt given the input values and returns a formatted
   * prompt value.
   * @param values The input values to format the prompt.
   * @returns A Promise that resolves to a formatted prompt value.
   */
  async formatPromptValue(
    values: TypedPromptInputValues<RunInput>
  ): Promise<StringPromptValueInterface> {
    const formattedPrompt = await this.format(values);
    return new StringPromptValue(formattedPrompt);
  }
```

从上一个小结我们知道，messages是stringPromptValue的数组形式。那么如果要创建message，则我们需要使用：

```typescript
let msg = [
    PromptTemplate.fromTemplate('xxx1'),
    PromptTemplate.fromTemplate('xxx2'),
];
```

langchains封装ChatMessagePromptTemplate用于承载这种类型。









