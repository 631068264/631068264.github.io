---
layout:     post
rewards: false
title:   kubeflow predict server
categories:
    - k8s


---

用户可以上传已经训练好的模型，平台提供开箱即用的http server服务，AI应用可以通过rest api调用模型服务。

支持的框架

- TensorFlow
- Pytorch

下面以不同框架为例子，介绍训练，打包的过程，[更多列子可以参考这个。](https://github.com/tduffy000/kfserving-uri-examples)

模型运行在server上，server部署到集群上。server会load model，提供http服务，接受post请求，经过model的处理，返回对应输出。

# Tensorflow

**模型提供者需要根据以下介绍，对自己的模型作出对应的修改。**

镜像：[tensorflow/serving:1.14](https://github.com/tensorflow/serving)

TensorFlow ModelServer: 1.14.0-rc0

TensorFlow Library: 1.14.0

**[tensorflow-serving 输入输出格式，参考官方文档](https://www.tensorflow.org/tfx/serving/api_rest#predict_api)**

例子用到input，**行向量**输入一组数据4个参数，对应output一组三个的预测几率结果

```json
{
  "instances": [
    [6.8,  2.8,  4.8,  1.4],
    [6.0,  3.4,  4.5,  1.6]
  ]
}
```

output

```json
{"predictions":[[0.00760850124,0.628415406,0.363976091],[0.00809291936,0.643412352,0.348494798]]}
```

模型打包，**iris.tar.gz**就是可以上传的模型

```
cd forzen
tar -czvf iris.tar.gz 0001 
```

**打包注意事项**

- 打包格式为**tar.gz**
- 压缩包**不能用0001.tar.gz这种命名方式**，最好用英文命名，不然[报错](https://stackoverflow.com/questions/45544928/tensorflow-serving-no-versions-of-servable-model-found-under-base-path)



# Pytorch

镜像： [kfserving/pytorchserver:v0.6.1](https://github.com/kserve/kserve/blob/master/python/pytorch.Dockerfile)

**pytorch 版本1.7.1**

由于这不是AI框架提供的server，是kserve写的server，提供的功能也比较简单。

[参照源码](https://github.com/kserve/kserve/tree/master/python/pytorchserver)里面的predict方法

```python
// 处理请求
inputs = torch.tensor(request["instances"]).to(self.device)

// 返回结果
return {"predictions":  self.model(inputs).tolist()}
```

**得到输入输出格式**，模型**forward**函数里面做input的预处理

```json
// 输入  只支持行向量格式
{
  "instances": tensor_data
}

// 输出
{"predictions":模型输出结果数组}
```



例子用到input，**行向量**输入一组数据4个参数，对应output一组三个的预测几率结果

```json
{
  "instances": [
    [6.8,  2.8,  4.8,  1.4],
    [6.0,  3.4,  4.5,  1.6]
  ]
}
```

output

```json
{"predictions":[[0.00760850124,0.628415406,0.363976091],[0.00809291936,0.643412352,0.348494798]]}
```



打包格式为**tgz**，**直接压缩两个文件，不然会模型运行会报错**

```sh
cp model.py forzen/model.py
cd forzen
tar czvf iris.tgz *
```



# 自定义Model Server

如果开箱即用的model server不能满足要求，可以使用 KServe ModelServer API 构建自己的模型服务器。

**用户可以完全掌握model server的输入输出和数据处理**

## 继承扩展kserve.Model

可以继承`KServe.Model`自定义里面的三个方法 `preprocess`, `predict` and `postprocess`, **按序执行**。

- preprocess：预处理input
- predict: 利用input执行推理，输出ouput
- postprocess：美化prdict的output

具体原理如下

```python
    async def __call__(self, body, model_type: ModelType = ModelType.PREDICTOR):
        request = await self.preprocess(body) if inspect.iscoroutinefunction(self.preprocess) \
            else self.preprocess(body)
        request = self.validate(request)
        if model_type == ModelType.EXPLAINER:
            response = (await self.explain(request)) if inspect.iscoroutinefunction(self.explain) \
                else self.explain(request)
        elif model_type == ModelType.PREDICTOR:
            response = (await self.predict(request)) if inspect.iscoroutinefunction(self.predict) \
                else self.predict(request)
        else:
            raise NotImplementedError
        response = self.postprocess(response)
        return response
```

还有一个额外的`load`方法用于编写自定义代码以将模型从本地文件系统或远程模型存储加载到内存中，一般的好做法是在model server 的 `__init__`中调用，以便加载模型当用户进行预测调用时启动并准备好服务。

代码实例model.py  [完整示例](https://github.com/kserve/kserve/blob/master/python/custom_model/model.py)

```python
import kserve
from torchvision import models, transforms
from typing import Dict
import torch
from PIL import Image
import base64
import io

class AlexNetModel(kserve.Model):
    def __init__(self, name: str):
        super().__init__(name)
        self.name = name
        self.load()

    def load(self):
        model = models.alexnet(pretrained=True)
        model.eval()
        self.model = model
        self.ready = True

    def predict(self, request: Dict) -> Dict:
        inputs = request["instances"]

        # Input follows the Tensorflow V1 HTTP API for binary values
        # https://www.tensorflow.org/tfx/serving/api_rest#encoding_binary_values
        data = inputs[0]["image"]["b64"]

        raw_img_data = base64.b64decode(data)
        input_image = Image.open(io.BytesIO(raw_img_data))

        preprocess = transforms.Compose([
            transforms.Resize(256),
            transforms.CenterCrop(224),
            transforms.ToTensor(),
            transforms.Normalize(mean=[0.485, 0.456, 0.406],
                                 std=[0.229, 0.224, 0.225]),
        ])

        input_tensor = preprocess(input_image)
        input_batch = input_tensor.unsqueeze(0)

        output = self.model(input_batch)

        torch.nn.functional.softmax(output, dim=1)[0]

        values, top_5 = torch.topk(output, 5)

        return {"predictions": values.tolist()}


if __name__ == "__main__":
    model = AlexNetModel("custom-model")
    model.load()
    kserve.ModelServer(workers=1).start([model])

```

## 镜像构建

文件结构

```
.
├── Dockerfile
├── model.py
├── requirements.txt

```

Dockerfile

```dockerfile
# 使用稳定python瘦身版,减少镜像体积
FROM python:3.9.10-slim-buster
# 禁用pip缓存,减少镜像体积
ENV PIP_NO_CACHE_DIR=1

# 安装依赖，没用到第三方包别往requirements.txt写
COPY requirements.txt ./
# 指定依赖镜像地址，加快下载

RUN pip install -i https://mirrors.aliyun.com/pypi/simple/ -r requirements.txt

# 复制源码
COPY model.py ./

ENTRYPOINT ["python","-m","model"]
```

ModelServer参数

- --workers ：model server workers(multi-processing) 数目，默认为1，[获取使用并行推理](https://kserve.github.io/website/0.8/modelserving/v1beta1/custom/custom_model/#parallel-inference)
- --max_buffer_size：model server接受的最大数据量 默认10M

[详细参数参考这里](https://github.com/kserve/kserve/blob/master/python/kserve/kserve/model_server.py)，关于端口最后别改。

requirements.txt

```
kserve
torchvision
```

镜像构建

```
docker build -t custom-model:v1 .
```



input

```json
{
  "instances": [
    {
      "image": { "b64": "xxxx"}
    }
   ]
}
```



测试镜像

```sh
docker run -ePORT=8080 -p8080:8080 custom-model:v1

curl localhost:8080/v1/models/custom-model:predict -d @./input.json

{"predictions": [[14.861763000488281, 13.94291877746582, 13.924378395080566, 12.182709693908691, 12.00634765625]]}

```

