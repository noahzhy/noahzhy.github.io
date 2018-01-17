---
layout:     post
title:      通过 OpenCV 和 Python 访问树莓派相机
subtitle:   Python 2.7/Python3.4 + OpenCV2.4.X/OpenCV3.0+
date:       2018-01-17
author:     Noah Zhang
header-img: img/post-bg-2018.jpg
catalog: true
tags:
- Raspberry Pi
- Python
- OpenCV
- 中文
- 翻译
---
# 通过 OpenCV 和 Python 访问树莓派相机
![](https://github.com/noahzhy/noahzhy.github.io/raw/master/img/raspi_still_example_1-1024x768.jpg)  
  
过去一年在[PyImageSearch](https://www.pyimagesearch.com/)博客上出现了许多受欢迎的博文。[使用k-means聚类算法来查找图像中的主色](https://www.pyimagesearch.com/2014/05/26/opencv-python-k-means-color-clustering/)就是其中很受欢迎的一篇。[建立一个移动文件扫描仪](https://www.pyimagesearch.com/2014/09/01/build-kick-ass-mobile-document-scanner-just-5-minutes/)是我最喜欢的文章之一，也是[PyImageSearch](https://www.pyimagesearch.com/)上连续好几个月的最受欢迎的文章。我写的第一篇教程，[霍比特人和直方图](https://www.pyimagesearch.com/2014/01/27/hobbits-and-histograms-a-how-to-guide-to-building-your-first-image-search-engine-in-python/)，讲的是如何实现一个简单的图像搜索引擎，直到今天仍然有很高的点击量。  
  
到目前为止，在[PyImageSearch](https://www.pyimagesearch.com/)上最受欢迎的文章是我的在你的 [Raspberry Pi 2 和 B+上安装 OpenCV 和 Python](https://www.pyimagesearch.com/2015/02/23/install-opencv-and-python-on-your-raspberry-pi-2-and-b/)。从这点可以看出大家对于树莓派的热爱，我也计划继续写一些关于 OpenCV + Raspberry Pi 的文章的原因。  
  
在我出版了 Raspberry Pi + OpenCV 安装教程后，许多读者希望我继续讨论如何使用 Python 和 OpenCV 访问树莓派相机。  
  
在本文中我们将使用 [picamera](https://picamera.readthedocs.org/en/release-1.9/)，它提供了一个纯 Python 接口的相机模块。接下来，我将展示如何使用 picamera 以 OpenCV 格式捕获图像。


**注意**：我们将要参考我的原创教程在 [Raspberry Pi 上安装 OpenCV 和 Python](https://www.pyimagesearch.com/2015/02/23/install-opencv-and-python-on-your-raspberry-pi-2-and-b/)。如果你没有在树莓派上安装和配置好 OpenCV 和 Python，请花时间阅读本教程，并使用 Python + OpenCV 设置你自己的 Raspberry Pi 参考我的文章：[在 Raspberry Pi 上安装OpenCV 和 Python](https://www.pyimagesearch.com/2015/02/23/install-opencv-and-python-on-your-raspberry-pi-2-and-b/)。


**OpenCV 以及 Python 版本：**  
这个示例运行在 **Python 2.7/Python3.4+** 及 **OpenCV2.4.X/OpenCV3.0+** 上


### 第一步：需要的东西
在开始之前，你需要有一个树莓派相机模块。  

我从亚马逊购买了我的 [5MP Raspberry Pi 摄像机模块](http://www.amazon.com/gp/product/B00E1GGE40/ref=as_li_tl?ie=UTF8&camp=1789&creative=390957&creativeASIN=B00E1GGE40&linkCode=as2&tag=trndingcom-20&linkId=XF5KMO3TGBUENU5T)，售价不到30美元。很难相信相机板模块几乎和 Raspberry Pi 本身一样昂贵，但是这表明在过去的5年里有多少硬件已经进步了。我还买了一个[相机外壳](http://www.amazon.com/gp/product/B00IJZJKK4/ref=as_li_tl?ie=UTF8&camp=1789&creative=390957&creativeASIN=B00IJZJKK4&linkCode=as2&tag=trndingcom-20&linkId=PMQZXV7K7MWCPAZ3)，以保护相机，防范于未然。

假如你已经有了相机模块，你需要安装它。安装起来非常简单，并且替代我的教程来安装相机模块，我推荐你去看官方的 Raspberry Pi 相机安装指南：

<iframe width="854" height="480" src="https://www.youtube.com/embed/GImeVqHQzsE" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>  
  
如果安装好的话，应该如下图所示：  

![](https://github.com/noahzhy/noahzhy.github.io/raw/master/img/installing_camera_board-768x1024.jpg)  

### 第二步：启用你的相机模块
现在你的 Raspberry Pi 相机模块已经安装了，你需要启用它。打开终端并执行以下命令：
```sh
$ sudo raspi-config
```

这将会弹出一个如下所示的屏幕：  

![](https://github.com/noahzhy/noahzhy.github.io/raw/master/img/raspi_config-1024x768.jpg)  

使用箭头键向下滚动到选项5：启用相机，按回车键启用相机，然后向下箭头至```Finish```按钮，再次按```Enter```。最后，你需要重新启动你的 Raspberry Pi 才能使配置生效。

### 第三步：测试相机模块
在开始代码运行之前，先进行一次完整性检查，以确保树莓派相机正常工作。  
  
**注意**：请相信我，在开始运行代码之前，你需要运行这个完整性检查。 在运行 OpenCV 代码之前，确保你的相机正常工作，否则当代码不能正常工作时，你可能很容易会浪费时间，因为这只是相机模块本身引起的问题。  
  
总之，为了运行完整性检查，我把我的树莓派连接到了我的电视机上，并将其朝向我的沙发：  
  
![](https://github.com/noahzhy/noahzhy.github.io/raw/master/img/example_setup-768x1024.jpg)  
  
并且在那，我打开终端，并执行以下命令：
```sh
$ raspistill -o output.jpg
```

这个命令是用来激活树莓派相机模块，显示图像预览，并且几秒钟后捕捉图片，并将其保存到当前工作目录中作为```output.jpg```。  
  
下面是我拍摄我的电视屏幕的照片的一个例子（所以我可以记录本教程的过程）比如 Raspberry Pi 拍摄了我的一张照片：  
  
![](https://github.com/noahzhy/noahzhy.github.io/raw/master/img/raspi_still_example_simple-1024x768.jpg)  
  
这里是```output.jpg```的样子：  
  
![](https://github.com/noahzhy/noahzhy.github.io/raw/master/img/raspberry_pi_output-1024x768.jpg)  
  
很明显，我的树莓派相机模块工作正常！现在我们可以继续讨论一些更令人兴奋的东西。

### 第四步：安装 picamera
所以在这一点上，我们知道我们的树莓派相机工作正常。但是，我们如何使用 Python 与 Raspberry Pi 相机模块进行交互呢？  
  
答案是 [picamera](http://picamera.readthedocs.org/en/release-1.9/index.html) 模块。  
  
记得在[以前的教程](https://www.pyimagesearch.com/2015/02/23/install-opencv-and-python-on-your-raspberry-pi-2-and-b/)中，我们如何利用 ```virtualenv``` 和 ```virtualenvwrapper``` 从系统的 Python 包中完整安装我们的 Python 包。  
  
那么，我们将在这里做同样的操作。  
  
在安装 ```picamera``` 之前，确保激活我们的 ```cv``` 虚拟环境：
```sh
$ source ~/.profile
$ workon cv
```
通过 source 我们的 ```.profile``` 文件，确保有正确的虚拟环境设置的路径。从中我们可以访问我们的虚拟环境。  
  
**注意**：如果要在系统范围内安装 ```picamera``` 模块，则可以跳过以前的命令。 但是，如果您从[之前的教程](https://www.pyimagesearch.com/2015/02/23/install-opencv-and-python-on-your-raspberry-pi-2-and-b/)开始学习，那么在继续执行下一个命令之前，需要确保你处于 ```cv``` 虚拟环境中。  
  
然后，我们可以利用 pip 来安装 ```picamera```：
```sh
$ pip install "picamera[array]"
```
**注意**：我指定了 ```picamera[array]``` 而不是 ```picamera```。  
  
当操作 ```picamera``` 模块中的方法来和相机交互时，如果我们需要使用 OpenCV，我们需要使用 ```array``` 子模块。在 Python 中，OpenCV 的图像格式为 Numpy arrays——而 ```array``` 子模块允许我从树莓派相机模块中获取 Numpy arrays。  
  
如果你安装完成没有错误，你现在已经安装了 ```picamera``` 模块（支持 NumPy 数组）。

### 第五步：使用 Python 和 OpenCV 访问 Raspberry Pi 的单个图像
现在，开始编写代码！  
  
打开一个新的文件，命名为 ```test_name.py```，添加以下代码：
```python
# import the necessary packages
from picamera.array import PiRGBArray
from picamera import PiCamera
import time
import cv2
 
# initialize the camera and grab a reference to the raw camera capture
camera = PiCamera()
rawCapture = PiRGBArray(camera)
 
# allow the camera to warmup
time.sleep(0.1)
 
# grab an image from the camera
camera.capture(rawCapture, format="bgr")
image = rawCapture.array
 
# display the image on screen and wait for a keypress
cv2.imshow("Image", image)
cv2.waitKey(0)
```
第2-5行代码表示导入相关的包。  
  
第8行代码表示初始化 PiCamera 对象，第9行代码表示获取对原始捕获组件的引用。
这个 ```rawCapture``` 对象特别有用，因为：  
1. 可以直接访问相机流。  
2. 避免了压缩流为 JPEG 格式的时间，因为我们最终得到的是 OpenCV 图像格式。  
我强烈推荐在树莓派相机上使用 ```PiRGBArray``` 函数——因为它的性能值得一试。  
  
接下来在第12行，我们 ```sleep``` 2秒——为了预热相机模块，必须这样做。  
  
第15行我们从相机上捕获了一张图片，保存在 ```rawCapture``` 对象上，并且明确图像颜色排列为 BGR 而不是 RGB——这是因为 OpenCV 的图像在 Python 中的格式为 BGR 而不是 RGB。这很重要，不注意这个很可能会产生错误的结果。  
  
最后，第19和第20行展示捕获了的图片。  
  
要执行这个例子的话，打开终端，导航到你的 ```test_image.py``` 文件，然后执行以下命令：
```sh
$ python test_image.py
```
如果一切如预期，应该有一个图像显示在你的屏幕上：  
  
![](https://github.com/noahzhy/noahzhy.github.io/raw/master/img/single_image_example.png)  
  
**注意**：在完成文章的其余部分之后，我决定添加一些这篇博客帖子的内容，所以我没有将我的相机设置面向沙发（我实际上正在调试一些我正在制作的定制家庭监控软件）。 如有任何混淆，很抱歉，但请放心，如果你按照文章中的说明进行操作，一切都将按照文章内容顺利运行！

### 第六步：使用 Python 和 OpenCV 访问 Raspberry Pi 的视频流
好的，现在我们已经学会了如何从 Raspberry Pi 相机中获取单个图像。那么视频流呢？  
  
你可能会猜想，我们将在这里使用 ```cv2.VideoCapture``` 函数——但实际上我建议不要这样做。 把 ```cv2.VideoCapture``` 与 Raspberry Pi 配合使用并不是一个好的体验（你需要安装额外的驱动程序），而你通常应该会避免使用它。 另外，当我们可以使用 ```picamera``` 模块轻松访问原始视频流时，为什么我们还要去使用 ```cv2.VideoCapture``` 函数呢？   
  
让我们继续看看如何访问视频流。打开一个新文件，命名为 ```test_video.py```，然后插入以下代码：
```python
# import the necessary packages
from picamera.array import PiRGBArray
from picamera import PiCamera
import time
import cv2
 
# initialize the camera and grab a reference to the raw camera capture
camera = PiCamera()
camera.resolution = (640, 480)
camera.framerate = 32
rawCapture = PiRGBArray(camera, size=(640, 480))
 
# allow the camera to warmup
time.sleep(0.1)
 
# capture frames from the camera
for frame in camera.capture_continuous(rawCapture, format="bgr", use_video_port=True):
    # grab the raw NumPy array representing the image, then initialize the timestamp
    # and occupied/unoccupied text
    image = frame.array
 
    # show the frame
    cv2.imshow("Frame", image)
    key = cv2.waitKey(1) & 0xFF
 
    # clear the stream in preparation for the next frame
    rawCapture.truncate(0)
 
    # if the `q` key was pressed, break from the loop
    if key == ord("q"):
        break
```
第2-5行代码表示导入相关的包。  
  
第8行构造了 camera 对象，使用它可以和树莓派相机进行交互，第9行我们设置了相机的分辨率（640x480像素），第10行设置了帧率（即每秒帧数，或者FPS）。在第11行初始化了 ```PIRGBArray``` 对象，同时指定了相同的分辨率。


在第17行代码中，我们调用 ```camera``` 对象的函数 ```capture_continuous``` 来访问视频流。该方法返回一**帧**图像，这**帧**图像有一个 ```array``` 属性，表示 Numpy array 格式的图像，所以在第20行 image 得到的就是 Numpy array 格式的图像。所用困难的工作都在第17和第20行完成了！  
  
第23和24行显示了这帧图片。  
  
需要注意的是第27行：在装载下一帧图像之前，必须清空当前帧！  
  
如果你没有清空当前帧，Python 脚本将会抛出一个错误——所以在执行自己的程序时一定要注意这点！  
  
最后，如果使用者点击```q```键，表示退出循环，结束程序。  
  
执行脚本，只需打开终端（当然，确保你在cv虚拟环境中），然后执行以下命令：
```sh
$ python test_video.py
```
下面是我执行上面的命令的一个示例：  
  
<iframe width="854" height="480" src="https://www.youtube.com/embed/HVuEQ7avDqo" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>  
  
正如你所看到的，Raspberry Pi 相机的视频流正在被 OpenCV 读取，然后显示在屏幕上！此外，Raspberry Pi 相机在以 32 FPS 的速度访问画面时不会出现滞后现象。 当然，我们并没有对单个帧进行任何处理，但是正如我在博客文章中所展示的那样，即使在处理每个帧时，Pi 2 也可以很容易地保持 24-32 FPS。

#### 原文：https://www.pyimagesearch.com/2015/03/30/accessing-the-raspberry-pi-camera-with-opencv-and-Python/
