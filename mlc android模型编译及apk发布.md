

# MLC Android 中文文档

## 开始使用

参考自[MLC使用文档](https://llm.mlc.ai/docs/index.html)

### MLC介绍

这里稍微讲解了一些MLC的基本概念，以帮助我们使用和了解 MLC LLM。

MLC-LLM 由三个不同的子模块组成：模型定义、模型编译和模型运行。

 ![项目结构](https://llm.mlc.ai/docs/_images/project-structure.svg) 

<center>MLC LLM 的三个独立子模块</center>

**➀ Python 中的模型定义。**MLC 提供各种预定义架构，例如 Llama（例如 Llama2、Vicuna、OpenLlama、Wizard）、GPT-NeoX（例如 RedPajama、Dolly）、RNN（例如 RWKV）和 GPT-J（例如MOSS）。开发人员可以仅使用纯 Python 定义模型，而无需接触编码。

**➁ Python 中的模型编译。**[模型由TVM Unity](https://llm.mlc.ai/docs/install/tvm.html)编译器编译，其中编译配置为纯 Python。MLC LLM 将基于 Python 的模型量化导出到模型库并量化模型权重。可以用纯 Python 开发量化和优化算法，以针对特定用例压缩和加速 LLM。

**➂ 不同平台的模型运行。**每个平台上都提供了 MLCChat 的变体：用于命令行的**C++** 、用于 Web 的**Javascript **、用于 iOS 的**Swift** 和用于 Android 的**Java**，可通过 JSON 进行配置。开发人员只需熟悉平台即可将 MLC 编译的 LLM 集成到他们的项目中。

### MLC相关术语

MLC中有一些自己定义的术语，我们通过了解这些基本术语，对后面的使用会提供很大的帮助。

- **modelweights**：模型权重是一个文件夹，其中包含语言模型的量化神经网络权重以及分词器配置。
- **model lib**：模型库是指能够执行特定模型架构的可执行库。在 Linux 上，这些库文件的后缀为 `.so`，在 macOS 上，后缀为`.dylib`，在 Windows 上，后缀为`.dll`，在WebGPU上，后缀是`.wasm`。
- **chat config**：推理配置，包含允许的自定义参数（例如Temperature 和system prompt）的设置。推理配置与模型权重默认位于同一个目录中。推理配置还包含下面两个在多种模型设置中都支持使用的元数据字段。
  - `local_id`，应用程序中模型的唯一标识
  - `model_lib`，指定要使用哪个模型库。

### MLC运行流程图

配置好模型权重、模型库和推理设置后，MLC提供了各种MLC 聊天程序帮助用户能直接使用模型，下图显示了 MLC 聊天程序的工作流程。

![https://raw.githubusercontent.com/mlc-ai/web-data/de9a5e5b424f36119bd464ddf5a3ddb4c58cc85e/images/mlc-llm/tutorials/mlc-llm-flow.svg](https://raw.githubusercontent.com/mlc-ai/web-data/de9a5e5b424f36119bd464ddf5a3ddb4c58cc85e/images/mlc-llm/tutorials/mlc-llm-flow.svg)

图右侧的伪代码，说明了聊天应用程序的结构。

`ChatModule`管理模型的类。

`chat.reload`模型加载，通过`local_id`来确定需要加载的模型，`model_lib`来确定需要加载的模型库，同时允许覆盖Temperature 和system prompt等设置

`chat.generate`结果生成，`input`为需要输入的内容，`stream_callbace`为多轮对话的轮数

所有的 MLC聊天程序 运行时（包括 iOS、Web、CLI 等）都会使用这三个元素。所以运行时读取的都是同一个模型权重文件夹。而模型库会根据编译时环境的选择而生成不同的文件。其中 CLI的模型库存储在 DLL 目录中。iOS 和 Android 由于动态加载限制，在应用程序中就预先打包好了模型库。WebLLM则是利用URL来访问本地文件或互联网的 WebAssembly (Wasm) 文件 。

下面我会基于自己尝试过的Android程序，从模型编译到Android apk打包进行一个详细的介绍。

## 基本环境配置

### Anaconda安装

在MLC中需要使用的python最低版本在3.10以上，为避免环境冲突，所以最好使用虚拟环境来进行环境配置，这里推荐使用Anconda来配置虚拟环境，相关的安装方法，可参考[Anaconda安装](https://zhuanlan.zhihu.com/p/32925500)

创建虚拟环境，后续的一些操作都尽量在虚拟环境中去操作

```shell
# 创建一个 your-environment的虚拟环境，python版本为3.10
conda create -n your-environment python==3.10 
# 进入虚拟环境
activate your-environment
```



## 模型编译

模型编译包括使用MLC、Distribute和Python API三种方式，当前文档主要采用MLC的方法进行编译，如果想尝试其他的方式，可以参考[MLC使用文档](https://llm.mlc.ai/docs/index.html)

MLC的模型编译包含了在各种各样平台的编译，如上面所说的，我们主要是为了生成不同模型的一个模型库，因为在不同的系统中，模型库的后缀都不一样。

### 安装MLC-LLM包

参考官网，可以直接克隆存储库

```powershell
# 克隆源代码及源代码附属的子项目
git clone https://github.com/mlc-ai/mlc-llm.git --recursive
# 进入mlc-llm文件夹
cd mlc-llm
# 将mlc-llm安装进环境
pip install .
```

> 这里有一个坑，对国内用户来说，受限于网络，使用git clone ** --recursive会出现失败的可能
>
> 这里就可以考虑用
>
> ```shell
> # 克隆源代码
> git clone https://github.com/mlc-ai/mlc-llm.git
> # 进入mlc-llm文件夹
> cd mlc-llm
> # 加载子项目，如果加载失败，可反复运行，直到运行后没有返回为止
> git submodule update --init --recursive
> ```

### 验证安装

```shell
# linux 和 mac 中运行
python3 -m mlc_llm.build --help
# windows运行
python -m mlc_llm.build --help
```

通过上面的代码，可以看到对应的帮助信息才是构建好了mlc的编译环境

### 模型下载

还是由于环境限制，已经网络流量限制，我这边是使用国内大佬提供的模型

Llama2模型下载地址

```
Llama2-7B Hugging Face版本：https://pan.xunlei.com/s/VN_t0dUikZqOwt-5DZWHuMvqA1?pwd=66ep

Llama2-7B-Chat Hugging Face版本：https://pan.xunlei.com/s/VN_oaV4BpKFgKLto4KgOhBcaA1?pwd=ufir

Llama2-13B Hugging Face版本：https://pan.xunlei.com/s/VN_yT_9G8xNOz0SDWQ7Mb_GZA1?pwd=yvgf

Llama2-13B-Chat Hugging Face版本：https://pan.xunlei.com/s/VN_yA-9G34NGL9B79b3OQZZGA1?pwd=xqrg

Llama2-70B-Chat Hugging Face版本：https://pan.xunlei.com/s/VNa_vCGzCy3h3N7oeFXs2W1hA1?pwd=uhxh#
```

这里推荐下载Llama2-7B-Chat

### 开始编译

官方原文档中包含了6中选择，这里就不重复了，只介绍Android的模型编译

在cmd命令行中直接输入

```shell
# linux or mac
python3 -m mlc_llm.build --model Llama-2-7b-chat-hf --target android --max-seq-len 768 --quantization q4f16_1
# windows
python -m mlc_llm.build --model Llama-2-7b-chat-hf --target android --max-seq-len 768 --quantization q4f16_1
```

如果编译成功，则会在 `dist`目录下生成对应的编译后文件，文件名一般是`Llama-2-7b-chat-hf-q4f16_1`，，里面包含模型文件、对应环境的模型库。

#### 编译命令解释

编译命令框架

```shell
python3 -m mlc_llm.build \
    --model MODEL_NAME_OR_PATH \
    [--hf-path HUGGINGFACE_NAME] \
    --target TARGET_NAME \
    --quantization QUANTIZATION_MODE \
    [--max-seq-len MAX_ALLOWED_SEQUENCE_LENGTH] \
    [--reuse-lib LIB_NAME] \
    [--use-cache=0] \
    [--debug-dump] \
    [--use-safetensors]
```

- **`python3 -m mlc_llm.build`** 运行mlc_llm模块下的build 函数

- **`--model MODEL_NAME_OR_PATH`**指定要编译的模型名称，根据`--model`或者`--hf-path`指定模型路径

其中`--model`为选择本地模型，mlc将在本地的路径中去进行搜索，存放模型的文件夹要求在mlc-llm下

例如：

`--model Llama-2-7b-chat-hf`   ---> `../mlc-llm/dist/models/Llama-2-7b-chat-hf`

`--model /my-model/Llama-2-7b-chat-hf`  ---> `../mlc-llm/my-model/Llama-2-7b-chat-hf`

- **`--hf-path HUGGINGFACE_NAME `**

  为选择模型在 Hugging Face存储库中的名称，会将模型下载至`../mlc-llm/dist/models/Llama-2-7b-chat-hf`

- **`--target TARGET_NAME`**

  选择编译模型的目标平台，默认值：`auto`,将会根据当前的机器的情况使用`cuda`、`metal`、`vulkan` 或`opencl`，其他可选项有：`metal`(适用于 M1/M2）、`metal_x86_64`（适用于 Intel CPU）、`iphone`、 `vulkan`、`cuda`、`webgpu`、`android`和`opencl`。 

  > 小知识
  >
  > [metal](https://baike.baidu.com/item/Metal/10917053?fr=aladdin)             [vulkan](https://baike.baidu.com/item/Vulkan/17543632?fr=ge_ala)         [opencl](https://baike.baidu.com/item/OpenCL/8477301?fr=ge_ala)

- **`--quantization QUANTIZATION_MODE `**

  编译的量化模式。代码的格式为`qAfB(_0)`，其中`A`表示存储权重的位数，`B`表示存储激活的位数。可用选项有：`q3f16_0`、`q4f16_1`、`q4f16_2`、`q4f32_0`、`q0f32`和`q0f16`。MLC鼓励使用 4 位量化，因为根据模型的不同，3 位量化模型生成的文本可能质量较差。

其余是可选参数：

- **`--max-seq-len MAX_ALLOWED_SEQUENCE_LENGTH`**

  模型允许的最大序列长度。如果未指定，将会使用`config.json`模型配置文件中的最大序列长度。

- **`--reuse-lib LIB_NAME`**

  指定要重用的先前生成的库。当构建具有不同权重的相同模型架构时，这非常有用。有关此参数的详细信息，您可以参考[模型分布页面。](https://llm.mlc.ai/docs/compilation/distribute_compiled_models.html#distribute-model-step3-specify-model-lib)

- **`--use-cache`**

  指定后`--use-cache=0`，模型编译将不会使用以前构建的缓存文件，而是从一开始就编译模型。使用缓存可以帮助减少编译所需的时间。

- **`--debug-dump`**

  指定编译期间是否转储调试文件。

- **`--use-safetensors`**

  指定加载模型权重时是否使用`.safetensors`，默认值：`.bin`



## Android应用程序打包

### 前言

模型编译安装教程配置好对应的环境即可，但是Android应用程序打包需要特别注意，需要配置的环境变量很多，而且在纯windows系统上进行尝试会出现很多奇奇怪怪的报错，建议最好是在**有图形化的ubuntu系统**上进行打包，或者利用ubuntu去生成对应的模型构建依赖。只有windows系统的可以考虑安装[WLS2](http://www.bryh.cn/a/315513.html)

### 环境配置

#### TVM Unity编辑器安装

程序打包也会需要用到python环境，以及会使用**[TVM Unity 编译器](https://llm.mlc.ai/docs/install/tvm.html)** ，建议利用Anaconda去重新创建一个新的虚拟环境，去进行打包

```shell
conda activate your-environment
python -m pip install --pre -U -f https://mlc.ai/wheels mlc-ai-nightly
```

由于mlc-ai-nigtly这个包不在国内的pip源中，需要在mlc.ai网站的wheels中去进行下载，如果收环境限制的同学，可以考虑在github中直接下载对应版本的whl包，[mlc github package](https://github.com/mlc-ai/package/releases/tag/v0.9.dev0)

例如我在ubuntu中使用的是，注意要在whl所在的目录运行以下命令

```shell
conda activate your-environment
python -m pip install mlc_ai_nightly-0.12.dev1880-cp310-cp310-manylinux_2_28_x86_64.whl
```

TVM的安装验证：以下命令可以帮助确认 TVM 是否已正确安装为 python 包并提供 TVM python 包的位置 

```shell
>>> python -c "import tvm; print(tvm.__file__)"
/some-path/lib/python3.10/site-packages/tvm/__init__.py
```

#### Rust安装

将 HuggingFace 标记器交叉编译到 Android 需要[**Rust**](https://www.rust-lang.org/tools/install) 。

在windows中，按照下载的rustup-init.exe，直接按照默认设置安装即可

在linux中，可以执行下面命令进行安装

```shell
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

安装完成后，可以在命令行输入一下命令检验是否安装成功，如果没有结果，则表示安装失败，可以尝试重新安装rust。

```shell
rustc --version
```

#### Android Studio安装

进行apk的打包，需要使用Android Studio

要使用Android Studio就需要安装 SDK、NDK、CMake和JAVA

其中[Android Studio](https://developer.android.com/studio)可以在官网中去下载，windows直接执行安装程序，按照默认设置安装即可，linux则是将tar.gz文件解压到对应的目录即可。运行以下命令即可开始使用android studio（需要图形化界面）

```shell
./bin/studio.sh
```

在 Android Studio 单击“File → Settings → Languages & Frameworks → Android SDK → SDK Tools”，选择安装NDK、CMake和Android SDK Platform-Tools。安装完成后，需要在环境变量中去对NDK等进行配置才可使用。

windows中，可以根据SDK安装的目录，去设置环境变量，比如我的SDK安装在了D盘，则设置的环境变量有：

```shell
ANDROID_HOME  :  D:\Program\Android\Sdk
ANDROID_NDK   :  %ANDROID_HOME%\ndk\26.1.10909125
NDK_ROOT      :  %ANDROID_HOME%\ndk\26.1.10909125
Path          :  %ANDROID_HOME%\cmake\3.22.1\bin
Path          :  %ANDROID_HOME%\platform-tools
TVM_NDK_CC    :  %ANDROID_NDK%\toolchains\llvm\prebuilt\windows-x86_64\bin\aarch64-linux-android24-clang
TVM_HOME      :  D:\code\mlc-llm\3rdparty\tvm
```

在linux中设置的环境变量：

```shell
export ANDROID_NDK=/home/User/Android/Sdk/ndk/26.1.10909125
export ANDROID_HOME=/home/User/Android/Sdk
export PATH=$PATH:/home/User/Android/Sdk/cmake/3.22.1/bin
export PATH=$PATH:/home/User/Android/Sdk/platform-tools
export TVM_NDK_CC=$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android24-clang
export TVM_HOME=/home/User/mlc-llm/3rdparty/tvm
export TVM_HOME=$TVM_HOME/include/tvm/runtime
source $HOME/.cargo/env # Rust
```

另外，Android Studio需要安装openjdk，官方要求的版本是>17

在Windows中，可以在 [JAVA](https://jdk.java.net/java-se-ri/17)中下载，下载后解压到自定义的目录下，然后配置环境变量即可

```shell
JAVA_HOME  :  D:\Program\java\java-se-8u43-ri
JRE_HOME   ： D:\Program\java\java-se-8u43-ri\jre
Path       :  %JAVA_HOME%\bin
Path       :  %JRE_HOME%\bin
```

在linux中，可以通过以下命令安装JAVA 并配置环境

```shell
# 更新update
sudo apt update
# 安装openjdk17
sudo apt install openjdk-17-jdk 
# 查看jdk17的安装路径
sudo update-alternatives --list java
# 用上面命令获取的路径，编入到bashrc文件的最后一行中
vi ~/.bashrc
# 将下面的命令，编入到bashrc文件的最后一行中
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64/bin/java
# 更新环境变量
source ~/.bashrc
```

#### 模型列表文件更新

在git下的源代码中，模型列表文件加入了三个模型，如果我们没有这么多模型的话，构建会报错，所以我们只需要把自己需要构建的模型填上去，多余的删掉

```
vim ./MLCChat/app/src/main/assets/app-config.json
```

### 模型依赖构建

构建依赖最好切换的对应的anaconda虚拟环境中去构建，因为会使用python。

```shell
# 进入虚拟环境
activate your-environment
# 进入android构建目录
cd ./mlc-llm/android/
# 开始构建
./prepare_libs.sh
```

如果是在Windows环境，可以直接在Git Bash下运行构建，在cmd或者powrshell下，构建失败会闪退，无法看到对应的报错。

> 相关报错解决方法：
>
> `Could NOT find JNI`
>
> 如果出现上面的报错，则需要修改`android`目录下的`CMakelist.txt`，在find_package(JNI REQUIRED)前面添加
>
> ```shell
> # windows
> set(JAVA_AWT_LIBRARY "D:/Program/java/jdk-8.0.302.8-hotspot/jdk8u302-b08/include/win32")
> set(JAVA_JVM_LIBRARY "D:/Program/java/jdk-8.0.302.8-hotspot/jdk8u302-b08/include/win32")
> set(JAVA_INCLUDE_PATH2 "D:/Program/java/jdk-8.0.302.8-hotspot/jdk8u302-b08/include")
> set(JAVA_AWT_INCLUDE_PATH "D:/Program/java/jdk-8.0.302.8-hotspot/jdk8u302-b08/include")
> 
> # linux
> set(JAVA_AWT_LIBRARY "/usr/lib/jvm/java-17-openjdk-amd64/include/linux")
> set(JAVA_JVM_LIBRARY "/usr/lib/jvm/java-17-openjdk-amd64/include/linux")
> set(JAVA_INCLUDE_PATH2 "/usr/lib/jvm/java-17-openjdk-amd64/include")
> set(JAVA_AWT_INCLUDE_PATH "/usr/lib/jvm/java-17-openjdk-amd64/include")
> ```

> ` $<LINK_LIBRARY:WHOLE_ARCHIVE,tvm_runtime>`
>
> 如果出现上述报错，可以参考github中的[#134](https://github.com/mlc-ai/mlc-llm/issues/134)
>
> 修改`mlc-llm`目录下的`CMakelist.txt`
>
> 将`target_link_libraries(mlc_chat_cli PRIVATE "$<LINK_LIBRARY:WHOLE_ARCHIVE,tvm_runtime>")`
>
> 替换为`target_link_libraries(mlc_chat_cli PRIVATE mlc_llm_static sentencepiece-static tokenizers tvm_runtime)`即可

其余问题都可在[github中提问](https://github.com/mlc-ai/mlc-llm/issues)，会有专业的人事帮忙解决。

如果构建成功，会在`./build/output`目录下出现下面两个文件

```
./build/output/arm64-v8a/libtvm4j_runtime_packed.so
./build/output/tvm4j_core.jar
```

我们只需要将他们复制到Android的项目中，然后再进行apk的打包即可

```shell
cp -a ./build/output/. ./MLCChat/app/src/main/libs
```

### apk构建

apk的构建我们需要在Android studio 中进行

**启动Android Studio**：将`./android/MLCChat`文件作为Android studio 项目打开。

**生成 APK**：点击“Build→Generate Signed Bundle/APK”构建APK进行发布。如果是第一次生成APK，则需要根据[Android官方指南](https://developer.android.com/studio/publish/app-signing#generate-key)创建密钥。此 APK 将放置在`android/MLCChat/app/release/app-release.apk`.

**安装ADB和USB调试**：在手机设置的开发者模式中启用“USB 调试”。运行以下命令，如果ADB安装正确，您的手机将显示为设备：(这里用到的adb 是上面在安装Android studio时安装好的`Android SDK Platform-Tools`，并且环境已配置好，如果adb不可用，考虑检查环境配置和重新安装)

```
adb devices
```

**将 APK 和权重安装到您的手机上**：运行以下命令，将`${MODEL_NAME}`和替换`${QUANTIZATION}`为实际模型名称（例如 Llama-2-7b-chat-hf）和量化格式（例如 q4f16_1）。

```shell
# 将apk安装进你的android手机
adb install android/MLCChat/app/release/app-release.apk
# 将模型权重文件上传至android手机的临时文件夹
adb push dist/${MODEL_NAME}-${QUANTIZATION}/params /data/local/tmp/${MODEL_NAME}-${QUANTIZATION}/
# 在android手机上创建apk读取本地模型的文件夹路径
adb shell "mkdir -p /storage/emulated/0/Android/data/ai.mlc.mlcchat/files/"
# 将模型拷贝至apk读取文件夹路径
adb shell "mv /data/local/tmp/${MODEL_NAME}-${QUANTIZATION} /storage/emulated/0/Android/data/ai.mlc.mlcchat/files/"
```

走完上面的操作，你就可以在自己的android手机上去运行llama2了

## 运行截图

在飞行模式下成功运行的截图

![1701912957513](https://github.com/mlc-ai/mlc-llm/assets/57121762/0272eeeb-5fd1-42fa-99ab-6f6454374e0a)

## 引用

① [MLC AI DOC](https://llm.mlc.ai/docs/index.html)

② [mlc-llm github](https://github.com/mlc-ai/mlc-llm)