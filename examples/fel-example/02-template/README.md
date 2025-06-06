# 提示词

提示词(Prompt)在大语言模型应用中至关重要，他是提供给大模型的一段文字、指令或者图片和文件。使用提示词可以引导模型生成更加明确的内容，也用来限制和减少模型生成无用内容的次数，提高对话的效率。

``` java
public interface Prompt {
    List<ChatMessage> messages();
    default String text() {...}
}
public class ChatMessages implements Prompt {
    private final List<ChatMessage> messages = new ArrayList<>();
    // 省略构建器和方法
}
```

下图展示了消息框架下的接口和实现：

``` plantuml
@startuml
   interface ChatMessage {
   }

   abstract class AbstractChatMessage implements ChatMessage {
       - final String content
       + String text()
       + String toString()
   }

   class AiMessage extends AbstractChatMessage {
       - final List<ToolCall> toolCalls
       + MessageType type()
       + List<ToolCall> toolCalls()
       + boolean isToolCall()
       + String toString()
   }

   class HumanMessage extends AbstractChatMessage {
       - final List<Media> medias
       + MessageType type()
       + List<Media> medias()
   }

   class SystemMessage extends AbstractChatMessage {
       + MessageType type()
   }

   class ToolMessage extends AbstractChatMessage {
       - final String id
       + ToolMessage(String id, String text)
       + Optional<String> id()
       + MessageType type()
       + String toString()
   }
   @enduml
```

`AbstractChatMessage`和它的实现包括了 4 种消息类型。

消息类型主要用于定义发出消息的一方的身份：
- `system`：表示系统消息，用于指导大模型的行为和输出规范，可以定义表达风格或者制定规则来影响模型的回应方式和输出。
- `human`：表示人类消息，用于给大模型提问题，发指令或是陈述需求，人类消息通常占据对话的一方。
- `ai`：表示 ai 消息，用于返回大模型的输出，构成了对话的另一方。
- `tool`：表示工具消息，用于返回大模型调用工具后的结果输出。一些大模型支持工具的调用，工具消息可以满足大模型与工具之间的交互。

``` java
public enum MessageType {
    SYSTEM("system"),
    HUMAN("human"),
    AI("ai"),
    TOOL("tool");

    private final String role;

    private static final Map<String, MessageType> RELATIONSHIP =
        Arrays.stream(MessageType.values()).collect(
            Collectors.toMap(MessageType::getRole, Function.identity()));

    MessageType(String role) {
        this.role = role;
    }

    public static MessageType parse(String role) {
        return RELATIONSHIP.getOrDefault(role, MessageType.HUMAN);
    }
}
```

## 模板

模板的核心作用是提供一个预定义的格式或者结构，用来快速生成具有一致性的内容。

通用泛型提示模板接口定义如下：

``` java
public interface GenericTemplate<I, O> {
    O render(I values);
    Set<String> placeholder();
}
```

- `O render(I values)`：根据输入参数渲染模板，生成结果。
- `Set<String> placeholder()`：用于获取模板占位符集合。

### 字符串模板

字符串模板基于 [mustache](https://mustache.github.io/) 语法的文本模板引擎，输入从字符串到字符串的映射用于填充模板占位符，输出为字符串。字符串模板支持完全填充和部分填充。

``` java
public interface StringTemplate extends GenericTemplate<Map<String, String>, String> {}
```

#### 完全填充

完全填充 `render()` 需要一次性填充全部的占位符，示例：

``` java
String template = "给我讲个关于{{adjective}}的{{content}}。";
Map<String, String> values = 
        MapBuilder.<String, String>get().put("adjective", "兔子").put("content", "故事").build();
String output = new DefaultStringTemplate(template).render(values);
```

```plaintext
给我讲个关于兔子的故事。
```

#### 部分填充

部分填充可以先用 `partial()` 填充部分占位符，然后再用 `render()` 填充剩余的占位符，示例：

``` java
String template = "给我讲个关于{{adjective}}的{{content}}。";
StringTemplate partial = new DefaultStringTemplate(template).partial("adjective", "兔子");
```

```plaintext
给我讲个关于兔子的故事。
给我讲个关于兔子的笑话。
```
### 消息模板

消息内容可以是简单的文本，也可以包含图片信息。根据模型可接收的消息类型不同可以使用不同的方法来包装内容。

根据角色不同，模板分成了 `HumanMessageTemplate` 和 `SystemMessageTemplte` ，分别用于渲染人类消息和系统消息。

示例：

``` java
MessageTemplate template = new HumanMessageTemplate("我喜欢{{staff1}}, {{staff2}}还有{{staff3}}");
Tip tip = new Tip().add("staff1", MessageContent.from("唱歌", new Media("image/png", "singing.png")))
        .add("staff2", MessageContent.from("跳舞", new Media("image/png", "dance.png")))
        .add("staff3", MessageContent.from("打篮球", new Media("image/png", "basketball.png")));
ChatMessage message = template.render(tip.freeze());
System.out.println(message.text())
```

```markdown
我喜欢唱歌，跳舞还有打篮球。
```
### 示例

1. 在项目 pom.xml 加入以下依赖：

``` xml
    <dependencies>
        <dependency>
            <groupId>org.fitframework</groupId>
            <artifactId>fit-starter</artifactId>
            <version>${fit.version}</version>
        </dependency>
        <dependency>
            <groupId>org.fitframework</groupId>
            <artifactId>fit-plugins-starter-web</artifactId>
            <version>${fit.version}</version>
        </dependency>
        <dependency>
            <groupId>org.fitframework.plugin</groupId>
            <artifactId>fit-http-client-okhttp</artifactId>
            <version>${fit.version}</version>
        </dependency>
        <dependency>
            <groupId>org.fitframework</groupId>
            <artifactId>fel-core</artifactId>
            <version>${fel.version}</version>
        </dependency>
        <dependency>
            <groupId>org.fitframework</groupId>
            <artifactId>fel-model-openai-plugin</artifactId>
            <version>${fel.version}</version>
        </dependency>
    </dependencies>
```

2. 在 application.yml 配置文件中加入以下配置：

```yaml
fel:
  openai:
    api-base: '${api-base}'
    api-key: '${your-api-key}'
example:
  model: '${model-name}'
```

3. 添加如下代码：

``` java
@Component
@RequestMapping("/ai/example")
public class ChatTemplateExampleController {
    private final ChatModel chatModel;
    private final MessageTemplate template;
    @Value("${example.model}")
    private String modelName;

    public ChatTemplateExampleController(ChatModel chatModel) {
        this.chatModel = chatModel;
        this.template = new HumanMessageTemplate("给我讲个关于{{adjective}}的{{content}}。");
    }

    @GetMapping("/chat")
    public ChatMessage chat(@RequestParam("adjective") String adjective, @RequestParam("content") String content) {
        ChatOption option = ChatOption.custom().model(this.modelName).stream(false).build();
        return this.chatModel.generate(ChatMessages.from(this.template.render(Tip.from("adjective", adjective)
                .add("content", content)
                .freeze())), option).first().block().get();
    }
}
```

### 验证

在浏览器栏输入：`http://localhost:8080/ai/example/chat?adjective=兔子&content=笑话`

返回如下响应：

```json
{
  "content": "当然，这是一个关于兔子的笑话：\n\n为什么兔子眼睛是红的？\n\n因为它们胡萝卜吃太多了，以至于连看眼科医生的时候，医生都说：“你的眼睛真胡萝卜！” \n\n不过，兔子听到医生这么说，反而很高兴，因为它一直想成为一个“胡萝卜”眼的超级英雄。所以，它决定多吃胡萝卜，让眼睛变得更红，成为森林里的“胡萝卜眼兔侠”！\n\n当然，这只是个笑话，实际上兔子眼睛红是因为它们的眼睛里有丰富的血管，而不是因为吃太多胡萝卜。",
  "toolCalls": []
}
```

修改输入参数：`http://localhost:8080/ai/example/chat?adjective=老虎&content=笑话`

```json
{
  "content": "当然，这是一个关于老虎的笑话：\n\n为什么老虎不喜欢吃人？\n\n因为人太多骨头了！\n\n希望这能让你开心一笑！",
  "toolCalls": []
}
```