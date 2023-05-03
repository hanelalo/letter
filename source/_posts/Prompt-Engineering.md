---
title: Prompt Engineering
date: 2023-05-03 15:46:56
tags: AIGC
categories: Prompt
description: 吴恩达的 Prompt Engineering 课程的总结笔记。
---

# 写在前面

本文是学习吴恩达的 Prompt Engineering 课程的总结笔记。

课程地址：https://learn.deeplearning.ai/chatgpt-prompt-eng/lesson/2/guidelines

本文提到的实例代码依赖 openai 这个 python 库，所以需要事先进行安装：

```Shell
$ pip install openai
```

另外，还需要设置 openai 的 API Key：

```Python
 import openai
 openai.api_key="sk-...."
```

其中的 API Key 可以点击这个链接获取自己账户的 API Key：https://platform.openai.com/account/api-keys

另外，本文的代码基本都会包含对以下辅助函数的调用。

```Python
def get_completion(prompt, model="gpt-3.5-turbo"): 
    messages = [{"role":"user","content":prompt}]
    response = openai.ChatCompletion.create(
        model=model,
        messages=messages,
        temperature=0,
    )
    return response.choices[0].message["content"]
```

prompt 就是输入的文本了，model 是使用的模型，这里硬编码成了 gpt-3.5-turbo。

更多的 api 使用规范参考：https://platform.openai.com/docs/api-reference

# Temperature 参数

temperature 参数的取值是 0~2，这个值越高，输出越随机，值越低，输出越稳定。

同样的 prompt，当 temperature 为 0 时，每次输出的内容基本能够保持一致，但是如果 temperature 是 0.8，或者其他数值时，每次输出的结果可能会不一样。

所以，如果构建的系统需要保持输出的稳定性，temperature 参数的值建议设置成 0，而如果想让应用更具有创造性，则可以适当调高该参数的值。

# 使用准则

- 编写清晰而具体的指令。
- 给予模型思考的时间。

## 编写清晰而具体的指令

需要注意的是，不要把短的提示词和清晰、具体的提示词混淆，很多时候，更长的提示词能更清晰的描述问题的上下文，让模型能更准确的输出。

要编写一个清晰、准确的提示词，可以使用以下 4 种策略。

### 使用分隔符

使用分隔符，可以明确指出输入的不同部分。

比如使用 `"""`、`````、`<>` 、`---`等。

~~~Python
text = f"""
You should express what you want a model to do by \ 
providing instructions that are as clear and \ 
specific as you can possibly make them. \ 
This will guide the model towards the desired output, \ 
and reduce the chances of receiving irrelevant \ 
or incorrect responses. Don't confuse writing a \ 
clear prompt with writing a short prompt. \ 
In many cases, longer prompts provide more clarity \ 
and context for the model, which can lead to \ 
more detailed and relevant outputs.
"""
prompt = f"""
Summarize the text delimited by triple backticks \ 
into a single sentence.
```{text}```
"""
response = get_completion(prompt)
print(response)
~~~

上面的例子中的 prompt 是让 gpt 模型将 `````中的文本总结成 1 句话，这里使用 `````作为分隔符，有效的限定了文本的不同部分之间的边界。

这样做的好处是，可以有效地防止 prompt 注入。举个例子，如果这里没有指定某种符号作为分隔符，那么如果用户输入的 text 的内容是让 gpt 忽略前面的 prompt 内容并写一首关于熊猫的诗，那么整个应用将不会按照预想的逻辑运行。

总之，使用分隔符的好处是：

1. 使 prompt 更加清晰，gpt 能更清晰识别 prompt 文本中的不同部分。
2. 防止 prompt 注入。

### 要求模型进行结构化的输出

可以要求  gpt 输出一个 json、xml，甚至 html。

举个例子：

```Python
prompt = f"""
Generate  a list of three made-up book title along \
with their authors and genres.
Provide them in JSON format with the follow key:
book_id,title,author,genre.
"""
response = get_completion(prompt)
print(response)
```

Output：

```JSON
{
  "book1": {
    "title": "The Lost City of Zorath",
    "author": "Aria Blackwood",
    "genre": "Fantasy"
  },
  "book2": {
    "title": "The Last Survivors",
    "author": "Ethan Stone",
    "genre": "Science Fiction"
  },
  "book3": {
    "title": "The Secret Life of Mrs. Jenkins",
    "author": "Lila Rosewood",
    "genre": "Mystery"
  }
}
```

### 要求模型检查条件是否得到满足

可以要求模型做一些前置的检查，如果不满足这些检查条件时，则提示并终止任务。

比如，现在给 gpt 一段文本，要求它对文本进行总结，需要总结 step by step 的结构进行输出，如果提供的文本不满足这种结构，就提示。

```python
text_1 = f"""
Making a cup of tea is easy! First, you need to get some \ 
water boiling. While that's happening, \ 
grab a cup and put a tea bag in it. Once the water is \ 
hot enough, just pour it over the tea bag. \ 
Let it sit for a bit so the tea can steep. After a \ 
few minutes, take out the tea bag. If you \ 
like, you can add some sugar or milk to taste. \ 
And that's it! You've got yourself a delicious \ 
cup of tea to enjoy.
"""
prompt = f"""
You will be provided with text delimited by triple quotes. 
If it contains a sequence of instructions, \ 
re-write those instructions in the following format:

Step 1 - ...
Step 2 - …
…
Step N - …

If the text does not contain a sequence of instructions, \ 
then simply write \"No steps provided.\"

\"\"\"{text_1}\"\"\"
"""
response = get_completion(prompt)
print("Completion for Text 1:")
print(response)
```

这里要求对三引号中的文本进行总结，并按照 Step 1 ~ Step N 进行输出，如果 text 的内容不存在这样的结构，就仅仅只是输出`No Stpes provided`。

Output：

```
Completion for Text 1:
Step 1 - Get some water boiling.
Step 2 - Grab a cup and put a tea bag in it.
Step 3 - Once the water is hot enough, pour it over the tea bag.
Step 4 - Let it sit for a bit so the tea can steep.
Step 5 - After a few minutes, take out the tea bag.
Step 6 - Add some sugar or milk to taste.
Step 7 - Enjoy your delicious cup of tea!
```

当 text 的内容不存在这种结构时，比如 text 仅仅只有 Hello 时：

Output：

```
Completion for Text 1:
No steps provided.
```

### 向模型提供一些提示样本

给予模型一些成功的完整样本，并让执行任务。

比如，要求模型以一致的对话风格完善文本。

```python
prompt = f"""
Your task is to answer in a consistent style.

<child>: Teach me about patience.

<grandparent>: The river that carves the deepest \ 
valley flows from a modest spring; the \ 
grandest symphony originates from a single note; \ 
the most intricate tapestry begins with a solitary thread.

<child>: Teach me about resilience.
"""
response = get_completion(prompt)
print(response)
```

Output：

```
<grandparent>: Resilience is like a tree that bends with the wind but never breaks. It is the ability to bounce back from adversity and keep moving forward, even when things get tough. Just like a tree that grows stronger with each storm it weathers, resilience is a quality that can be developed and strengthened over time.
```

这是一个小孩在跟祖父母对话的场景，小孩说“教我什么是忍耐”时，祖父母的回答使用了比喻的方式来进行阐述。

那么当孩子说“什么是适应力”时，gpt 的回答也使用了比喻来进行阐述。

## 给予模型思考的时间

当模型短时间内得不到答案时，应该重新设计 prompt，让模型在得出答案前有一系列推理，最终得到答案。

另外，给模型一个复杂任务时，如果模型无法短时间内完成，或者无法用少量的词完成时，往往会**编造**出一些错误的答案。

所以，在这个准则上，也提供了以下几个准则。

### 指定完成一项任务所需要的步骤

比如下面的例子：

~~~python
text = f"""
In a charming village, siblings Jack and Jill set out on \ 
a quest to fetch water from a hilltop \ 
well. As they climbed, singing joyfully, misfortune \ 
struck—Jack tripped on a stone and tumbled \ 
down the hill, with Jill following suit. \ 
Though slightly battered, the pair returned home to \ 
comforting embraces. Despite the mishap, \ 
their adventurous spirits remained undimmed, and they \ 
continued exploring with delight.
"""
# example 1
prompt_1 = f"""
Perform the following actions: 
1 - Summarize the following text delimited by triple \
backticks with 1 sentence.
2 - Translate the summary into French.
3 - List each name in the French summary.
4 - Output a json object that contains the following \
keys: french_summary, num_names.

Separate your answers with line breaks.

Text:
```{text}```
"""
response = get_completion(prompt_1)
print("Completion for prompt 1:")
print(response)
~~~

这个 prompt 让模型做了四件事并用换行符分隔：将`````中的内容总结成为一句话，将总结的内容翻译成法语，列出法语版的摘要总结中出现了每个名字，输出一个包含 french_summary、num_names 两个 key 的 json 文本。

Output:

```Python
Completion for prompt 1:
Two siblings, Jack and Jill, go on a quest to fetch water from a hilltop well, but misfortune strikes as they both fall down the hill, yet they return home slightly battered but with their adventurous spirits undimmed.

Deux frères et sœurs, Jack et Jill, partent en quête d'eau d'un puits au sommet d'une colline, mais ils tombent tous les deux et retournent chez eux légèrement meurtris mais avec leur esprit d'aventure intact. 
Noms: Jack, Jill.

{
"french_summary": "Deux frères et sœurs, Jack et Jill, partent en quête d'eau d'un puits au sommet d'une colline, mais ils tombent tous les deux et retournent chez eux légèrement meurtris mais avec leur esprit d'aventure intact.",
"num_names": 2
}
```

更进一步，对于输出的格式，还可以通过 prompt 进一步定制：

```Python
prompt_2 = f"""
Your task is to perform the following actions: 
1 - Summarize the following text delimited by 
  <> with 1 sentence.
2 - Translate the summary into French.
3 - List each name in the French summary.
4 - Output a json object that contains the 
  following keys: french_summary, num_names.

Use the following format:
Text: <text to summarize>
Summary: <summary>
Translation: <summary translation>
Names: <list of names in Italian summary>
Output JSON: <json with summary and num_names>

Text: <{text}>
"""
response = get_completion(prompt_2)
print("\nCompletion for prompt 2:")
print(response)
```

Output：

```
Completion for prompt 2:
Summary: Jack and Jill go on a quest to fetch water, but misfortune strikes and they tumble down the hill, returning home slightly battered but with their adventurous spirits undimmed. 
Translation: Jack et Jill partent en quête d'eau, mais la malchance frappe et ils dégringolent la colline, rentrant chez eux légèrement meurtris mais avec leurs esprits aventureux intacts.
Names: Jack, Jill
Output JSON: {"french_summary": "Jack et Jill partent en quête d'eau, mais la malchance frappe et ils dégringolent la colline, rentrant chez eux légèrement meurtris mais avec leurs esprits aventureux intacts.", "num_names": 2}
```

会发现 prompt 中使用`<>`这个符号的地方其实不止一个，但依然不影响模型的运行，并给出了指定格式的输出内容。

> 从通常的编程语言的角度来看，这其实很神奇，虽然我们的提示词中有说总结 <> 中的内容，但整个提示词中不止一个地方有 <>，模型竟然自动识别到了要总结的文本是 Text 后面的内容。
>
> 这或许就是 ai 的魅力所在。

### 在得到答案前，先自定寻找解决方案

下面的例子：

```Python
prompt = f"""
Determine if the student's solution is correct or not.

Question:
I'm building a solar power installation and I need \
 help working out the financials. 
- Land costs $100 / square foot
- I can buy solar panels for $250 / square foot
- I negotiated a contract for maintenance that will cost \ 
me a flat $100k per year, and an additional $10 / square \
foot
What is the total cost for the first year of operations 
as a function of the number of square feet.

Student's Solution:
Let x be the size of the installation in square feet.
Costs:
1. Land cost: 100x
2. Solar panel cost: 250x
3. Maintenance cost: 100,000 + 100x
Total cost: 100x + 250x + 100,000 + 100x = 450x + 100,000
"""
response = get_completion(prompt)
print(response)
```

Output:

```Plain
The student's solution is correct.
```

这个 Prompt 是要模型判定学生的解题是否正确。

问题是：每平方土地成本 100 美元，每平方太阳能电池板 250 美元，签订了每年固定 100000 美元的维护合同，每平方还有额外的 10 美元成本，那么第一年运营的总成本是多少？

学生解题：假设有 x 平方，那么总成本为 100x + 250x + 100000 + 100x = 450x + 100000。

模型判定结果：学生是对的。

但实际上，是错误的，在这种情况下，我们可以试着让模型自己先解题，然后再和学生的解决方案对比，让判定结果更加准确。

~~~Python
prompt = f"""
Your task is to determine if the student's solution \
is correct or not.
To solve the problem do the following:
- First, work out your own solution to the problem. 
- Then compare your solution to the student's solution \ 
and evaluate if the student's solution is correct or not. 
Don't decide if the student's solution is correct until 
you have done the problem yourself.

Use the following format:
Question:
```
question here
```
Student's solution:
```
student's solution here
```
Actual solution:
```
steps to work out the solution and your solution here
```
Is the student's solution the same as actual solution \
just calculated:
```
yes or no
```
Student grade:
```
correct or incorrect
```

Question:
```
I'm building a solar power installation and I need help \
working out the financials. 
- Land costs $100 / square foot
- I can buy solar panels for $250 / square foot
- I negotiated a contract for maintenance that will cost \
me a flat $100k per year, and an additional $10 / square \
foot
What is the total cost for the first year of operations \
as a function of the number of square feet.
``` 
Student's solution:
```
Let x be the size of the installation in square feet.
Costs:
1. Land cost: 100x
2. Solar panel cost: 250x
3. Maintenance cost: 100,000 + 100x
Total cost: 100x + 250x + 100,000 + 100x = 450x + 100,000
```
Actual solution:
"""
response = get_completion(prompt)
print(response)
~~~

Output:

```Plain
Let x be the size of the installation in square feet.

Costs:
1. Land cost: 100x
2. Solar panel cost: 250x
3. Maintenance cost: 100,000 + 10x

Total cost: 100x + 250x + 100,000 + 10x = 360x + 100,000

Is the student's solution the same as actual solution just calculated:
No

Student grade:
Incorrect
```

这个新的 prompt 让模型先自己对问题进行解决，然后再和学生的解决方案对比，再判断和学生的解决方案是否一致，再判定学生的解决方案是否正确。

为了确保是模型自己解决这个问题，prompt 中还特地强调在自己完成解决方案之前不能判定学生的方案的对错：Don't decide if the student's solution is correct until you have done the problem yourself.

# 模型的局限性

因为模型读取了大量的数据，但是并不能完整准确地记录所有的内容，所以当它回答一些晦涩难懂的问题时，可能会产生幻觉，会编造一些看起来真实但实际上是并不真实的回答。这也是 gpt 模型目前已知的一个缺陷。

```Python
prompt="简单讲讲刘备和曹操结拜兄弟的故事"
response = get_completion(prompt)
print(response)
```

Output:

```Python
刘备和曹操结拜兄弟的故事源于《三国演义》。据传，刘备和曹操在东汉末年，都是出身名门的豪族，但由于时局动荡，两人都失去了家族的地位和财富。他们在一次相遇后，互相交流了彼此的遭遇和心情，深感彼此的志向和品德相投，于是决定结为兄弟。
刘备和曹操在结拜时，按照古代的习俗，要进行三次拜礼。第一次拜礼时，刘备说：“我刘备愿与你结为兄弟，如同手足之情，共同扶持天下。”曹操回答：“我曹操也愿与你结为兄弟，如同肝胆相照，共同建功立业。”第二次拜礼时，两人交换了誓言，表示要共同努力，不离不弃。第三次拜礼时，两人用刀割破手指，混合鲜血，喝下了誓血，表示要永远保持兄弟之情。
刘备和曹操结拜后，曹操帮助刘备重建了家族，刘备也在曹操的帮助下，成为了一名有名的将领。但是，由于两人的政治理念和野心不同，最终还是走上了对立的道路，成为了三国时期的两大敌对势力。
```

为了减少这种幻觉，你可以在 prompt 中指定模型在引用某些资源的基础上作答，这样能更好的避免出现“幻觉”。

# 总结

- 如果构建的系统需要保持输出的稳定性，temperature 参数的值建议设置成 0。
- 对与 gpt 模型的使用，为了让它的回答更准确，指令需要清晰、具体，但并不代表指令一定非常短，同时还可以通过增加分隔符来防止 prompt 注入，通过限制输出的结构、指定条件的检查、prompt 带上提示样本等策略，亦可以提高输出的准确性。
- 对于比较复杂的任务，可以告诉模型，完成这个任务需要的步骤，让模型一步一步进行推进，而不是一步到位。
- 模型的输出，在面对晦涩难懂的任务时，会出现“幻觉”，这种情况，可以在 prompt 指定让模型在引用某些资源的基础上进行回答，可以尽量避免出现“幻觉”。
