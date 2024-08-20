## 一、Prompt构成
- **指示:** 对任务进行描述
- **上下文:** 给出与任务相关的其它背景信息（尤其在多轮交互中）
- **例子:** 必要时给出举例，学术中称为 one-shot learning, few-shot learning 或 in-context learning；实践证明其对输出正确性有帮助
- **输入:** 任务的输入信息；在提示词中明确的标识出输入
- **输出:** 输出的格式描述，以便后继模块自动解析模型的输出结果，比如（JSON、XML）

### 2.1、设定一个业务场景来讲解上述知识

**业务场景：办理流量包的智能客服**

流量包产品：

| 名称 | 流量（G/月） | 价格（元/月） | 适用人群 |
| :---: | :---: | :---: | :---: | 
| 经济套餐 | 10 | 50 | 无限制 |
| 畅游套餐 | 100 | 180 | 无限制 |
| 无限套餐 | 1000 | 300 | 无限制 |
| 校园套餐 | 200 | 150 | 在校生 |


### 2.2、对话系统的基本模块（简介）

<img src="dm.png" width=600px>

对话流程举例：

<img src="dialog.png" width=600px>

### 2.3、用Prompt实现上述模块功能

**环境搭建**

调试 prompt 的过程其实在图形界面里开始会更方便，但为了方便演示和大家上手体验，我们直接在代码里调试。

**环境搭建**

```python
#免费获取？？可以阅读wenda.py llms/llm_openai.py plugins/common.py 理解配置加载过程免费获取GPT_APIhttps://github.com/chatanywhere/GPT_API_free

# 加载环境变量 ???.env 还是需要在macOS系统中进行配置，这两步都是需要的吗
from openai import OpenAI
import os

from dotenv import load_dotenv, find_dotenv
_ = load_dotenv(find_dotenv())  # 读取本地 .env 文件，里面定义了 OPENAI_API_KEY
openai.api_key = os.getenv('OPENAI_API_KEY')

#在pycharm中.env文件中定义 
OPENAI_API_KEY = sk-D2fd3wTDzH3BR1B9zO0FaJDSFBbD9QfMur90d19ljEzl5PMz
```

macOS系统，在shell配置文件中设置环境变量

```shell
#打开 Shell 配置文件，根据您的 shell 类型，找到相应的配置文件:
vi ~/.bash_profile #对于Bash
vi ~/.zshrc #对于Zsh
```

文件打开后，点击键盘i，进入编辑状态，如区域出现insert，然后在文本末尾增加 ` export OPENAI_API_KEY="sk-D2fd3wTDzH3BR1B9zO0FaJDSFBbD9QfMur90d19ljEzl5PMz"` 

然后退出 esc 然后输入: wq

激活配置，通过运行命令 `source ~/.bash_profile `或 `source ~/.zshrc `来使更改生效

 验证环境变量设置，运行命令 `echo $OPENAI_API_KEY `来验证环境变量是否已成功设置。如果一切正常，它应该显示您API 密钥。

```python
#Jupyter 中的环境
import os
from openai import OpenAI
client = OpenAI(
  api_key = os.environ['OPENAI_API_KEY'],
  base_url="https://api.chatanywhere.com.cn/v1"
)

# 基于prompt生成文本
def get_completion(prompt, client, model="gpt-3.5-turbo"):
  messages = [{"role": "user", "content": prompt}]
  response = client.chat.completions.create(
  model=model,
  messages=messages,
  temperature=0,
  )
  return response.choices[0].message.content
```

- NLU

  **任务描述+输入**

  ```python
  # 任务描述
  instruction = """
  你的任务是识别用户对手机流量套餐产品的选择条件。
  每种流量套餐产品包含三个属性：名称，月费价格，月流量。
  根据用户输入，识别用户在上述三种属性上的倾向。
  """
  
  # 用户输入
  input_text = """
  办个100G的套餐。
  """
  
  # 这是系统预置的 prompt。魔法咒语的秘密都在这里
  prompt = f"""
  {instruction}
  
  用户输入：
  {input_text}
  """
  
  response = get_completion(prompt, client)
  print(response)
  ```

  ```
  根据用户输入，可以看出用户在流量这一属性上的倾向是希望拥有较大的流量，即100G。因此，用户可能更倾向于选择月流量较大的手机流量套餐产品。
  ```

  **约定输出格式** 

  ```python
  # 输出描述
  output_format = """
  以JSON格式输出
  """
  
  # 稍微调整下咒语
  prompt = f"""
  {instruction}
  
  {output_format}
  
  用户输入：
  {input_text}
  """
  
  response = get_completion(prompt, client)
  print(response)
  ```

  ```
  {
      "属性": "月流量",
      "倾向": "100G"
  }
  ```

  **把描述定义更精细**

  ```python
  #任务描述
  instruction = """
  你的任务是识别用户对手机流量套餐产品的选择条件。
  每种流量套餐产品包含三个属性：名称(name)，月费价格(price)，月流量(data)。
  根据用户输入，识别用户在上述三种属性上的倾向。
  """
  #用户输入
  input_text = "有没有便宜的套餐"
  #输出描述
  output_format = """
  以JSON格式输出。
  1. name字段的取值为string类型，取值必须为以下之一：经济套餐、畅游套餐、无限套餐、校园套餐 或 null；
  
  2. price字段的取值为一个结构体 或 null，包含两个字段：
  (1) operator, string类型，取值范围：'<='（小于等于）, '>=' (大于等于), '=='（等于）
  (2) value, int类型
  
  3. data字段的取值为取值为一个结构体 或 null，包含两个字段：
  (1) operator, string类型，取值范围：'<='（小于等于）, '>=' (大于等于), '=='（等于）
  (2) value, int类型或string类型，string类型只能是'无上限'
  
  4. 用户的意图可以包含按price或data排序，以sort字段标识，取值为一个结构体：
  (1) 结构体中以"ordering"="descend"表示按降序排序，以"value"字段存储待排序的字段
  (2) 结构体中以"ordering"="ascend"表示按升序排序，以"value"字段存储待排序的字段
  
  只输出中只包含用户提及的字段，不要猜测任何用户未直接提及的字段，不输出值为null的字段。
  """
  #系统预置的prompt
  prompt = f"""
  {instruction}
  
  {output_format}
  
  用户输入：
  {input_text}
  """
  response = get_completion(prompt,client)
  print(response)
  ```

  ```
  {
    "name": "经济套餐",
    "sort": {
      "ordering": "ascend",
      "value": "price"
    }
  }
  ```

  **加入例子：让输出更稳定**

  ```python
  #prompt里面有{output_format}，但是并没有像前面的代码一样定义
  examples = """
  便宜的套餐：{"sort":{"ordering"="ascend","value"="price"}}
  有没有不限流量的：{"data":{"operator":"==","value":"无上限"}}
  流量大的：{"sort":{"ordering"="descend","value"="data"}}
  100G以上流量的套餐最便宜的是哪个：{"sort":{"ordering"="ascend","value"="price"},"data":{"operator":">=","value":100}}
  月费不超过200的：{"price":{"operator":"<=","value":200}}
  就要月费180那个套餐：{"price":{"operator":"==","value":180}}
  经济套餐：{"name":"经济套餐"}
  """
  
  #input_text = "办个200G的套餐"
  #input_text = "有没有流量大的套餐"
  input_text = "200元以下，流量大的套餐有啥"
  #input_text = "你说那个10G的套餐，叫啥名字"
  
  prompt = f"""
  {instruction}
  
  {output_format}
  
  例如：
  {examples}
  
  用户输入：
  {input_text}
  
  """
  
  response = get_completion(prompt,client)
  print(response)
  ```

  ```
  {"price":{"operator":"<=","value":200},"sort":{"ordering":"descend","value":"data"}}
  ```

  优先用Prompt解决问题，用好Prompt可以减轻预处理和后处理的工作量和复杂度。

- 实现上下文DST

  即在Prompt的example中加入上下文，但是example太复杂，调优复杂

- 实现NLG和对话策略

  ```python
  # 加载环境变量
  from openai import OpenAI
  import os, json, copy
  
  client = OpenAI(
    api_key = os.environ['OPENAI_API_KEY'],
    base_url="https://api.chatanywhere.com.cn/v1"
  )
  
  instruction = """
  你的任务是识别用户对手机流量套餐产品的选择条件。
  每种流量套餐产品包含三个属性：名称(name)，月费价格(price)，月流量(data)。
  根据用户输入，识别用户在上述三种属性上的倾向。
  """
  # 输出描述
  output_format = """
  以JSON格式输出。
  1. name字段的取值为string类型，取值必须为以下之一：经济套餐、畅游套餐、无限套餐、校园套餐 或 null；
  
  2. price字段的取值为一个结构体 或 null，包含两个字段：
  (1) operator, string类型，取值范围：'<='（小于等于）, '>=' (大于等于), '=='（等于）
  (2) value, int类型
  
  3. data字段的取值为取值为一个结构体 或 null，包含两个字段：
  (1) operator, string类型，取值范围：'<='（小于等于）, '>=' (大于等于), '=='（等于）
  (2) value, int类型或string类型，string类型只能是'无上限'
  
  4. 用户的意图可以包含按price或data排序，以sort字段标识，取值为一个结构体：
  (1) 结构体中以"ordering"="descend"表示按降序排序，以"value"字段存储待排序的字段
  (2) 结构体中以"ordering"="ascend"表示按升序排序，以"value"字段存储待排序的字段
  
  只输出中只包含用户提及的字段，不要猜测任何用户未直接提及的字段。
  DO NOT OUTPUT NULL-VALUED FIELD! 确保输出能被json.loads加载。
  """
  examples = """
  便宜的套餐：{"sort":{"ordering"="ascend","value"="price"}}
  有没有不限流量的：{"data":{"operator":"==","value":"无上限"}}
  流量大的：{"sort":{"ordering"="descend","value"="data"}}
  100G以上流量的套餐最便宜的是哪个：{"sort":{"ordering"="ascend","value"="price"},"data":{"operator":">=","value":100}}
  月费不超过200的：{"price":{"operator":"<=","value":200}}
  就要月费180那个套餐：{"price":{"operator":"==","value":180}}
  经济套餐：{"name":"经济套餐"}
  """
  class NLU:
      def __init__(self):
          self.prompt_template = f"{instruction}\n\n{output_format}\n\n{examples}\n\n用户输入：\n__INPUT__"
      def _get_completion(self, prompt, model="gpt-3.5-turbo"):
          messages = [{"role": "user", "content": prompt}]
          response = client.chat.completions.create(
              model=model,
              messages=messages,
              temperature=0,  # 模型输出的随机性，0 表示随机性最小
          )
          semantics = json.loads(response.choices[0].message.content)
          return { k:v for k,v in semantics.items() if v }
      def parse(self, user_input):
          prompt = self.prompt_template.replace("__INPUT__", user_input)
          return self._get_completion(prompt)
  
  class DST:
      def __init__(self):
          pass
  
      def update(self, state, nlu_semantics):
          if "name" in nlu_semantics:
              state.clear()
          if "sort" in nlu_semantics:
              slot = nlu_semantics["sort"]["value"]
              if slot in state and state[slot]["operator"] == "==":
                  del state[slot]
          for k, v in nlu_semantics.items():
              state[k] = v
          return state
  
  class MockedDB:
      def __init__(self):
          self.data = [
              {"name": "经济套餐", "price": 50, "data": 10, "requirement": None},
              {"name": "畅游套餐", "price": 180, "data": 100, "requirement": None},
              {"name": "无限套餐", "price": 300, "data": 1000, "requirement": None},
              {"name": "校园套餐", "price": 150, "data": 200, "requirement": "在校生"},
          ]
  
      def retrieve(self, **kwargs):
          records = []
          for r in self.data:
              select = True
              if r["requirement"]:
                  if "status" not in kwargs or kwargs["status"] != r["requirement"]:
                      continue
              for k, v in kwargs.items():
                  if k == "sort":
                      continue
                  if k == "data" and v["value"] == "无上限":
                      if r[k] != 1000:
                          select = False
                          break
                  if "operator" in v:
                      if not eval(str(r[k])+v["operator"]+str(v["value"])):
                          select = False
                          break
                  elif str(r[k]) != str(v):
                      select = False
                      break
              if select:
                  records.append(r)
          if len(records) <= 1:
              return records
          key = "price"
          reverse = False
          if "sort" in kwargs:
              key = kwargs["sort"]["value"]
              reverse = kwargs["sort"]["ordering"] == "descend"
          return sorted(records, key=lambda x: x[key], reverse=reverse)
  
  class DialogManager:
      def __init__(self, prompt_templates):
          self.state = {}
          self.session = [
              {
                  "role": "system",
                  "content": "你是一个手机流量套餐的客服代表，你叫小瓜。可以帮助用户选择最合适的流量套餐产品。"
              }
          ]
          self.nlu = NLU()
          self.dst = DST()
          self.db = MockedDB()
          self.prompt_templates = prompt_templates
  
      def _wrap(self, user_input, records):
          if records:
              prompt = self.prompt_templates["recommand"].replace(
                  "__INPUT__", user_input)
              r = records[0]
              for k, v in r.items():
                  prompt = prompt.replace(f"__{k.upper()}__", str(v))
          else:
              prompt = self.prompt_templates["not_found"].replace(
                  "__INPUT__", user_input)
              for k, v in self.state.items():
                  if "operator" in v:
                      prompt = prompt.replace(
                          f"__{k.upper()}__", v["operator"]+str(v["value"]))
                  else:
                      prompt = prompt.replace(f"__{k.upper()}__", str(v))
          return prompt
      def _call_chatgpt(self, prompt, model="gpt-3.5-turbo"):
          session = copy.deepcopy(self.session)
          session.append({"role": "user", "content": prompt})
          response = client.chat.completions.create(
              model=model,
              messages=session,
              temperature=0,
          )
          return response.choices[0].message.content
  
      def run(self, user_input):
          # 调用NLU获得语义解析
          semantics = self.nlu.parse(user_input)
          print("===semantics===")
          print(semantics)
  
          # 调用DST更新多轮状态
          self.state = self.dst.update(self.state, semantics)
          print("===state===")
          print(self.state)
  
          # 根据状态检索DB，获得满足条件的候选
          records = self.db.retrieve(**self.state)
  
          # 拼装prompt调用chatgpt
          prompt_for_chatgpt = self._wrap(user_input, records)
          print("===gpt-prompt===")
          print(prompt_for_chatgpt)
  
          # 调用chatgpt获得回复
          response = self._call_chatgpt(prompt_for_chatgpt)
  
          # 将当前用户输入和系统回复维护入chatgpt的session
          self.session.append({"role": "user", "content": user_input})
          self.session.append({"role": "assistant", "content": response})
          return response
  ```

  ```python
  #将垂直知识加入prompt，以使其准确回答
  prompt_templates = {
      "recommand": "用户说：__INPUT__ \n\n向用户介绍如下产品：__NAME__，月费__PRICE__元，每月流量__DATA__G。",
      "not_found": "用户说：__INPUT__ \n\n没有找到满足__PRICE__元价位__DATA__G流量的产品，询问用户是否有其他选择倾向。"
  }
  
  dm = DialogManager(prompt_templates)
  ```

  ```python
  #response = dm.run("300太贵了，200元以内有吗")
  response = dm.run("流量大的")
  print("===response===")
  print(response)
  ```

  ```
  ===semantics===
  {'sort': {'ordering': 'descend', 'value': 'data'}}
  ===state===
  {'sort': {'ordering': 'descend', 'value': 'data'}}
  ===gpt-prompt===
  用户说：流量大的 
  
  向用户介绍如下产品：无限套餐，月费300元，每月流量1000G。
  ===response===
  您好，对于流量需求大的用户，我们推荐我们的无限套餐。这个套餐每月只需300元，就能享受1000G的流量。这样您就可以尽情畅游互联网，不用担心流量不够用的问题了。您觉得这个套餐符合您的需求吗？
  ```

  ```python
  response = dm.run("300太贵了，200元以内有吗")
  #response = dm.run("流量大的")
  print("===response===")
  print(response)
  ```

  ```
  ===semantics===
  {'price': {'operator': '<=', 'value': 200}}
  ===state===
  {'sort': {'ordering': 'descend', 'value': 'data'}, 'price': {'operator': '<=', 'value': 200}}
  ===gpt-prompt===
  用户说：300太贵了，200元以内有吗 
  
  向用户介绍如下产品：畅游套餐，月费180元，每月流量100G。
  ===response===
  您好，如果您觉得300元的无限套餐有点贵的话，我们还有一个更经济实惠的选择——畅游套餐。这个套餐每月只需180元，就能享受100G的流量。虽然流量量相对少一些，但对于一般的日常使用已经足够了。您觉得这个畅游套餐符合您的需求吗？
  ```

  <div class="alert alert-success">
  <b>划重点：</b>
  <ul>
  <li>Temperature 参数很关键，生成结果的多样性</li>
  <li>执行任务用 0，文本生成用 0.7-0.9</li>
  <li>无特殊需要，不建议超过1</li>
  </ul>
  </div>

  





