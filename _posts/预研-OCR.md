参数调整
[https://www.npmjs.com/package/@gutenye/ocr-node](https://www.npmjs.com/package/@gutenye/ocr-node)

## 替换检测模型成最新的

#### **第一步：安装环境**

bash

```bash
pip install paddleocr paddle2onnx onnx
pip install paddlepaddle
```

#### **第二步：下载PP-OCRv5检测模型**

**Mobile版本（轻量）：** https://paddle-model-ecology.bj.bcebos.com/paddlex/official_inference_model/paddle3.0.0/PP-OCRv5_mobile_det_infer.tar

powershell命令行下载

```powershell
# 下载压缩包（如果有的话）
Invoke-WebRequest -Uri "https://paddle-model-ecology.bj.bcebos.com/paddlex/official_inference_model/paddle3.0.0/PP-OCRv5_mobile_det_infer.tar" -OutFile "PP-OCRv5_mobile_det_infer.tar"

# 解压
tar -xf PP-OCRv5_mobile_det_infer.tar

# 检查解压后的文件
dir PP-OCRv5_mobile_det_infer

```

#### 第三步，转换

```powershell
paddle2onnx --model_dir PP-OCRv5_mobile_det_infer `
            --model_filename inference.pdmodel `
            --params_filename inference.pdiparams `
            --save_file PP-OCRv5_mobile_det_infer `
            --opset_version 11 `
            --enable_onnx_checker True
```

老是转换不成功
找到一个开源项目
https://github.com/jingsongliujing/OnnxOCR/tree/main/onnxocr

## 遇到的困难

### 1.当前使用的框架node端不支持参数

API地址[https://www.npmjs.com/package/@gutenye/ocr-node](https://www.npmjs.com/package/@gutenye/ocr-node)

![image](https://wiki.huawei.com/vision-file-storage/api/file/download/upload-v2/WIKI202508137860869/27324266/ab57f096a1834a0f82e44e4c3d9e963b.png)

```
detectionThreshold (默认: 0.3)
作用：控制文本像素分类的置信度阈值
降低可检测更多文本，但可能增加噪声

detectionBoxThreshold (默认: 0.6)
作用：过滤低置信度的文本检测框

detectionUnclipRatiop (默认: 1.5)
作用：文本检测框的膨胀系数，控制框的扩张

```

#### 1.1 当前为什么node不支持

##### OCR的完整流程

```
图片 → 预处理 → Detection ONNX模型 → 概率图 → 后处理 → 文本框坐标 → Recognition模型 → 文字内容
```

[python开源实现](https://github.com/jingsongliujing/OnnxOCR)有自己的后处理代码，开源重写的
![image](https://wiki.huawei.com/vision-file-storage/api/file/download/upload-v2/WIKI202508137860869/27342590/2b894cb2070047acba34616b7e5c80f5.png)

##### Node.js版本架构：

* ​**依赖**​：使用原生OpenCV绑定 (如opencv4nodejs)
* ​**后处理**​：依赖OpenCV的C++实现进行图像处理，后处理逻辑封装在OpenCV内部
* ​**参数控制**​：OpenCV内部处理大部分后处理逻辑，参数固化在代码中，不对外暴露

证据：[https://github.com/gutenye/ocr/blob/main/.vscode/c_cpp_properties.json](https://github.com/gutenye/ocr/blob/main/.vscode/c_cpp_properties.json/)

![image](https://wiki.huawei.com/vision-file-storage/api/file/download/upload-v2/WIKI202508137860869/27343228/e36efef883b143f1b9c318a19b2cab22.png)
