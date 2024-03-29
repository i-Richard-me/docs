---
title: "从不同规模模型效果认识模型局限"
description: ""
---

模型的参数规模与其处理任务的能力之间存在明显的相关性。在本文中，我对比了三个不同规模的模型（Qwen-14B、Yi-34B、Qwen-72B）在执行相同的文本分类任务时的表现。

## 任务说明

我们依旧构建一个文本分类任务，文本数据模拟在绩效奖金发放后的员工反馈。分类涵盖回复有效性、是否含有敏感信息、回复内容的主题、以及情绪的正负向四个方面。

任务设置模仿了实际工作场景中可能遇到的复杂情况，旨在深入探讨模型在理解自然语言方面的细微差别。

:::note
测试的文本样本使用 ChatGPT 辅助生成，如有雷同，纯属巧合。
:::

## 任务构建代码

```python
# 省略导入库和环境配置

# 构建提示词
classification_prompt = """
    作为一个NLP专家，你需要评估员工敬业度调研中的回复内容。请按照以下步骤操作：
    
    1. 回复有效性判断：若回复非常短，如一个词、符号或空白，判断为“无效”。超过10个字即为“有效”。
    2. 情感分类：根据回复内容，将情绪归类为“正向”、“中性”或“负向”。注意回复的情感色彩、态度和情绪。对于使用反话或反讽的回复，尝试识别实际意图，并据此分类。
    3. 主题分类：根据回复的具体内容，将其归类到以下类别，并仅输出分类名称。以下是分类名称的表格形式呈现：
    
        | 分类名称       |
        |--------------------|
        | 上级沟通       |
        | 公开公示       |
        | 标准透明       |
        | 客观公正       |
        | 目标管理       |
        | 系统体验       |
        | 政策制定       |
        | 年终奖打折       |
    
    4. 是否敏感信息：根据回复内容，判断是否包含敏感信息，并填写“是”或“否”。敏感信息包括：
        - 提到了具体人名或具体部门名称
        - 举报、投诉所在部门的上级管理者的严重管理问题
        - 敏感信息的标准是基于员工回复中的具体内容，特别是涉及具体个人、部门名称或举报投诉管理问题的情况。
    
    请基于提供的回复内容做出判断，避免任何推测或脑补。
    
    要点提醒：
    - 直接回答每个任务的问题。
    - 使用明确的词汇“有效”或“无效”来描述回复的有效性。
    - 确保情感分类结果仅为“正向”、“中性”或“负向”之一。
    - 在主题分类任务中，确保仅返回表中的分类名称。
    - 在判断是否包含敏感信息时，使用“是”或“否”来明确回答。
    
    员工ID与员工回复 >>>{query}<<<
    
    按以上要求输出结果，不要包含任何额外信息
    \n{format_instructions}
    """

class classification_format(BaseModel):
    """
    定义输出格式
    """
    ID: str = Field(description="ID")
    validity: str = Field(description="回复的有效性。必须为`有效`或`无效`。")
    sentiment_class: str = Field(description="回复的情感倾向。可能的值为`正向`、`中性`或`负向`。")
    topic: list = Field(description="基于回复内容识别出的主题分类，根据预定义的主题分类列表中选择。")
    sensitive_info: str = Field(description="是否包含敏感信息。必须为`是`或`否`。")
    
parser = JsonOutputParser(pydantic_object=classification_format)

prompt = PromptTemplate(
    template=classification_prompt,
    input_variables=["query"],
    partial_variables={"format_instructions": parser.get_format_instructions()},
)

chain = prompt | model | parser

# 创建一个空的 dataframe 用于存储分类结果
result_df = pd.DataFrame()

# 执行分类任务
for i in tqdm(range(len(data))):
    classification_query = get_query(i)
    chain_result = chain.invoke({"query": classification_query})
    chain_result['topic'] = str(chain_result['topic'])  # 由于文本主题可能有多个，转换为dataframe时会报错，因此先将topic列表转为文本格式
    result_i = pd.DataFrame(chain_result, index=[0])
    result_df = pd.concat([result_df, result_i], axis=0)
```

## 模型效果对比

---

### Qwen-14B

Qwen-14B 模型显示出了一些明显的理解和分类错误：

1. 第一个样本的主题被错误地分类为`感谢`，这个分类并不在任务的预设分类中。
2. 第二个样本被错误地标记为`有效`，尽管它实际上是无效的回复。
3. 第四个样本在敏感信息判断中被错误的分成了`否`，但回复中实名投诉了部门领导。

<iframe width="784" style="height: 292px;" src="https://datalore.jetbrains.com/report/embed/IRsLD9S3oA5isRQeLedT3y/88rSYL3Xdk9jUc4KGvXcLh/BqqZhSFRjsnarihG4sWXTH?height=292" frameborder="0"></iframe>

### Yi-34B

在测试 Yi-34B 模型过程中，发现其对某些专有名词或小众词汇缺乏理解：

- 在第三个样本中，员工抗议年终奖打折，模型错误地将情感分类为`正向`。经过测试，当直接问模型什么是年终奖打折时，模型无法给出正确答案。

<iframe width="784" style="height: 292px;" src="https://datalore.jetbrains.com/report/embed/IRsLD9S3oA5isRQeLedT3y/88rSYL3Xdk9jUc4KGvXcLh/Xh14nfsvP1C7WeORCwVxmG?height=292" frameborder="0"></iframe>

### Qwen-72B

当模型规模扩大到 Qwen-72B 时，模型几乎在所有测试样本上都表现出色，没有明显错误。

<iframe width="784" style="height: 292px;" src="https://datalore.jetbrains.com/report/embed/IRsLD9S3oA5isRQeLedT3y/88rSYL3Xdk9jUc4KGvXcLh/UtagecK3srdDG0EVo76fzT?height=292" frameborder="0"></iframe>

## 总结

1. **模型遗忘现象**：在 Qwen-14B 模型测试中，主题被错误地分类提示词未提到的分类，意味着模型输出时遗忘了任务的具体要求。一般来说，在输入相同的情况下，模型规模越大，遗忘现象越少。
2. **原生知识缺失**：在 Yi-34B 模型测试中，模型对于特定领域的知识缺失导致了错误的情感分类，即模型的训练数据中就没有包含相关知识。
3. **模型规模与性能**：一般来说，模型规模越大，在语意理解和推理方面的能力就越强。在 Qwen-72B 模型测试中，模型几乎没有出现错误，表现出色。
4. **硬件资源需求**：在以上对比中没有提到的是，越大的模型规模需要越多的硬件资源，因此在实际应用中，需要权衡模型效果与硬件资源之间的平衡。

:::caution
以上对比为了突显模型间的差异，提示词构建较为简单，有故意设计的成分。在实际应用中，有很多调优方法可以提升模型的表现，即使14B模型在调优后也可以在这个任务上表现得很好。

解决以上模型局限的方法包括：
- 在提示词中重复任务要求，减少模型遗忘现象。
- 在提示词中告诉模型关于特定词汇的知识。
- 将表现不好的样本作为few-shot示例。
- ...
:::