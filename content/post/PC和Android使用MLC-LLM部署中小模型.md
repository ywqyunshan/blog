---
layout:     post

title:      "PC和Android使用MLC-LLM部署中小模型"
subtitle:   ""
excerpt: ""
author:     "iigeoywq button"
date:       2024-12-21
published: true 
tags:
    - 工程实战

categories: [ Tech ]
URL: "/2024/12/21"
---
## 一 中小模型介绍
在https://huggingface.co/网站上先找了基础规模小于等于5B(十亿)参数的中小模型。

| 模型名称 | 所属机构 | 基础规模 | 原生格式 | 量化格式 | 部署格式 | 主要用途 | HuggingFace地址 | 特点说明 
|:--|:--|:--|:--|:--|:--|:--|:--|:--|
| Phi-3-mini | Microsoft | 3.8B | safetensors | GPTQ(4bit/8bit), AWQ(4bit), exllama2 | ONNX, TensorRT, MLC-LLM | 代码生成,通用推理 | https://huggingface.co/collections/microsoft/phi-3-6626e15e9585a200d2d761e3 | 高性能代码能力,低资源消耗 |
| internlm2_5-1_8b-chat | 上海AI实验室 | 1.8B | safetensors | GPTQ(4bit/8bit), AWQ(4bit), W8A8 | ONNX, TensorRT, vLLM, MLC-LLM | 中文对话,知识问答 | https://huggingface.co/internlm/internlm2_5-1_8b-chat | 中文理解深入,上下文长度32K |
| Qwen2.5 | 阿里云 | 3B/1.5B | safetensors | GPTQ(3/4/8bit), AWQ(4bit), SqueezeLLM | ONNX, TensorRT, vLLM, MLC-LLM, FastTransformer | 双语对话,代码生成 | https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct https://huggingface.co/Qwen/Qwen2.5-3B-Instruct | 中英双语优秀,工具调用能力强 |
| SmolLM2 | MosaicML | 135/360M/1.7B | safetensors | GPTQ(3/4/8bit), AWQ(4bit) | ONNX, TensorRT, MLC-LLM | 轻量级推理 | https://huggingface.co/collections/HuggingFaceTB/smollm2-6723884218bcda64b34d7db9 | 超轻量高效,适合边缘设备 |
| Gemma-2 | Google | 2B/7B | PyTorchsafetensors, JAX | GPTQ(4bit), AWQ(4bit), GGUF | JAX, ONNX, vLLM, MLC-LLM | 通用推理,教育应用 | https://huggingface.co/collections/google/gemma-2-release-667d6600fd5220e7b967f315 | 开放许可证,文档完善 |
| Llama-3.2 | Meta | 1B/3B | safetensors | GPTQ(4bit/8bit), AWQ(4bit) | ONNX, TensorRT, vLLM, MLC-LLM | 通用对话,多语言任务 | https://huggingface.co/meta-llama/Llama-3.2-1B-Instruct https://huggingface.co/meta-llama/Llama-3.2-3B-Instruct | 多语言支持,推理增强 |

* 基础规模 (Base Model Size)
    * 小型模型(1B以下): SmolLM2的135M和360M版本,适合资源受限场景
    * 中型模型(1-3B): Llama-3.2的1B/3B, Qwen2.5的1.5B/3B, internlm2.5的1.8B, Gemma-2的2B
    * 较大模型(3B以上): Phi-3-mini的3.8B, Gemma-2的7B版本
    * 不同规模适合不同应用场景,较小模型适合边缘部署,较大模型性能更好
* 原生格式 (Native Format)
    * safetensors: 主流格式,具有更好的安全性和加载速度
    * PyTorch格式: 兼容性好,生态完善
    * JAX格式: Google专有,性能优化
    * 多格式支持: 如Gemma-2同时支持多种格式,增加了灵活性
* 量化格式 (Quantization)
    * GPTQ: 最通用的量化方案,支持3/4/8-bit精度
    * AWQ: 新一代4-bit量化方案,精度损失更小
    * 专有格式: 如Qwen2.5的SqueezeLLM, internlm的W8A8等
    * exllama2: 针对消费级显卡优化的量化方案
* 部署格式 (Deployment)
    * ONNX: 跨平台标准格式,生态完善
    * TensorRT: NVIDIA GPU加速方案
    * vLLM: 高性能服务部署方案
    * MLC-LLM: 面向多种硬件的统一部署方案
    * 专有方案: 如FastTransformer(高性能推理),JAX(Google专用)等

## 二 MLC-LLM部署框架介绍
直接看官网[MLC LLM: Universal LLM Deployment Engine With ML Compilation](https://llm.mlc.ai/)介绍
> MLC LLM(Machine Learning Compiler LLM)是一个开源的统一大语言模型部署引擎，它主要由机器学习编译器（TVM）和高性能部署引擎(MLCEngine)组成，该项目的使命是让每个人都能在各种平台上原生地开发、优化和部署LLM模型。

![MLC_LLM工作流程](/img/MLC_LLM工作流程.png)

## 三 环境准备
### 3.1 Ubunut系统
### 3.2 conda （虚拟软件包管理环境）
使用conda可以和本地的软件包独立，尤其python的各种依赖容易冲突
```
# 下载安装脚本，需要科学上网
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

## 建议使用国内镜像地址
wget https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/Miniconda3-latest-Linux-x86_64.sh

## 运行安装
chmod 777 Miniconda3-latest-Linux-x86_64.sh
./Miniconda3-latest-Linux-x86_64.sh
## 使环境变量生效
source ~/.bashrc

## 注意安装后会自动生成一个base默认环境，可以退出base环境，并设置永久不激活
# 关闭自动激活base
conda config --set auto_activate_base false
# 如果后续要开启自动激活
conda config --set auto_activate_base true
conda deactivate

## 查看conda 多个配置文件层级：
1. 系统级配置文件 (通常在 conda 安装目录下的 .condarc)
2. 用户级配置文件 (~/.condarc)

conda config --show-sources

conda config --set show_channel_urls yes

## 配置国内镜像 参考https://mirrors.tuna.tsinghua.edu.cn/help/anaconda/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/

conda config --set show_channel_urls yes

```
### 3.3 Rust（本地环境安装）
编译Android so库的时候需要，pc 部署不需要
```
 curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs > rustup-init.sh

 chmod +x rustup-init.sh

 /rustup-init.sh 

 source $HOME/.cargo/env

 ## 查看rustup rustc cargo是否安装成功
 rustup --version
 rustc --version
 cargo --version
```
### 3.4 NDK（本地环境）并配置mlc环境变量

编译android库需要，pc不需要
ndk下载参考https://developer.android.com/studio?hl=zh-cn

配置
```
vi ~/.bashrc 
## 输入
export ANDROID_NDK=/home/win/Android/Sdk/ndk/27.0.11718014
export TVM_NDK_CC=$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android24-clang
export TVM_SOURCE_DIR=/home/win/llmworkspace/mlc-llm/3rdparty/tvm
export MLC_LLM_SOURCE_DIR=/home/win/llmworkspace/mlc-llm

source ~/.bashrc
```



## 四 下载和转换模型

### 4.1 创建conada mlc 转换模型环境 （conda mlc_pre环境）
```
# 创建conada mlc 转换模型环境
conda create -n mlc_pre python=3.11 (官方建议mlc依赖的python版本3.11)
# 进入conda环境
conda activate mlc_pre

conda install numpy=1.25.0 # 根据python版本决定
## 安装cpu版本mlc 参考https://llm.mlc.ai/docs/install/mlc_llm.html # Install MLC LLM Python Package

pip install --pre --force-reinstall mlc-ai-nightly-cpu mlc-llm-nightly-cpu -f https://mlc.ai/wheels

## 安装huggingface_hub 为了下载模型
pip install huggingface_hub
```

### 4.2 下载模型
#### 4.2.1 本地可以搞一个代码目录
```
mkdir /home/win/llmworkspace
cd  /home/win/llmworkspace
## 这个目录用来存储模型库
mkdir models 
```

#### 4.2.2 直接使用GPT生成一个下载脚本
```
from huggingface_hub import snapshot_download
import os

def download_model():
    # 设置国内镜像
    #os.environ['HF_ENDPOINT'] = "https://hf-mirror.com"

    # 设置本地保存路径
    local_dir = "/home/win/llmworkspace/models/Qwen/Qwen2.5-3B-instruct"

    # 创建保存目录
    os.makedirs(local_dir, exist_ok=True)

    try:
        # 下载模型
        snapshot_download(
            repo_id="Qwen/Qwen2.5-3B-instruct",
            local_dir=local_dir,
            token="*********"  # 替换为你的 HuggingFace token
        )
        print(f"模型已成功下载到: {local_dir}")

    except Exception as e:
        print(f"下载过程中出现错误: {str(e)}")

if __name__ == "__main__":
    download_model()
```

#### 4.2.3 下载模型
```
# conda mlc_pre环境 执行下载脚本
python downloadmodel.py

## 下载完成后输出如下信息，Qwen2.5-3B-instruct 大概5.8G
模型已成功下载到: /home/win/llmworkspace/models/Qwen/Qwen2.5-3B-instruct███████████████████████████████████████████████████████████████████████████████████| [04:26<00:00, 12.0MB/s]
```

## 4.3 转换模型为MLC格式
#### 4.3.1 直接GPT生成一个转换脚本convermode_pc.py 
```
import os

# 定义变量
MODEL_NAME = "Qwen2.5-3B-Instruct"
QUANTIZATION = "q4f16_1" 
MODEL_PATH = "/home/win/llmworkspace/models/Qwen"
OUT_DIR = "/home/win/llmworkspace/models/ConvertMode"

def convert_model():
    try:
        # 1. 转换模型权重
        print("开始转换模型权重...")
        os.system(f"python -m mlc_llm convert_weight {MODEL_PATH}/{MODEL_NAME} "
                 f"--source-format huggingface-safetensor "  # 或 huggingface
                 f"--quantization {QUANTIZATION} "
                 f"-o {OUT_DIR}/{MODEL_NAME}-{QUANTIZATION}/")

        # 2. 生成配置文件
        print("生成配置文件...")
        os.system(f"""python -m mlc_llm gen_config {MODEL_PATH}/{MODEL_NAME} \
            --quantization {QUANTIZATION} \
            --conv-template qwen2 \
            --context-window-size 2048 \
            --prefill-chunk-size 512 \
            -o {OUT_DIR}/{MODEL_NAME}-{QUANTIZATION}/""")

        print("转换完成！")
        print(f"输出文件位置: {OUT_DIR}/{MODEL_NAME}-{QUANTIZATION}-android/{MODEL_NAME}-{QUANTIZATION}")

    except Exception as e:
        print(f"转换过程中出现错误: {str(e)}")

if __name__ == "__main__":
    convert_model()
```

#### 4.3.2 conda mlc_pre环境 执行转换脚本
```
python convertmode_pc.py
```
使用q4f16_1转换后模型大概1.7G

## 4.4 PC部署运行
pc上面比较简单，conda mlc_pre环境 运行
```
mlc-llm chat /home/win/llmworkspace/models/ConvertMode/Qwen2.5-3B-Instruct-q4f16_1

### 下面就可以聊天了，这是我的聊天记录

You can use the following special commands:
  /help               print the special commands
  /exit               quit the cli
  /stats              print out stats of last request (token/sec)
  /metrics            print out full engine metrics
  /reset              restart a fresh chat
  /set [overrides]    override settings in the generation config. For example,
                      `/set temperature=0.5;top_p=0.8;seed=23;max_tokens=100;stop=str1,str2`
                      Note: Separate stop words in the `stop` option with commas (,).
  Multi-line input: Use escape+enter to start a new line.

>>> hello
Hello! How can I assist you today?
>>> 小明比小红年龄大，小红比小华年龄大。谁是年龄最小的？
根据题目中的信息，我们知道：

1. 小明比小红年龄大。
2. 小红比小华年龄大。

从这两个信息中可以得出小明年龄最大，小华年龄最小。所以，年龄最小的是小华。
>>> 如果一个气球被放在太阳下暴晒，会发生什么？为什么？
当气球被放在太阳下暴晒时，气球内部的空气会因为温度升高而膨胀。具体来说，以下几个因素会导致气球发生的变化：

1. **温度升高**：太阳光直接照射在气球上，使气球表面温度升高。
2. **热胀冷缩**：温度升高会导致气球内部气体分子的平均动能增加，分子之间的平均距离增加，从而使得气球内部气体膨胀。
3. **气球材料变化**：气球材料（通常是橡胶或塑料）在高温下可能会发生变形或变软，这取决于材料的热稳定性。

最终结果是，气球内部的压力增加，而气球外部的压力相对较小（因为外部环境温度较低）。这种压力差会导致气球内部的压力超过气球材料所能承受的最大压力，从而使气球膨胀，甚至可能破裂。

所以，气球被太阳暴晒时，内部气体膨胀，气球可能会因为压力增大而破裂。
>>>

好的，如果气球被放在太阳下暴晒，可能会发生气球破裂的情况。这是因为气球内部的气体在高温下膨胀，而气球材料在高温下可能变得不那么牢固，导致气球破裂。具体情况取决于气球的材料和温度条件。

如果你有其他问题或需要进一步的信息，请告诉我！
>>>
当然，我可以继续帮助你！如果你有关于气球在太阳下暴晒的问题或者其他任何问题，请随时告诉我。你可以具体描述一下你关心的情况，我会尽力解答。
>>>
好的，有什么具体问题或需要进一步的信息，请告诉我！我会尽力帮助你。
```
## 4.5 Android部署
这是官方文档说明可以参考
https://llm.mlc.ai/docs/deploy/android.html
#### 4.5.1 转换为Android格式
之前的convertmode_pc 脚本修改一下,另外我用的pixel3a手机，不支持gpu运行，性能太差，用1.5参数的模型（Qwen2.5-1.5B-Instruct）下载这个模型用前面的下载脚本就行。
```
import os

# 定义变量
MODEL_NAME = "Qwen2.5-1.5B-Instruct"
QUANTIZATION = "q4f16_1"
MODEL_PATH = "/home/win/llmworkspace/models/Qwen"
OUT_DIR = "/home/win/llmworkspace/models/Android"

def convert_model():
    try:
        # 1. 转换模型权重
        print("开始转换模型权重...")
        os.system(f"python -m mlc_llm convert_weight {MODEL_PATH}/{MODEL_NAME} "
                 f"--source-format huggingface-safetensor "  # 或 huggingface
                 f"--quantization {QUANTIZATION} "
                 f"-o {OUT_DIR}/{MODEL_NAME}-{QUANTIZATION}-android/")

        # 2. 生成配置文件
        print("生成配置文件...")
        os.system(f"""python -m mlc_llm gen_config {MODEL_PATH}/{MODEL_NAME} \
            --quantization {QUANTIZATION} \
            --conv-template qwen2 \
            --context-window-size 2048 \
            --prefill-chunk-size 512 \
            -o {OUT_DIR}/{MODEL_NAME}-{QUANTIZATION}-android/""")

        # 3. 编译为安卓格式
        print("编译为安卓格式...")
        os.system(f"""python -m mlc_llm compile \
            {OUT_DIR}/{MODEL_NAME}-{QUANTIZATION}-android/mlc-chat-config.json \
            --device android \
            -o {OUT_DIR}/{MODEL_NAME}-{QUANTIZATION}-android/{MODEL_NAME}-{QUANTIZATION}-android.tar""")

        print("转换完成！")
        print(f"输出文件位置: {OUT_DIR}/{MODEL_NAME}-{QUANTIZATION}-android/{MODEL_NAME}-{QUANTIZATION}-android.tar")

    except Exception as e:
        print(f"转换过程中出现错误: {str(e)}")

if __name__ == "__main__":
    convert_model()
```

#### 4.5.2 mlc-llm 源码下载
注意依赖前置环境rust和ndk
```
# 下载mlc-llm 源码
git clone https://github.com/mlc-ai/mlc-llm.git

cd mlc-llm

git submodule update --init --recursive
cd android/MLCChat

mkdir dist
cd dist
mkdir lib
cd lib
## copy 转换后的模型到/dist/lib

cp /home/win/llmworkspace/models/Android/Qwen2.5-1.5B-Instruct-q4f16_1-android/Qwen2.5-1.5B-Instruct-q4f16_1-android.tar /home/win/llmworkspace/mlc-llm/android/MLCChat/dist/lib/

备份原有的mlc-package-config
mv /home/win/llmworkspace/mlc-llm/android/MLCChat/mlc-package-config.json /home/win/llmworkspace/mlc-llm/android/MLCChat/mlc-package-config.json.back

```

#### 4.5.3 自定义mlc-package-config文件
修改mlc-package-config 为自己的模型。
```
vi /home/win/llmworkspace/mlc-llm/android/MLCChat/mlc-package-config.json

{
  "device": "android",
  "model_list": [
    {
      "model": "/home/win/llmworkspace/models/Android/Qwen2.5-1.5B-Instruct-q4f16_1-android",
      "bundle_weight": true,
      "model_id": "qwen2_q4f16_1",
      "model_lib": "qwen2_q4f16_1",
      "estimated_vram_bytes": 1630522470,
      "overrides": {
        "context_window_size": 512,
        "prefill_chunk_size": 128
      }
    }
  ],
  "model_lib_path_for_prepare_libs": {
    "qwen2_q4f16_1": "./dist/lib/Qwen2.5-1.5B-Instruct-q4f16_1-android.tar"
  }
}

```

#### 4.5.4 编译so库

```
conda mlc_pre环境 编译mlc 源码

cd /home/win/llmworkspace/mlc-llm/android/MLCChat/

## 编译
mlc_llm package

## /home/win/llmworkspace/mlc-llm/android/MLCChat/这个目录下输出
dist
└── lib
    └── bundle
        ├── qwen2.5-1.5b-q4f16_1
    └── mlc4j
        ├── build.gradle
        ├── output
        │   ├── arm64-v8a
        │   │   └── libtvm4j_runtime_packed.so
        │   └── tvm4j_core.jar
        └── src
            ├── cpp
            │   └── tvm_runtime.h
            └── main
                ├── AndroidManifest.xml
                ├── assets
                │   └── mlc-app-config.json
                └── java
                    └── ...
```

#### 4.5.5 编译和运行apk并push权重参数

```
我是把上面的/home/win/llmworkspace/mlc-llm/android/MLCChat/ copy到一个新的android studio 工程单独编译的。

# android studio 编译apk

#conda mlc_pre 环境执行脚本

cd /home/win/project/demo/MLCChat/

python bundle_weight.py --apk-path 
/home/win/project/demo/MLCChat/app/build/outputs/apk/debug/app-debug.apk

```
其实bundle_weight.py 这个脚本干了3件事,也可以手动执行
```
1.adb install xxx.apk
2.adb push //home/win/project/demo/MLCChat/dist/bundle/qwen2.5-1.5b-q4f16_1 /data/local/tmp/qwen2.5-1.5b-q4f16_1
3.mv /data/local/tmp/qwen2.5-1.5b-q4f16_1 /storage/emulated/0/Android/data/ai.mlc.mlcchat/files/
```

#### 开始对话
![android运行结果](/img/mlc-llm-android运行结果.jpg)

* 评价
pixel3a 手机，纯cpu运行，经常内存溢出，可以用更好的手机

## 引用

* [Install MLC LLM Python Package](https://llm.mlc.ai/docs/install/mlc_llm.html)

* https://llm.mlc.ai/docs/deploy/android.html

* [[ML Story] MobileLlama3: Run Llama3 locally on mobile](https://medium.com/google-developer-experts/ml-story-mobilellama3-run-llama3-locally-on-mobile-36182fed3889)

* [清华大学开源软件镜像站](https://mirrors.tuna.tsinghua.edu.cn/)

* https://mirrors.tuna.tsinghua.edu.cn/help/anaconda/