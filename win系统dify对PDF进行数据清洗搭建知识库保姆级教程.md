# win系统dify对PDF进行数据清洗搭建知识库保姆级教程

### 环境准备：

1.python环境要求：3.12，建议采用conda创建一个新环境，没有conda要保证python版本在3.12，下列步骤正常进行。

2.下载pytorch库，网址链接：https://pytorch.org/。先在conda prompt打开conda创建好的环境，再输入pip指令（在官网上选择对应版本，如果有GPU选择CUDA，没有选cpu）这里最好开VPN加速。

### PDF转换markdown：

3.下载好后继续输入：pip install marker-pdf，下载将pdf装换为markdown文件的工具，这里可以把vpn关掉。

4.接下来在你要解析的pdf路径里打开conda prompt，进入到一开始创建的python版本等于3.12的环境，输入   **marker_single 科技园车床智能问答.pdf（此处为你要转换PDF文件的名称） --output_dir ./**。此时你就会得到一个未清洗的markdown文件和拆分出来的图片，后续只会用到markdown文件。

5接下来打开docker（**请确保dify是基于docker本地部署，网页版无法将清洗好的文件下载到本地**）选择sandbox-1点击exec写入权限下载清洗好的文件，如图操作（有**#**是自己打的，没有是系统提示）

![dify权限](C:/Users/fg/Desktop/image/dify%E6%9D%83%E9%99%90.png)

此时点击sandbox-1,点击Files（exec右边），找到var--sandbox--sandbox-py--data--file，这就是我们清洗好之后markdown文件的位置。

### dify工作流搭建

- 选择**chatflow** 创建应用，不要选择工作流，在页面右上方{x}会话变量添加一个file_catalog变量

- **开始** 输入字段选入单文件--文档，勾选必填，输入变量命名为file_name。

- **文档提取器** 输入变量选择为file_name

- **代码执行** 将arg2删去保留arg1，后边变量选取文档提取，保留前20000个字段提取出文章大纲。代码实现如下：

  ```
  def main(arg1: str) -> dict:
  
    return {
  
  ​    "result": arg1[:20000],
  
    }
  ```

  

- **LLM** 选chat模型，在使用dify之前配置好模型商，我选取的是qwen。上下文变量选取代码执行。在system中写入提示词：

  ```
  你是一名专业的文档分析师，擅长从技术文档/论文中提取结构化大纲。
  
  你的任务是：
  - 识别标题层级(H1-H6)
  - 统计章节分布与内容概要
  - 输出带缩进层级的摘要树
  
  任务执行步骤：
  1. 通读文档，标记所有标题（格式如#、##等）；
  2. 按缩进层级构建大纲树（H为一级，H2为二级，以此类推）
  3. 为每个章节生成1-2句内容摘要；
  4. 输出Markdown格式的层级列表，包含标题和摘要。
  例如
  输出格式：
  markdown
  - [标题层级] 标题文本
     摘要： 本节核心内容（100字以内）
  
  示例输入：
  # 引言
  ## 研究背景
  ### 行业现状
  ## 研究方法
  
  示例输出：
  - [H1]引言
     摘要：阐述研究背景与方法论
     -[H2] 研究背景
       摘要：分析行业现状与问题
       -[H3]行业现状
        摘要：描述当前市场数据
     -[H2] 研究方法
       摘要：说明实验设计与技术路线
  
  注：基于原文来设计，不允许补充内容，不允许扩写内容
  ```

  添加消息，在USER里写到：“需要处理的文档为（点击右上方{x}选择**上下文**），即可**启用上下文功能**。

  

- **变量赋值** 将catalog存入LLM导出的text，作为全局中间变量

- **代码执行** 保留arg1，变量选取LLM—text

  ```
  
  import os
  import re
  
  def main(arg1: str) -> dict:
      try:
          # 修改为你想保存的路径，例如 D 盘根目录或某个具体文件夹
          directory = 'D:\\'  # 或者使用 D:/ 也可以
          file_path = os.path.join(directory, 'summary.md')
          
          # 确保目录存在（如果有必要创建目录）
          os.makedirs(directory, exist_ok=True)
          
          # 写入文件，同时替换非法文件名字符
          with open(file_path, 'w', encoding='utf-8') as f:
              f.write(re.sub(r'[\/\\|:、]', '', arg1))  # 使用 \/ 兼容正则表达式
          
          return {
              "result": '文件生成成功',
          }
      except Exception as e:
          return {"result": str(e)}
  ```

  

- **直接回复** ：成功生成大纲

  ------

  #### 此时在变量赋值模块再添加一个代码执行，与代码执行2并列

  ```
  
  def main(arg1: str,segment_length=3000,overlap=500) -> dict:
      total_length = len(arg1)
      segments = []
      start = 0
      while start < total_length:
          end = start + segment_length
          if end > total_length:
              end = total_length
          segment = arg1[start:end]
          segments.append(segment)
          start +=segment_length - overlap
          if start >= total_length:
              break
      return {
          "result": segments
      }
  ```

保留arg1，对应变量选择文档提取器text，输出变量类型选择Array[String]

- **迭代** 输入变量选择代码执行3的result

  在迭代里增加LLM节点，上下文选择迭代item，system写入，迭代输出变量选择LLM2—text（在写完迭代里节点后才能选择）

  ```
  你是一个专业数据清洗及标注工程师；
  请先阅读并理解本文档的全部大纲，
  <文本大纲>为：file_catalog
  再按照以下规则处理文本，
  <文本内容>为： 上下文
  一、数据清洗：
  1. 提取结构化正文
  2. 过滤噪声内容，移除页眉/页脚、广告、版权声明、乱码、空白行、无效数据行、重复段落、html标签(比如<br>、段落序号数字(1,2,3...)
  3. 自动合并碎片化段落（<80字符且无标点）
  4. 识别并标记专业术语
  5. 如果文本内容为没有任何含义的符号和html标签，这段内容直接清除不需要输出
  
  二、数据标注：
  1. 动态分段机制
    - 基础段长： 200~250字节，允许学术类扩展至450字节
    - 对话类强制分段规则：每3论对话必须拆分段落
    - 保留逻辑标记符：如"首先/其次/最后”等序列次需要在通一段落
  
  2.  标签tags构成
  
  标题： 标题1/标题2/标题3/标题4/标题5/等（从大纲和本段文字前1000字内搜索，获取分段标题）
  
  关键词： 关键词1/关键词2/关键词3/关键词4/关键词5/关键词6/关键词7/关键词8/关键词9/关键词10/专业术语等（从文段获取10个以上关键词与元数据，同时要加上识别到的专业术语）
  
  3. 特殊处理
  冲突解决机制
     - 当相邻段落标签重复率>60%，触发智能合并
     - 语义断裂检测：使用<gap>标记逻辑跳跃点
  
  4. 示例：
  输入段落：
  "除非谷歌和 OpenAI 改变态度,选择和开源社区合作,否则将被后者替代",据彭博 和 SemiAnalysis 报道,4 月初,谷歌工程师 Luke Sernau 发文称,在人工智能大语言模 型(Large Language Models,LLM,以下简称"大模型")赛道,谷歌和 ChatGPT 的推 出方 OpenAI 都没有护城河,开源社区正在赢得竞赛。
  这一论调让公众对"年初 Meta 开源大模型 LLaMA 后,大模型大量出现"现象的关注推 向了高潮,资本市场也在关注大公司闭源超大模型和开源大模型谁能赢得竞争,在"模 型""算力""数据"三大关键要素中,大模型未来竞争格局如何,模型小了是否就不再 需要大量算力,数据在其中又扮演了什么角色?……本报告试图剖析这波开源大模型风 潮的共同点,回顾开源标杆 Linux 的发展史,回答以上问题,展望大模型的未来。
  
  期望输出下面这样格式的内容：
  {
     "segment_01":{
         "text": “谷歌工程师Luke Sernau认为，谷歌和OpenAI在AI大模型领域缺乏护城河，开源社区（如Meta的LLaMA）正逐渐占据优势。这一观点引发了对闭源与开源模型竞争格局的讨论，涉及模型、算力、数据三大要素的未来影响。报告通过分析开源大模型风潮的共同点，并类比Linux的发展历史，探讨了开源模式是否将主导AI领域，以及小模型、算力需求和数据角色的演变趋势。”，
          “tags”: [“开源大模型 ”,"AI竞争","MetaLLaMA,"闭源VS开源"]
      }
  }
  
  注意：
  1. text是原文部分，不允许修改和扩展，直接使用格式清洗之后的原文内容
  2. 标签tags里面要列出所有的标题，识别内容后，标题参考<文本大纲>， 生成时，不需要带上层级序号
  3. 如果是有条文类的文档，比如规则、法律类等文档，tags里面需要标明是第几章，第几条
  4. tags除了上述外，还要再从文段获取10个左右关键词
  5. tags内容放在一行，关键词用英文双引号""包围，用应为逗号分割即可
  ```

  在LLM后增加节点代码执行输入变量为：1.arg1—LLM 2text；2.index—迭代 **index**Number

```
import os
import re

def main(arg1: str, index) -> dict:
    try:
        # 设置保存到 D 盘根目录，也可以指定具体文件夹如：'D:\\output'
        directory = 'D:\\'
        file_path = os.path.join(directory, f'part{index}.md')

        # 确保目标目录存在
        os.makedirs(directory, exist_ok=True)

        # 写入文件，去除非法字符
        with open(file_path, 'w', encoding='utf-8') as f:
            cleaned_content = re.sub(r'[\/\\|:、]', '', arg1)
            f.write(cleaned_content)

        return {
            "result": '文件生成成功',
            "file_saved_to": file_path  # 可选：返回保存路径
        }

    except Exception as e:
        return {"result": str(e)}
```

- **代码执行** 此时**跳出迭代**新增节点，将两个输入变量都删除，代码执行为

  ```
  import os
  import re
  import json
  from pathlib import Path
  
  def main() -> dict:
      """处理文件夹中的所有part*.md文件，合并内容并添加标记"""
      try:
          folder = Path('/data/file')
          output_file = folder / 'context_for_all.txt'
          
          # 获取并按数字排序文件
          files = sorted(
              [f for f in folder.glob('part*.md') if f.is_file()],
              key=lambda x: int(re.search(r'\d+', x.name).group())
          )
          
          with output_file.open('w', encoding='utf-8') as f_out:
              for file in files:
                  with file.open(encoding='utf-8') as f_in:
                      content = f_in.read()
                      
                      # 使用正则表达式提取所有segment
                      segments = re.finditer(r'{\s*"segment_\d+"\s*{([^}]+)}', content)
                      
                      for seg in segments:
                          segment_content = seg.group(1)
                          
                          # 提取text部分
                          text_match = re.search(r'"text"\s*"([^"]+)"', segment_content)
                          if text_match:
                              f_out.write(text_match.group(1) + "###\n")
                          
                          # 提取tags部分并处理
                          tags_match = re.search(r'"tags"\s*"([^"]+)"', segment_content)
                          if tags_match:
                              tags = [t.strip() for t in tags_match.group(1).split(',') if t.strip()]
                              f_out.write('###'.join(tags) + "###\n")  # 用###分割每个tag
                          
                          f_out.write("&&&&\n")  # 分隔不同的segment
          
          return {"result": "文件处理完成"}
  
      except Exception as e:
          return {"result": f"处理失败: {str(e)}"}
  ```

  

- **直接回复** 文件处理成功

  ------

  #### 项目总结

  此chatflow亮点在于先取20000字段总结大纲并分级处理，再分割文本与大纲一一对应，符合知识库的导入习惯，可用于处理多领域PDF。
