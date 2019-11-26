自然语言的内容有很多，本文所介绍的自然语言处理部分是「文本摘要」和「关键词提取」。

很多朋友会有自己的博客，在博客上发文章时，这些文章发出去后，有的很容易被搜索引擎检索，有的则很难。那么有没有什么方法，让搜索引擎对博客友好一些呢？这里有一个好方法 —— 那就是填写网页的 Description 还有 Keywords。

但是每次都需要我们自己去填写，非常繁琐。这个过程能否自动化实现？本文将会通过 Python 的 jieba 和 snownlp 进行文本摘要和关键词提取的实现。

### ▎准备资源

下载以下资源：

- [Python 中文分词组件](https://github.com/fxsjy/jieba)
- [Simplified Chinese Text Processing](https://github.com/isnowfy/snownlp)

下载完成后，新建文件夹，拷贝对应的文件：

![](https://main.qcloudimg.com/raw/6e6db37cad89defb573870bb1d57d5e1.jpg)

拷贝之后，建立文件 index.py

```
# -*- coding: utf8 -*-
import json
import jieba.analyse
from snownlp import SnowNLP


def FromSnowNlp(text, summary_num):
    s = SnowNLP(text)
    return s.summary(summary_num)


def FromJieba(text, keywords_type, keywords_num):
    if keywords_type == "tfidf":
        return jieba.analyse.extract_tags(text, topK=keywords_num)
    elif keywords_type == "textrank":
        return jieba.analyse.textrank(text, topK=keywords_num)
    else:
        return None


def main_handler(event, context):
    text = event["text"]
    summary_num = event["summary_num"]
    keywords_num = event["keywords_num"]
    keywords_type = event["keywords_type"]

    return {"keywords": FromJieba(text, keywords_type, keywords_num),
            "summary": FromSnowNlp(text, summary_num)}
```

超简单的代码有没有！

### ▎上传文件

在云函数 SCF 控制台上新建一个项目：

![](https://main.qcloudimg.com/raw/b1d8b165daff4bb8a8bc74d44f0307e3.jpg)

![](https://main.qcloudimg.com/raw/d17bd0196f2e9d8c10db02485a343c17.jpg)

提交方法选择上传 zip：

然后我们压缩文件，并改名为 index.zip：

![](https://main.qcloudimg.com/raw/9ad879bbaa6d10c75c5c90da4665df35.jpg)

### ▎测试

测试之前可以适当调整一下我们的配置：

![](https://main.qcloudimg.com/raw/15bca21dafbef68fc4efc1210993ea1e.jpg)

然后进行 input 模板的输入：

![](https://main.qcloudimg.com/raw/c1fa1ce2d15bdde2e672722ad78e9c6b.jpg)

模板可以是：

```
{
  "text": "前来参观的人群络绎不绝。在“两弹历程馆”里……（略）”",
  "summary_num": 5,
  "keywords_num": 5,
  "keywords_type": "tfidf"
}
```

然后点击测试：

![](https://main.qcloudimg.com/raw/ba3539b1d4dc761cbeb54f019e57ad94.jpg)

### ▎应用

至此，我们完成了简单的关键词提取功能和抽取式文本摘要过程。

当然，这只是简单的抛砖引玉，因为摘要这里还有声称是文本摘要，而且抽取式摘要也可能会根据不同的文章类型，有着不同的特色方法，所以这里只是通过一个简单的 Demo 来实现一个小功能，帮助大家做一个简单的 SEO 优化。

大家以后自己做博客的时候，可以增加 keywords 或者 description 字段，然后每次从 sql 获得文章数据的时候，将这两个部分放到 meta 中，会大大提高页面被索引的概率哦～！