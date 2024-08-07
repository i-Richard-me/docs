---
title: "不同规模模型在复杂文本分类中的表现"
description: "全面深入地比较三种不同规模的大模型在处理复杂且多维度的文本分类任务时的性能表现，详细探讨模型规模与分类效果之间的紧密关联，旨在为自然语言处理领域的模型选择和应用提供有价值的参考。"
---

在自然语言处理（NLP）领域，模型的参数规模与其处理复杂任务的能力之间存在着密切的关联。
本文中将对比三种不同规模的大型语言模型（Qwen-14B、Yi-34B、Qwen-72B）在处理较高难度文本分类任务中的表现差异，探讨模型规模与性能的关系。

## 任务说明

为了模拟实际工作场景中可能遇到的复杂情况，我们构建了一个多维度的文本分类任务。该任务基于员工在调研中反馈，涵盖了以下四个分类维度：

1. 回复有效性
2. 敏感信息识别
3. 回复内容主题
4. 情感倾向

这种多维度的分类任务不仅要求模型具备基本的文本理解能力，还需要在多个层面上进行细致的语义分析和推理。
通过这样的设置，我们可以更全面地评估不同规模模型在处理复杂NLP任务时的表现差异。

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

# 执行分类任务
result_df = pd.DataFrame()
for i in tqdm(range(len(data))):
    classification_query = get_query(i)
    chain_result = chain.invoke({"query": classification_query})
    chain_result['topic'] = str(chain_result['topic'])
    result_i = pd.DataFrame(chain_result, index=[0])
    result_df = pd.concat([result_df, result_i], axis=0)
```

这段代码展示了如何使用LangChain框架构建一个文本分类任务。
我们定义了输出格式、构建了详细的提示词，并使用JsonOutputParser确保模型输出符合预定义的格式。

## 实验结果与分析

---

:::note
测试的文本样本使用 ChatGPT 辅助生成，提到的人名均为虚构。如有雷同，纯属巧合。

利用ChatGPT生成文本样本的方法，在上一篇[利用大模型提升情感分类任务准确性](sentiment_classification)文章给出了参考的方法和提示词示例。
:::

### Qwen-14B

Qwen-14B 模型显示出了一些明显的理解和分类错误：

1. 主题分类错误：在第一个样本中，模型将主题错误地分类为"感谢"，这个类别并不在预设的分类列表中。这反映出模型可能存在"遗忘"现象，即在生成答案时忽略了任务中明确定义的分类范围。
2. 有效性判断失误：第二个样本（"无意见"）被错误地标记为"有效"，根据任务定义，应该被视为无效回复。
3. 敏感信息识别不准：第四个样本中，尽管回复明确提到了具体人名并投诉了管理问题，模型却未能正确识别其中的敏感信息。

```text title="Output"
+----+--------------------------------------------+---------+------------+-------------+--------------------------+-------------+
|    | subjective_answer                          | ID      | validity   | sentiment   | topic                    | sensitive   |
|----+--------------------------------------------+---------+------------+-------------+--------------------------+-------------|
|  0 | 感谢公司，感谢老板！                       | id00000 | 有效       | 正向        | ['感谢']                 | 否          |
|  1 | 无意见                                     | id00001 | 有效       | 中性        | ['无意见']               | 否          |
|  2 | 希望年终奖不要再打折了                     | id00002 | 有效       | 负向        | ['年终奖打折']           | 否          |
|  3 | 整个评估过程中领导刘海涛并未与我进行交流。 | id00003 | 有效       | 负向        | ['上级沟通']             | 否          |
|  4 | 完全是一人独裁，只有上级说了算，私下报复。 | id00004 | 有效       | 负向        | ['上级沟通', '客观公正'] | 是          |
+----+--------------------------------------------+---------+------------+-------------+--------------------------+-------------+
```

### Yi-34B

Yi-34B模型相比Qwen-14B有所改进，但其对某些专有名词或新兴词汇缺乏理解：

- 在第三个样本中，模型错误地将"希望年终奖不要再打折了"这一明显负面情绪的回复分类为"正向"。这是由于模型对"年终奖打折"这一词汇缺乏了解，当以聊天方式询问模型什么是年终奖打折时，模型也无法给出准确的解释，甚至被认为是一种员工福利。

```text title="Output"
+----+--------------------------------------------+---------+------------+-------------+----------------+-------------+
|    | subjective_answer                          | ID      | validity   | sentiment   | topic          | sensitive   |
|----+--------------------------------------------+---------+------------+-------------+----------------+-------------|
|  0 | 感谢公司，感谢老板！                       | id00000 | 有效       | 正向        | ['上级沟通']   | 否          |
|  1 | 无意见                                     | id00001 | 有效       | 中性        | ['上级沟通']   | 否          |
|  2 | 希望年终奖不要再打折了                     | id00002 | 有效       | 正向        | ['年终奖打折'] | 否          |
|  3 | 整个评估过程中领导刘海涛并未与我进行交流。 | id00003 | 有效       | 负向        | ['上级沟通']   | 是          |
|  4 | 完全是一人独裁，只有上级说了算，私下报复。 | id00004 | 有效       | 负向        | ['上级沟通']   | 是          |
+----+--------------------------------------------+---------+------------+-------------+----------------+-------------+
```

### Qwen-72B

当模型规模扩大到 Qwen-72B 时，模型几乎在所有测试样本上都表现出色，没有明显错误。

```text title="Output"
+----+--------------------------------------------+---------+------------+-------------+----------------+-------------+
|    | subjective_answer                          | ID      | validity   | sentiment   | topic          | sensitive   |
|----+--------------------------------------------+---------+------------+-------------+----------------+-------------|
|  0 | 感谢公司，感谢老板！                       | id00000 | 有效       | 正向        | ['上级沟通']   | 否          |
|  1 | 无意见                                     | id00001 | 无效       | 无法判断    | []             | 否          |
|  2 | 希望年终奖不要再打折了                     | id00002 | 有效       | 负向        | ['年终奖打折'] | 否          |
|  3 | 整个评估过程中领导刘海涛并未与我进行交流。 | id00003 | 有效       | 负向        | ['上级沟通']   | 是          |
|  4 | 完全是一人独裁，只有上级说了算，私下报复。 | id00004 | 有效       | 负向        | ['上级沟通']   | 是          |
+----+--------------------------------------------+---------+------------+-------------+----------------+-------------+
```

## 总结

1. **模型遗忘现象**：在 Qwen-14B 模型测试中，主题被错误地分类提示词未提到的分类，意味着模型输出时遗忘了任务的具体要求。一般来说，在输入相同的情况下，模型规模越大，遗忘现象越少。
2. **原生知识缺失**：在 Yi-34B 模型测试中，模型对于特定领域的知识缺失导致了错误的情感分类，即模型的训练数据中就没有包含相关知识。
3. **模型规模与性能**：一般来说，模型规模越大，在语意理解和推理方面的能力就越强。在 Qwen-72B 模型测试中，模型几乎没有出现错误，表现出色。
4. **硬件资源需求**：在以上对比中没有提到的是，越大的模型规模需要越多的硬件资源，因此在实际应用中，需要权衡模型效果与硬件资源之间的平衡。

:::caution
以上对比为了突显模型间的差异，提示词构建较为简单，有故意设计的成分。

在实际应用中，有很多调优方法可以提升模型的表现，即使14B模型在调优后也可以在这个任务上表现得很好。

解决以上模型局限的方法包括：
- 在提示词中重复任务要求，减少模型遗忘现象。
- 在提示词中告诉模型关于特定词汇的知识。
- 将表现不好的样本作为few-shot示例。
- ...
:::