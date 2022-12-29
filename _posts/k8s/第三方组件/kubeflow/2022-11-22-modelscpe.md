---
layout:     post
rewards: false
title:   kubeflow 和 modelscope现成model结合
categories:
    - k8s

---



modelscope有其专有的模型仓库和数据集仓库，通过git管理版本，使用时指定namespace/model-name就可以缓存到本地，方便复用。模型主要基于TensorFlow，Pytorch

与kubeflow在离线环境结合需要下载模型，加载本地模型，http服务，测试部署。

下面以https://www.modelscope.cn/models/damo/cv_vit-base_image-classification_Dailylife-labels/summary 为例子

# 目录结构

```
.
├── Dockerfile
├── bird.jpeg
├── interface.yaml
├── model.py
├── model
│   ├── README.md
│   ├── checkpoints.pth
│   ├── config.py
│   ├── configuration.json
│   └── resources

```



# 下载模型

[文档链接](https://www.modelscope.cn/docs/%E6%A8%A1%E5%9E%8B%E7%9A%84%E4%B8%8B%E8%BD%BD)

```sh
# 例如: git clone https://www.modelscope.cn/damo/cv_vit-base_image-classification_Dailylife-labels.git
git clone https://www.modelscope.cn/<namespace>/<model-name>.git

```

下载模型到model目录

# 加载本地模型

加载example

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from modelscope.pipelines import pipeline
from modelscope.utils.constant import Tasks

# img_path 文件目录或图片URL
img_path = 'bird.jpeg'
# model参数传入模型目录
image_classification = pipeline(Tasks.image_classification, model='model')
result = image_classification(img_path)
print(result)

```

# http服务

我们面临两个选择

- 使用自定义镜像，运行在kserve的HTTP server
- 使用kubeflow开箱即用的http server服务

我们需要泛用性高，工作量少的方法

选择方案一，方案二有其局限性

- 开箱即用的http server服务有框架版本限制，**自定义镜像则和框架甚至py版本版本无关**

- 我们只得到**模型文件没有模型源码**，使用http server服务可能得修改模型的输入输出，自定义镜像更加灵活适应场景

  就像模型返回的output 有numpy object 和 中文，不经过特殊处理，又没有模型源码根本解决不了response

  ```json
  {'scores': [0.45598775, 0.25016323, 0.035127528, 0.021287015, 0.019946197], 'labels': ['鸟', '小鸟', '羽毛', '蜂鸟', '鸽']}
  
  ```

  关于模型只接受路径、URL，离线环境下不做处理，根本用不了

model.py

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import base64
import json
import os
import tempfile
from typing import Dict, Any

import kserve
import numpy
from modelscope.pipelines import pipeline
from modelscope.utils.constant import Tasks

# 处理numpy object json序列化
class CustomJsonEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, numpy.integer):
            return int(obj)
        elif isinstance(obj, numpy.floating):
            return float(obj)
        elif isinstance(obj, numpy.ndarray):
            return obj.tolist()
        return super(CustomJsonEncoder, self).default(obj)

# 处理可能出现的序列化问题
def safe_json(output: Any) -> str:
    return json.dumps(
        output,
        ensure_ascii=False,
        allow_nan=False,
        indent=None,
        separators=(",", ":"),
        cls=CustomJsonEncoder,
    ).replace("</", "<\\/")


class AIModel(kserve.Model):
    def __init__(self, name: str):
        super().__init__(name)
        self.name = name
        self.load()

    def load(self):
        model = pipeline(Tasks.image_classification, model='model')
        self.model = model
        self.ready = True

    def predict(self, request: Dict) -> Dict:
        inputs = request["instances"]
        data = inputs[0]["image"]["b64"]
				# 模型只接受路径，使用临时文件处理
        img = base64.b64decode(data)
        with tempfile.NamedTemporaryFile(mode='wb', delete=False) as temp:
            temp.write(img)

        output = self.model(temp.name)
        os.remove(temp.name)

        return {"predictions": safe_json(output)}


if __name__ == "__main__":
    # 必须预留从环境变量中接收模型名字，MODEL_NAME这个key不能改，value影响到模型部署的URL: /v1/models/{MODEL_NAME}:predict
    model = AIModel(os.environ.get('MODEL_NAME', 'custom-model'))
    model.load()
    kserve.ModelServer(workers=1).start([model])

```

# 测试部署

requirements.txt

```
modelscope==1.0.3
kserve
mmcls
mmcv
torchvision
transformers
fairseq
timm
unicodedata2
zhconv
```

modelscope没有在pypi，可以download命令拿到whl

```sh
pip download "modelscope[cv]" -f https://modelscope.oss-cn-beijing.aliyuncs.com/releases/repo.html --only-binary=:all: --platform linux_x86_64 --n
o-deps --python-version 39 -d package
```

Dockfile

```dockerfile
# 使用稳定python瘦身版,减少镜像体积
FROM python:3.9.15-slim-buster
# 禁用pip缓存,减少镜像体积
ENV PIP_NO_CACHE_DIR=1

# 编译报错补依赖
RUN sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list && \
	apt update && \
	apt-get install -y --no-install-recommends gcc ffmpeg libsm6 libxext6 g++ && \
	rm -rf /var/lib/apt/lists/*

# 安装依赖，没用到第三方包别往requirements.txt写
COPY *.whl ./
# 指定依赖镜像地址，加快下载
RUN python -m pip install --upgrade pip \
    && pip install -i https://pypi.tuna.tsinghua.edu.cn/simple *.whl \
    && rm -rf *.whl

COPY requirements.txt ./
RUN pip install -i https://pypi.tuna.tsinghua.edu.cn/simple -r requirements.txt


# 复制源码
COPY model ./model
COPY model.py ./

ENTRYPOINT ["python","-m","model"]
```



docker 测试

```sh
docker build -t custom-model:v1 .
docker run -ePORT=8080 -p8080:8080 custom-model:v1

curl localhost:8080/v1/models/custom-model:predict -d @./input.json

input
{
  "instances": [
    {
      "image": { "b64": "xxxxx"}
    }
   ]
}

output
{
    "predictions": "{\"scores\":[0.4559873044490814,0.2501634657382965,0.035127561539411545,0.021287014707922935,0.01994621567428112],\"labels\":[\"鸟\",\"小鸟\",\"羽毛\",\"蜂鸟\",\"鸽\"]}"
}
```

interface.yaml

```yaml
apiVersion: "serving.kubeflow.org/v1beta1"
kind: "InferenceService"
metadata:
  name: "imageclassification"
spec:
  predictor:
    containers:
      - name: kserve-container
        image: xxx/model/imageclassification:v1
        env:
          - name: MODEL_NAME
            value: imageclassification

```

部署到kubeflow

```yaml
kubectl apply -f interface.yaml
```

测试

```sh
curl -v -H "Host: ${SERVICE_HOSTNAME}" http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/${MODEL_NAME}:predict -d @./input.json




kubectl get InferenceService -A
NAMESPACE   NAME                  URL                                              READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION                           AGE
default     imageclassification   http://imageclassification.default.example.com   True           100                              imageclassification-predictor-default-00001   3h1m

SERVICE_HOSTNAME=imageclassification.default.example.com
INGRESS_HOST=kubeflow IP
INGRESS_PORT=kubeflow 端口
MODEL_NAME=imageclassification


```



