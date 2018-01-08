---
layout:     post
title:      在 Raspberry Pi 上安装 Tensorflow
subtitle:   在 Raspberry Pi 上交叉编译 Tensorflow
date:       2018-01-08
author:     Noah Zhang
header-img: img/post-bg-2018.jpg
catalog: true
tags:
- Raspberry Pi
- Tensorflow
- 中文
- 翻译
---
# 在 Raspberry Pi 上安装 Tensorflow
我喜欢 Raspberry Pi，因为它是一个软件与物理世界互动的很好的平台。 TensorFlow 可以将来自摄像机和麦克风的杂乱无章的传感器数据转化为有用的信息，因此在 Raspberry Pi 上运行模型能够实现一些有趣的应用，从[预测列车时间](https://svds.com/tensorflow-image-recognition-raspberry-pi/)，[垃圾分类](https://techcrunch.com/2016/09/13/auto-trash-sorts-garbage-automatically-at-the-techcrunch-disrupt-hackathon/)，[改善机器人视觉](https://www.oreilly.com/learning/how-to-build-a-robot-that-sees-with-100-and-tensorflow)甚至[避免交通罚单](https://techcrunch.com/2016/09/11/not-today-satan/)！  
  
在 Raspberry Pi 上安装 TensorFlow 并不容易。 我已经创建了一个 makefile 脚本，可以使你[从头开始构建 C++ 部分](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/contrib/makefile#raspberry-pi)，但是需要花了几个小时才能完成，并且不支持 Python。 外部贡献者 Sam Abrahams 在维护用于[主要版本的 Python pip wheel](https://github.com/samjabrahams/tensorflow-on-raspberry-pi) 方面做出了杰出成绩，但是构建它需要你为 Raspberry Pi 添加 USB 设备的来用作交换空间，并且花费比 makefile 方法更长的时间。 Snips 设法为 [Rust 进行 TensorFlow 交叉编译](https://medium.com/snips-ai/how-we-made-tensorflow-run-on-a-raspberry-pi-using-rust-7478f7a31329)，但是不清楚如何将其应用于其他语言。  
  
团队中有很多人都是树莓派的爱好者，并且[尤金·布雷沃](https://ebrevdo.github.io/)开始潜心研究如何改善这种状况。 我们知道我们希望有一些东西可以作为 [TensorFlow 的 Jenkins 持续集成系统](https://ci.tensorflow.org/)的一部分来运行，这意味着要构建一个完全自动的解决方案，无需用户介入即可运行。 由于一个 Pi 插入机器来运行 makefile 构建的东西很难维护，我们尝试使用 [Mythic Beasts](https://www.mythic-beasts.com/) 的托管服务器。 尤金在经历一些小问题后生成了 makefile，但 Python 版本需要更多的内存，但我们不可能去插入一个 USB 设备！  
  
就树莓派而言，在 x86 Linux 机器上构建交叉编译，看起来更易于维护，但也更复杂。 幸运的是，我们有 Snips 的例子给我们一些指点，[我试了下一个善良的陌生人提供的解决方案，是个上次个困住我好久的崩溃](https://raspberrypi.stackexchange.com/questions/48225/whats-causing-these-crashes-after-cross-compiling)，另外尤金设法得到一个可行的初始版本。  
  
为了可以完整运行，我能[使它工作的东西](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/tools/ci_build/pi/build_raspberry_pi.sh)，提取进了一个 [Docker 容器](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/tools/ci_build/Dockerfile.pi)，现在我们已经将 [nightly builds](http://ci.tensorflow.org/view/Nightly/job/nightly-pi/) 作为 Jenkins 项目的一部分来运行。  
如果只是想尝试 Python 2.7，运行：  
```sh
sudo apt-get install libblas-dev liblapack-dev python-dev \
libatlas-base-dev gfortran python-setuptools
sudo ​pip2 install \
http://ci.tensorflow.org/view/Nightly/job/nightly-pi/lastSuccessfulBuild/artifact/output-artifacts/tensorflow-1.4.0-cp27-none-any.whl
```
这可能需要一段时间才能完成，主要因为它比如像是 SciPy 编译会非常慢。 一旦完成，你将能够在 Python 2 中运行 TensorFlow。如果在该 URL 处找不到有关.whl 文件的错误，则可能版本号已更改。 要想找到正确的名称，请访问<http://ci.tensorflow.org/view/Nightly/job/nightly-pi/lastSuccessfulBuild/artifact/output-artifacts/>，你应该看到列出的新版本。   
  
对于 Python 3.4 的支持，你需要使用不同的 wheel 和 pip 来代替 pip2，如下所示：
```sh
sudo apt-get install libblas-dev liblapack-dev python-dev \
 libatlas-base-dev gfortran python-setuptools
sudo ​pip install \
 http://ci.tensorflow.org/view/Nightly/job/nightly-pi-python3/lastSuccessfulBuild/artifact/output-artifacts/tensorflow-1.4.0-cp34-none-any.whl
```

如果你正在运行 Python 3.5，则可以使用同一个 wheel，但是文件名称会稍有变化，因为它会对版本进行编码。 每次导入 Tensorflow 时都会看到一些警告，但是它会正常工作。
```sh
sudo apt-get install libblas-dev liblapack-dev python-dev \
 libatlas-base-dev gfortran python-setuptools
curl -O http://ci.tensorflow.org/view/Nightly/job/nightly-pi-python3/lastSuccessfulBuild/artifact/output-artifacts/tensorflow-1.4.0-cp34-none-any.whl
mv tensorflow-1.4.0-cp34-none-any.whl tensorflow-1.4.0-cp35-none-any.whl
sudo ​pip install tensorflow-1.4.0-cp35-none-any.whl
```

如果你想要在 Raspberry Pi Zero 或 One 上使用 TensorFlow 的，则需要使用不包含 NEON 指令的替代 wheel。 这比上面为 Raspberry Pi 2 和 Raspberry Pi One 版本优化的那个慢很多，所以我不建议你在新设备上使用它。   
以下是 Python 2.7 的命令：
```sh
sudo apt-get install libblas-dev liblapack-dev python-dev \
libatlas-base-dev gfortran python-setuptools
​sudo pip2 install \
http://ci.tensorflow.org/view/Nightly/job/nightly-pi-zero/lastSuccessfulBuild/artifact/output-artifacts/tensorflow-1.4.0rc1-cp27-none-any.whl
```

以下是 Python 3.4 版本的 Raspberry Pi Zero 的命令：
```sh
sudo apt-get install libblas-dev liblapack-dev python-dev \
 libatlas-base-dev gfortran python-setuptools 
sudo ​pip install \
 http://ci.tensorflow.org/view/Nightly/job/nightly-pi-zero-python3/lastSuccessfulBuild/artifact/output-artifacts/tensorflow-1.4.0-cp34-none-any.whl

```

以下是 Python 3.5 版本的 Raspberry Pi Zero 的命令：
```sh
sudo apt-get install libblas-dev liblapack-dev python-dev \
 libatlas-base-dev gfortran python-setuptools
curl -O http://ci.tensorflow.org/view/Nightly/job/nightly-pi-zero-python3/lastSuccessfulBuild/artifact/output-artifacts/tensorflow-1.4.0-cp34-none-any.whl
mv tensorflow-1.4.0-cp34-none-any.whl tensorflow-1.4.0-cp35-none-any.whl
sudo ​pip install tensorflow-1.4.0-cp35-none-any.whl
```

我发现 Raspberry Pi Zeros/Ones 上的 SciPy 汇编非常慢（很多小时），等待它完成是不太可能的。 相反，我发现自己按下```Ctrl -C```取消它在 SciPy 编译中的相关步骤，然后在安装后用```-no-deps```标志重新运行以跳过构建依赖关系。 这是非常冒险的，但是由于需要 SciPy 的目的只是为了测试，所以如果所有其他的依赖完成，你应该在最后有一个可用的 TensorFlow 副本。 如果你想构建你自己的轮子副本，你可以在安装了 Docker 的 Linux 机器上的 TensorFlow 源代码根目录下运行这一行，以便通过 Python 2.7 为 Raspberry Pi 2 或者 Raspberry Pi 3 构建：
```sh
tensorflow/tools/ci_build/ci_build.sh PI tensorflow/tools/ci_build/pi/build_raspberry_pi.sh
```

通过 Python 3.4 构建：
```sh
CI_DOCKER_EXTRA_PARAMS="-e CI_BUILD_PYTHON=python3 -e CROSSTOOL_PYTHON_INCLUDE_PATH=/usr/include/python3.4" tensorflow/tools/ci_build/ci_build.sh PI-PYTHON3 tensorflow/tools/ci_build/pi/build_raspberry_pi.sh
```

通过 Python 2.7 为 Raspberry Pi Zero 构建：
```sh
tensorflow/tools/ci_build/ci_build.sh PI tensorflow/tools/ci_build/pi/build_raspberry_pi.sh PI_ONE
```

通过 Python 3.4 为 Raspberry Pi Zero 构建：
```sh
CI_DOCKER_EXTRA_PARAMS="-e CI_BUILD_PYTHON=python3 -e CROSSTOOL_PYTHON_INCLUDE_PATH=/usr/include/python3.4" tensorflow/tools/ci_build/ci_build.sh PI-PYTHON3 tensorflow/tools/ci_build/pi/build_raspberry_pi.sh PI_ONE
```

这一切仍然都是实验性的，所以如果这些都不适合你，请把错误[反馈](https://github.com/tensorflow/tensorflow/issues)上来。 我希望我们将能够为将来的每个主要版本提供官方稳定的 Pi 二进制文件，就像我们为 Android 和 iOS 所做的一样，所以了解事情的效果对我来说是非常重要的。 我也很高兴听到你在 Pi 上为 TensorFlow 找到的很酷的新应用程序，所以请让我知道你的构建过程！  

#### 参考：<https://petewarden.com/2017/08/20/cross-compiling-tensorflow-for-the-raspberry-pi/>
