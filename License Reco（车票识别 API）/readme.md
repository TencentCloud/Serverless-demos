## “灰常”简单的车牌识别 API 制作

> 本文的真正目的，其实并非要做一个完善的车牌识别工具，而是想要通过一些简单的 package 组合（包括深度学习框架等），实现一个简单的对外接口，用它来进行车牌识别。

**这个项目的小难点在于 —— 如何打包依赖（包含需要 .so 的依赖）。**

包含 .so 的依赖，通常是某些依赖需要编译一些文件（非纯 Python 实现的），此时，「稍有不慎」就会让我们无法执行代码。所以这个时候可以使自己的打包环境与云函数一致：CentOS + Python 3.6。

## ▎**本地测试**

编写代码：

![img](https://pic1.zhimg.com/80/v2-3f89fe1f90ac1f14408fedd19b949290_hd.jpg)

执行结果：

![img](https://pic4.zhimg.com/80/v2-fd782b825a8364c4edb1cf3058cc0b37_hd.png)

## ▎**打包上传**

CentOS + Python 3.6 的基本环境下：

建立文件夹并进入：

```text
mkdir mytest && cd mytest
```

安装依赖：

安装 opencv-python

```text
sudo pip install opencv-python -t /home/dfounderliu/code/mytest
```

安装 hyperlpr（这是一个基于 DNN 的深度学习模块。该模块的使用，也充分说明了，云函数 SCF 可以执行深度学习的项目模型，完美......）

```text
sudo pip install hyperlpr -t /home/dfounderliu/code/mytest
```

建立测试：

vim index.py

编写内容：

```text
from hyperlpr import *
import cv2
```

保存，并且打包，上传至云函数 SCF：

```text
zip -r index.zip .
```

云函数测试：

![img](https://pic3.zhimg.com/80/v2-31584131e79867c2850ad8575347202e_hd.jpg)

表面上看起来似乎失败了，但实际上，它是成功的。因为失败的是我们的方法没有建立，而我们的 import 已经正确导入了（就是说没有在添加依赖部分报错！）

## ▎**编写函数**

```text
# 导入包
from hyperlpr import *
import cv2
import base64
import json
import urllib.parse

def save_picture(base64data):
    try:
        imgdata = base64.b64decode(urllib.parse.unquote(base64data))
        file = open('/tmp/picture.png', 'wb')
        file.write(imgdata)
        file.close()
        return True
    except Exception as e:
        return str(e)


def ana_picture():
    print(cv2.imread("/tmp/picture.png"))
    return {"resulr": HyperLPR_PlateRecogntion(cv2.imread("/tmp/picture.png"))}


def main_handler(event, context):
    save_result = save_picture(event["body"].replace("image=",""))
    if  save_result == True:
        return ana_picture()
    else:
        return save_result
   
    # return save_picture
    
```

测试结果：

![img](https://pic4.zhimg.com/80/v2-e53b64b9727c459005cf27762bc029eb_hd.jpg)

测试图像转 base64 代码：

```text
#image转base64
import base64
with open("2.png","rb") as f:#转为二进制格式
    base64_data = base64.b64encode(f.read())#使用base64进行加密
    print(base64_data)
    file=open('1.txt','wt')#写成文本格式
    file.write(base64_data)
    file.close()
```

测试时 API 网关参数：

![img](https://pic2.zhimg.com/80/v2-f7f4585a72fc8a48a0d1d62cd44ff029_hd.jpg)

## ▎**对接 API 网关**

![img](https://pic3.zhimg.com/80/v2-c6b2aa61d597d31874e363a3b45f96c6_hd.jpg)

![img](https://pic4.zhimg.com/80/v2-8758907d5eb6b266d5b5acc2a4fc97d7_hd.jpg)

然后发布到测试环境，即可。

## ▎**编写测试**

测试代码：

```text
import base64
import urllib.request
import urllib.parse


with open("1.png","rb") as f:
    base64_data = base64.b64encode(f.read())  # 使用base64进行加密


url = "http://service-l2ksmbje-1256773370.gz.apigw.tencentcs.com/test/picture"
data = {
    "image": base64_data.decode("utf-8")
}

print(urllib.parse.unquote(urllib.request.urlopen(urllib.request.Request(url, data=urllib.parse.urlencode(data).encode("utf-8"))).read().decode("utf-8")))
```

测试结果：

![img](https://pic3.zhimg.com/80/v2-a44884a65e62b590bb9480cbd0e0a6de_hd.jpg)

依赖包下载：[https://myblog-1256773370.cos.ap-guangzhou.myqcloud.com/opencv_numpy_hyperlpr.zip](https://link.zhihu.com/?target=https%3A//myblog-1256773370.cos.ap-guangzhou.myqcloud.com/opencv_numpy_hyperlpr.zip)

## ▎**总结**

本文的主要作用，其实就是通过一些简单的 package 组合，实现对外接口并以此进行车牌识别。一方面，这说明了云函数 SCF 可以做深度学习相关的预测工作，另一方面，也进一步巩固了依赖的打包和与云 API 网关的结合使用。

当然，这个接口如果经过完善后，还可以和 Iot 等进行结合使用。最后，希望各位小伙伴们自行探索 Serverless 的新世界！

> 本文亦发布于[知乎](https://zhuanlan.zhihu.com/p/86194163)