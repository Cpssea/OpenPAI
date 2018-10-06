<font size="6"> OpenPAI上运行实验</font>
****
<font size="6">目录</font>
<!-- TOC -->

    - [创建环境](#创建环境)
        - [依赖项](#依赖项)
        - [NNI 安装](#nni-安装)
    - [运行实验](#运行实验)
- [查看实验结果](#查看实验结果)
        - [NNI WebUI](#nni-webui)
    - [贡献](#贡献)
    - [协议](#协议)
    - [附录](#附录)
        - [语言](#语言)
        - [结构](#结构)
        - [图文信息对应](#图文信息对应)
        - [名词解释](#名词解释)
        - [命令行](#命令行)
        - [Note & Notice](#note--notice)
        - [Contribution](#contribution)
        - [License](#license)

<!-- /TOC -->

NNI 支持在[OpenPAI](https://github.com/Microsoft/pai)(aka pai)上运行实验，此种模式叫做PAI 模式。用户的实验程序在PAI docker容器中运行。

OpenPAI是一个开源的大规模人工智能集群管理平台，支持完整的AI模型训练及资源管理功能，易于扩展，且支持不同规模的本地，云和混合环境。

> **注意**：使用NNI PAI模式前提是先拥有访问[OpenPAI](https://github.com/Microsoft/pai)集群的账户,关于如何部署使用OpenPAI,请参考[此处](https://github.com/Microsoft/pai#how-to-deploy)。

## 创建环境
****
### 依赖项
- python >= 3.5
- git
- wget

执行以下命令检查linux系统中是否正确安装了python pip。
```
python3 -m pip -V
```
>**Note**: 当前版本不支持虚拟环境。

### NNI 安装
- 通过pip安装NNI
```
python3 -m pip install -v --user 
git+https://github.com/Microsoft/nni.git@v0.2
source ~/.bashrc
```
- 通过源码安装NNI
```
git clone -b v0.2 https://github.com/Microsoft/nni.git
cd nni
chmod +x install.sh
source install.sh
```
关于NNI的安装和快速入门请参阅[此处](https://github.com/Microsoft/nni/blob/master/docs/GetStarted.md)。

## 运行实验
****
实验过程中会执行多个作业（Job），每个作业都需要配置具体的神经网络结构（或模型）和超参数值。因此要在NNI上进行实验，用户应该完成以下四步：

1. 提供可执行实验

2. 提供或选择一个优化器（tuner）

3. 提供或选择yaml配置文件

4. 提供或选择评估器（可选）

例：以`examples/trials/mnist-annotation`为例

NNI安装完毕后，用户可以执行`ls ~/nni/examples/trials`查看所有的实验示例。

**第一步：执行以下命令，来运行NNI mnist 示例:**
```
python ~/nni/examples/trials/mnist-annotation/mnist.py
```
此命令将写入yaml配置文件中。

**第二步：预调解：**

**Tuner**：NNI支持几种流行的自动机器学习的算法，包括随机搜索，TPE，评估算法等。用户可以编写自己的tuner（参见[此处](https://github.com/Microsoft/nni/blob/master/docs/CustomizedTuner.md)），但为了简单起见，我们选择NNI提供的tuner，如下：
```
tuner:
    builtinTunerName: TPE
    classArgs:
      optimize_mode: maximize
```
**builtinTunerName**: 确定算法TPE

**classArgs**: 传给tuner的参数

**optimization_mode**: 选择最大化还是最小化实验结果

**第三步：配置yaml文件**

NNI为每个实验提供了一个demo配置文件，执行以下命令可以看到配置文件：
```
cat ~/nni/examples/trials/mnist-annotation/config.yml
```
yaml文件内容如下所示：
```
authorName: your_name
experimentName: auto_mnist
# how many trials could be concurrently running
trialConcurrency: 2
# maximum experiment running duration
maxExecDuration: 3h
# empty means never stop
maxTrialNum: 100
# choice: local, remote, pai
trainingServicePlatform: pai
# choice: true, false  
useAnnotation: true
tuner:
  builtinTunerName: TPE
  classArgs:
    optimize_mode: maximize
trial:
  command: python3 mnist.py
  codeDir: ~/nni/examples/trials/mnist-annotation
  gpuNum: 0
  cpuNum: 1
  memoryMB: 8196
  image: openpai/pai.example.tensorflow
  dataDir: hdfs://10.1.1.1:9000/nni
  outputDir: hdfs://10.1.1.1:9000/nni
# Configuration to access OpenPAI Cluster
paiConfig:
  userName: your_pai_nni_user
  passWord: your_pai_password
  host: 10.1.1.1
```
**trainingServicePlatform**: 三种模式可供选择：本地，远程和PAI模式，此处选择PAI 模式。

与本地和远程机器模式相比，PAI模式下的文件配置有另外五个参数：
- **cpu Num**
   * 必填参数。根据实验程序的CPU需求设置合适的CPU个数
- **memoryMB**
   * 必填参数。根据实验程序的内存需求设置内存空间大小（以MB为内存单位）。
- **image**
   * 必填参数。PAI模式中，实验程序将在Docker容器中进行调度，该参数指定了实验运行的容器的Docker镜像。
   * 我们已经在[Docker Hub](https://hub.docker.com/)上创建了一个docker镜像[nnimsra / nni](https://hub.docker.com/r/msranni/nni/)。它包含NNI python包，运行实验所需的Node模块和javascript构件文件，以及所有NNI依赖。用于创建此docker镜像的文件可查阅[此处](https://github.com/Microsoft/nni/blob/master/deployment/Dockerfile.build.base)。你可以直接在配置文件中使用此镜像，也可以基于它创建自己的镜像。
- **dataDir**
   * 选填参数。指定在实验中HDFS是存储下载数据的存放路径。
该路径设置格式如下：
```
hdfs://{your HDFS host}:9000/{your data directory}
```
- **outputDir**
   * 选填参数。指定实验文件的HDFS输出路径，一旦实验完成（无论成功或失败），NNI SDK自动将stdout和stderr拷贝HDFS中，该路径设置格式如下：
```
hdfs://{your HDFS host}:9000/{your output directory}
```
**useAnnotation**: True， 实验示例使用了python注释。

**第四步：保存配置文件（如exp_pai.yaml），并执行以下命令：**
```
nnictl create --config exp_pai.yaml
```
关于NNI命令行工具详情参阅[此处](https://github.com/Microsoft/nni/blob/master/docs/NNICTLDOC.md)。

# 查看实验结果
****

PAI模式中，NNI为每个实验都创建了作业(Job), Job命名方式为： nni_exp_{experiment_id}_trial_{trial_id}，如下图所示：
![](https://github.com/Microsoft/nni/blob/master/docs/nni_pai_joblist.jpg?raw=true "joblist")

>**注意**: PAI 模式中，NNI管理端将启动监听在51189端口的rest服务，为了防止链接时刻的拥塞，请确保防火墙规则已开启了51189端口的TCP协议。

### NNI WebUI
NNI提供了WebUI以便查看实验进度，控制实验过程以及一些其他的特色功能。通过nnictl create命令打开默认WebUI服务。

一旦作业完成, 你可以前往NNI WebUI页面浏览实验信息，如下图所示:
![](https://github.com/Microsoft/nni/blob/master/docs/nni_webui_joblist.jpg?raw=true "WebUI")

展开具体的实验详情,点击logPath链接，跳转到HDFS web端口，浏览保存在HDFS中的实验输出文件，如下图所示：
![](https://github.com/Microsoft/nni/blob/master/docs/nni_trial_hdfs_output.jpg?raw=true "HDFS")

如图所示：HDFS中有三种输出文件：stderr, stdout和trial.log。

如果你想将实验中其他的文件输出到HDFS中，比如模型文件，你可以在实验代码中使用环境变量`NNI_OUTPUT_DIR`保存输出文件路径，NNI SDK将拷贝实验容器中`NNI_OUTPUT_DIR`目录下的所有文件到HDFS中。

## 贡献
****
NNI乐于构建交流社区，我们欢迎各位通过[NNI github repo](https://github.com/Microsoft/nni)或者<nni@microsoft.com>分享使用NNI过程中的各种想法与问题。

分享方式指南见[此处]()。

## 协议
****
MIT 协议

## 附录
****

您好，附录部分概述了我对该英文页面做出的些许修改，希望对该项目的技术文档有帮助。

### 语言

1. 英文的语言已经很专业了，部分存在口语表述，比如 let's say,为了是文档更加简洁专业，口语化表达尽量避免。
2. 拼写错误

   Eg: plesae issues on NNI github repo.(Please)
    
    You can see there are three fils in output folder.(files)

### 结构

1. 英文文档只有两个标题，第一个标题setup environment 下过于简洁，需要点击here进入才能知道怎么安装NNI, 而且进入之后还有一个quick start，此处的quick start内容与第二个标题run an experiment内容有交叉，用户容易产生疑惑，即我完成了installation之后接下来是继续quick start还是返回上一层继续run an experiment?

   修改看法：建议直接把相关安装的关键步骤放在本级标题下,Dependencies和NNI Installation 作为二级标题，最后加上“关于NNI安装和快速入门请参考此处”，详略结合，修改结构见上文。
   
2. “运行实验”这部分，英文文档内容不够清晰，仅仅是文字加图片的堆积，一眼看不出结构逻辑。

   修改看法：我参考quick start内容，并总结此页面内容，最后以“第一步，第二步，第三步，第四步“呈现实验步骤。

3. 英文文档只有“setup environment和 run an experiment”，虽然后面有写出WebUI页面查看实验结果，但是非常不清晰。


   修改看法：增加”View experiment results“，将其作为一级标题，与setup environment和 run an experiment并列，因为我觉得“创建环境，运行实验，查看结果”这样才完整。另外，一级标题下增加了二级标题”NNI WebUI“，我认为这是WebUI是查看结果的关键,属于重要内容,英文文档中也有图片,可以把它单独做个二级标题,直接清晰.

### 图文信息对应

1. 原文档如下所示：

  Once a trial job is completed, you can goto NNI WebUI's overview page (like http://localhost:8080/oview) to check trial's information.

  Expand a trial information in trial list view, click the logPath link like: 

![](https://github.com/Microsoft/nni/blob/master/docs/nni_webui_joblist.jpg?raw=true "WebUI")

And you will be redirected to HDFS web portal to browse the output files of that trial in HDFS: 

![](https://github.com/Microsoft/nni/blob/master/docs/nni_trial_hdfs_output.jpg?raw=true "HDFS")

  问题所在：我理解的是先看到NNI那个实验信息结果页面，然后才点击下方的logPath链接，跳转到HDFS页面的，原文英文文档容易误导用户需要先点击logPath才能看到WebUI结果页面.

  修改看法:应该把expand a trial information in trial list view, click the logPath link 和and you will be redirected to HDFS web portal to browse the output files of that trial in HDFS.这两句话放一块,处于两张图片之间.

### 名词解释

- 增加一些关键的名词解释,且加粗表示. 

  如配置文件中的tuner各项: builtintunerName, classArgs, optimization_mode等。这样用户在配置文件时就能清楚看到这些怎么填入了。其实这些词在原NNI安装页面的quick start里有解释，但是所有词都在一段，也没加粗，很不显眼，所以我把关键名词以并列，加粗的形式直接写在了配置文件那里，我觉得更加清晰，直观。

### 命令行

- 实验步骤中多点单独的命令行最好，不要嵌入文字中表述，如：

  NNI provides a demo configure file for each trial example, cat ~/nni/examples/trials/mnist-annotation/config.yml to see it.

### Note & Notice

- 原文中Note和Notice的内容与文档文字相同，也没加粗，非常不明显，起不到让人注意的效果。

  修改看法： Note & Notice 都要加粗表示，该部分内容以块的形式呈现更好，更能引人注意，如果Note要注意的内容与整个标题下的内容有关，应该放在标题下，且Note和标题之间需要少许文字描述。

### Contribution

- 文档中添加contribution，虽然原文档最后也有说欢迎用户分享问题，但是我觉得加个Contribution更加完整，另外建议文档可以提供详细的contribution guidelines，方便用户提供更好的修改完善想法，也便于我们维护github上的版本等。

  contribution guidelines可参考：
  
  <https://github.com/tensorflow/tensorflow>
   <https://github.com/apache/kafka/blob/trunk/README.md>

### License

- 参考其他开源项目的readme文档，如<https://github.com/tensorflow/tensorflow> 文档末尾最好补充协议名称，这样更加完整。因此，我最后加了License: MIT License