---
layout:     post
title:      用 Python 和 OpenCV 检测和跟踪运动对象
subtitle:   简单介绍如何用 Python 和 OpenCV 检测和跟踪运动对象
date:       2018-02-02
author:     Noah Zhang
header-img: img/post-bg-2018.jpg
catalog: true
tags:
- Raspberry Pi
- Python
- 中文
- 翻译
---
# 用 Python 和 OpenCV 检测和跟踪运动对象
![](https://github.com/noahzhy/noahzhy.github.io/raw/master/img/head.gif)  
这个该死的家伙。我就知道他偷了我最后一罐啤酒！  
  
对于一个男人来讲，这些话永远都不该说。但是当我关上冰箱门的时候，我愤怒地叹息，感到厌恶，自言自语地说了这些。  
  
你看，我花了12个小时写了这篇将要发表的文章《[PyImageSearch Gurus course](http://www.pyimagesearch.com/pyimagesearch-gurus/)》。我的脑子都糊掉了，像个半熟的摊鸡蛋一样，几乎要从耳朵里流出来了。当我深夜决定结束工作的时候，我只想放松一下，看看我最爱的电影——《侏罗纪公园》。同时喝着来自 Smuttynose 的最好的 IPA 冰啤，Smuttynose 是近来我非常喜欢的一家酒厂。  
  
但是，昨天晚上来串门的该死的 James 喝掉了我最后一罐啤酒。  
  
好吧，据称。  
  
**我并不能证明任何我的猜测。**实际上，我并没有亲眼看到他喝我的啤酒，因为我埋头于笔记本电脑中，手指在键盘上跳动，兴奋地敲击出教程和文章。但是我感觉他就是嫌疑犯。他是我唯一会喝 IPA 的（前）朋友。  
  
所以我做了一件任何男人都会做的事。  
  
**我在橱柜顶上安装了一个树莓派，来探测看他是不是打算再次偷啤酒。**   

![](https://github.com/noahzhy/noahzhy.github.io/raw/master/img/hardware.jpg)  

过分了？  
  
也许吧。  
  
但是，我很看重我的啤酒。而且如果 James 再次尝试偷我的啤酒的话，我会逮他个正着。  
  
## 一篇关于运动检测的系列文章（分为两部分）
**做一个用于家庭监控的运动检测和追踪系统**，分两部分，本文是第一篇。  
  
本文接下来的部分，将会详细介绍如何使用计算机视觉技术来建立一个用于家庭监控的基础的运动检测和追踪系统。本例对预先录制的视频和网络摄像头的实时数据流都可以工作；然而，我们将会在我们的笔记本/桌面电脑上进行开发。  
  
在本系列的第二部分中，我会向你展示如何升级代码，使其可以在树莓派和摄像头模块上工作，以及如何扩展家庭监控系统，来捕捉任何检测到的运动，并且上传到你的个人 Dropbox 中。  
  
也许到了最后，我们可以把 James 抓个正着。  
  
## 一点关于背景移除的内容
背景移除是很多计算机视觉应用的关键内容。我们通过它来计算经过收费站的汽车个数。我们通过它来计算进进出出一间商店的人的个数。  
  
同时我们使用它来进行运动检测。  
  
在本文开始写代码之前，让我告诉你，OpenCV 里有很多很多方法来进行运动检测、追踪和分析。有一些非常简单，而另外一些非常复杂。两个初级的方法是某种形式的基于混合高斯模型的前景和背景分割：  
1. KaewTraKulPong 等人发表的《[An improved adaptive background mixture model for real-time tracking with shadow detection](http://www.ee.surrey.ac.uk/CVSSP/Publications/papers/KaewTraKulPong-AVBS01.pdf)》。这个方法可以通过cv2.BackgroundSubtractorMOG来使用。  
2. Zivkovic 提出的《[Improved adaptive Gaussian mixture model for background subtraction](http://www.zoranz.net/Publications/zivkovic2004ICPR.pdf)》和《[Efficient Adaptive Density Estimation per Image Pixel for the Task of Background Subtraction](http://www.zoranz.net/Publications/zivkovicPRL2006.pdf)》。可以通过 cv2.BackgroundSubtractorMOG2 来使用。
在新版本的 OpenCV 中，我们有基于贝叶斯（概率）的前景和背景分割，是 Godbehere 等人在2012年的文章中实现的，《[Visual Tracking of Human Visitors under Variable-Lighting Conditions for a Responsive Audio Art Installation](http://goldberg.berkeley.edu/pubs/acc-2012-visual-tracking-final.pdf)》，我们可以在 cv2.createBackgroundSubtractorGMG 中找到它的实现（然而我们需要等 OpenCV 3的到来，才能使用它的全部功能。）  
  
所有这些方法都涉及到从前景中分离背景（它们甚至提供相应的机制来让我们辨别实际运动和阴影及关照的细微改变）！  
  
为什么这一点特别重要？为什么我们这么在意哪个像素属于前景哪个像素属于背景？  
  
在运动检测中，我们会做出如下的假设：  
  
**我们视频流中的背景在连续的视频帧内，多数时候应该是静止不变的，因此如果我们可以建立背景模型，我们的就可以监视到显著的变化。如果发生了显著的变化，我们就可以检测到它——通常这些变化和我们视频中的运动有关。**  
  
显然在现实世界中，我们这个假设比较容易失效。因为阴影、反色、光照条件以及环境中可能发生的其他变化，我们的背景可能会看上去变得非常不同，这会让我们的算法失效。所以为什么最成功的背景移除/前景检测系统需要固定安装的相机以及控制光照条件。  
  
上面我提到的方法，尽管非常强大，但同时计算非常耗时。而且我们最终的目标是在本系列的最后，把该系统部署在树莓派上，因此我们最好可以坚持使用简单的方法。我们将在未来的文章中回到这些强大的方法上，但是目前我们将保持简单和高效。  
  
## 用 Python 和 OpenCV 进行基础的运动检测和追踪
好了，准备好帮助我开发一个家用监视系统来抓住那个偷啤酒的混蛋了么？打开编辑器，新建一个文件，命名为```motion_detector.py```，然后让我们开始写代码吧。  
```python
# 导入必要的软件包
import argparse
import datetime
import imutils
import time
import cv2
 
# 创建参数解析器并解析参数
ap = argparse.ArgumentParser()
ap.add_argument("-v", "--video", help="path to the video file")
ap.add_argument("-a", "--min-area", type=int, default=500, help="minimum area size")
args = vars(ap.parse_args())
 
# 如果video参数为None，那么我们从摄像头读取数据
if args.get("video", None) is None:
    camera = cv2.VideoCapture(0)
    time.sleep(0.25)
 
# 否则我们读取一个视频文件
else:
    camera = cv2.VideoCapture(args["video"])
 
# 初始化视频流的第一帧
firstFrame = None
```
**2-6行**导入了我们必要的软件包。这些看上去都很熟悉，除了 imutils 这个包，它提供了一组由我编写的非常方便的函数，来让我们更简单的进行图像处理。如果你还没有安装 imutils 到你的系统，你可以通过 pip 来安装：
```sh
pip install imutils
```

下一步，我们在**9-12行**解析了命令行参数。我们定义了两个选项。第一个，```--video```，是可选的。它会指定一个路径，指向一个预先录制好的视频文件，我们可以检测该视频中的运动。如果你不提供视频的路径，那么 OpenCV 会从你的摄像头中来检测运动。  
  
我们同时还定义了```--min-area```，它表示一个图像区域被看做实际运动的最小尺寸（以像素为单位）。正如我接下来要讲的那样，我们会发现图像中比较小的区域变化会比较显著，可能是因为噪点或是光线的变化。在实际中，这些小区域并不是实际的运动——所以我们定义一个最小的尺寸来对付和过滤掉这些假阳性（false-positives）结果。  
  
**15-21行**获取一个我们摄像机对象的引用。在这个例子中，没有提供视频路径（**15-17行**），我们会取得一个摄像头的引用。如果提供了一个视频文件路径，那么我们会在**20-21行**建立一个指向它的指针。  
  
最后，我们以一个变量来结束这段代码，这个变量是```firstFrame```。 能猜到```firstFrame```是什么吗？  
  
**假设**：视频的第一帧不会包含运动，而仅仅是背景——因此我们可以使用第一帧来建立背景模型。 显然我们此处建立的假设有些太大了。但是再说一次，我们的目标是要在树莓派上运行这个系统，所以我们不能做的太复杂。正如你会在本文的结果一节所看到的那样，当有人在屋里走动的时候，我们可以轻易的检测到运动并追踪他们。
```python
# 遍历视频的每一帧
while True:
    # 获取当前帧并初始化occupied/unoccupied文本
    (grabbed, frame) = camera.read()
    text = "Unoccupied"
 
    # 如果不能抓取到一帧，说明我们到了视频的结尾
    if not grabbed:
        break
 
    # 调整该帧的大小，转换为灰阶图像并且对其进行高斯模糊
    frame = imutils.resize(frame, width=500)
    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    gray = cv2.GaussianBlur(gray, (21, 21), 0)
 
    # 如果第一帧是None，对其进行初始化
    if firstFrame is None:
        firstFrame = gray
        continue
```
现在我们已经获取了视频文件/摄像头数据流的引用，我们可以在第一行（原文第27行）开始遍历每一帧了。  
  
调用```camera.read()```为我们返回一个2元组。元组的第一个值是```grabbed```，表明是否成功从缓冲中读取了```frame```。元组的第二个值就是``frame``它本身。  
  
我们同时还定义了一个叫做```text```的字符串，并对其进行初始化来表明我们正在监控的这个房间“没有被占领”（**Unoccupied**）。如果这个房间确实有活动，我们可以更新这个字符串。  
  
在这个例子中，如果没有成功从视频文件中读取一帧，我们会在**10-11行**（原文**35-36行**）跳出循环。  
  
我们可以开始处理帧数据并准备进行运动分析（**15-17行**）。我们首先会调整它的大小到500像素宽——没有必要去直接处理视频流中的大尺寸，原始图像。我们同样会把图片转换为灰阶图像，因为彩色数据对我们的运动检测算法没有影响。最后，我们会使用高斯模糊来平滑我们的图像。  
  
认识到即使是相邻帧，也不是完全相同的这一点很重要！  
  
由于数码相机传感器的微小变化，没有100%相同的两帧数据——一些像素肯定会有不同的强度值。也就是说，我们需要，并应用高斯平滑对一个```11X11```的区域的像素强度进行平均。这能帮我们滤除可能使我们运动检测算法失效的高频噪音。  
  
正如我在上面提到的，我们需要通过某种方式对我们的图像进行背景建模。再一次的，我们会假设视频的第一帧不包含任何运动，它是一个很好的例子，表明我们的背景是如何的。如果```firstFrame```没有初始化，我们会把它保存然后继续处理视频的下一帧。（**20-22行**）  
  
这里有一个关于示例视频第一帧的例子：  

![](https://github.com/noahzhy/noahzhy.github.io/raw/master/img/first_frame_example_02.jpg)  
  
上面这一帧满足我们的假设，视频的第一帧仅仅是一个静止的背景——没有运动。  
  
有了这个静止的背景图片，我们已经准备好实时运动检测和追踪了：  
```python
# 计算当前帧和第一帧的不同
    frameDelta = cv2.absdiff(firstFrame, gray)
    thresh = cv2.threshold(frameDelta, 25, 255, cv2.THRESH_BINARY)[1]
 
    # 扩展阀值图像填充孔洞，然后找到阀值图像上的轮廓
    thresh = cv2.dilate(thresh, None, iterations=2)
    (cnts, _) = cv2.findContours(thresh.copy(), cv2.RETR_EXTERNAL,
        cv2.CHAIN_APPROX_SIMPLE)
 
    # 遍历轮廓
    for c in cnts:
        # if the contour is too small, ignore it
        if cv2.contourArea(c) < args["min_area"]:
            continue
 
        # compute the bounding box for the contour, draw it on the frame,
        # and update the text
        # 计算轮廓的边界框，在当前帧中画出该框
        (x, y, w, h) = cv2.boundingRect(c)
        cv2.rectangle(frame, (x, y), (x + w, y + h), (0, 255, 0), 2)
        text = "Occupied"
```
现在我们已经从firstFrame变量对背景进行了建模，我们可以利用它来计算起始帧和视频流数据中后续新帧之间的不同。  
  
计算两帧的不同是一个简单的减法，我们使用两方相应的像素强度差的绝对值。（第二行）  
```python
delta = |background_model – current_frame|
```
两帧差值图例如下：  

![](https://github.com/noahzhy/noahzhy.github.io/raw/master/img/outdoor.jpg)  

注意到图片的背景是如何变为黑色的。然而，包含运动的区域（比如包含我自己走过房间动作的区域）会更亮一些。这以为这两帧差值大的地方是图片中发生移动的区域。  
  
我们随后在**第3行**对```frameDelta```进行阀值化来显示图片中像素强度值有显著变化的区域。如果差值小于25，我丢弃该像素将其设置为黑色（例如，背景）。如果差值大于25，我们将其设定为白色（例如，前景）。阀值化的差值图片如下：  

![](https://github.com/noahzhy/noahzhy.github.io/raw/master/img/lighter.jpg)  

再一次，注意到图片的背景是黑色的，而前景（运动发生的位置）是白色的。 有了这个阀值化的图片，只要简单的进行实施轮廓检测来找到白色区域的外轮廓线（**第7行**）  
  
我们在**第14行**开始对轮廓线进行遍历，在**15行**滤掉小的，不相关的轮廓。 如果轮廓面积比我们提供的```--min-area```值大，我们会在前景和移动区域画边框线。（**23-25行**）。我们同样会更新text状态字符串来表示这个房间“被占领”（**Occupied**）了。
```python
# draw the text and timestamp on the frame
    # 在当前帧上写文字以及时间戳
    cv2.putText(frame, "Room Status: {}".format(text), (10, 20),
        cv2.FONT_HERSHEY_SIMPLEX, 0.5, (0, 0, 255), 2)
    cv2.putText(frame, datetime.datetime.now().strftime("%A %d %B %Y %I:%M:%S%p"),
        (10, frame.shape[0] - 10), cv2.FONT_HERSHEY_SIMPLEX, 0.35, (0, 0, 255), 1)
 
    # 显示当前帧并记录用户是否按下按键
    cv2.imshow("Security Feed", frame)
    cv2.imshow("Thresh", thresh)
    cv2.imshow("Frame Delta", frameDelta)
    key = cv2.waitKey(1) &amp; 0xFF
 
    # 如果q键被按下，跳出循环
    if key == ord("q"):
        break
 
# 清理摄像机资源并关闭打开的窗口
camera.release()
cv2.destroyAllWindows()
```

**11-13行**显示了我的工作成果，运行我们可以在视频中看到是否检测到了运动，使用帧差值和阀值图像我们可以调试我们的脚本。  
  
**注意**：如果你下载了本文的源代码并打算应用到你自己的视频文件上，你可能需要改变```cv2.threshold```的值和```--min-area```参数来获得你所在光照环境下的最佳效果。  
  
最后，**22行和23行**清理并释放了视频流的指针。  
  
## 结果
显然，我要确定我们的运动监测系统可以在 James 那个偷酒贼再次造访的之前能够正常工作——我们将在本系列第二篇文章中谈到他。为了测试我们使用 Python 和 OpenCV 搭建的运动监测系统，我录制了两个视频文件。  
  
第一个文件是```example_01.mp4```，监视了我公寓的正门，当门被打开时完成检测。第二个文件是```example_02.mp4```使用安装在橱柜上的树莓派录制的。它监控厨房和客厅，当有人在其中走动的时候完成检测。  
  
让我们给我们简单的探测器一次尝试的机会，打开终端并执行下面指令：
```sh
python motion_detector.py --video videos/example_01.mp4
```
下图是一个 gif 图，显示来自探测器的一些静止帧数据。  

![](https://github.com/noahzhy/noahzhy.github.io/raw/master/img/animated_motion_01.gif)  

注意到在门被打开前没有进行运动检测——然后我们可以检测到我自己从门中走过。你可以在这里看到全部视频：  

<iframe width="854" height="480" src="https://www.youtube.com/embed/fi4LORwk8Fc" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

现在，我安装在用于监视厨房和客厅的摄像机表现如何呢？然我们一探究竟。输入下面命令：
```sh
python motion_detector.py --video videos/example_02.mp4
```
来自第二个视频文件的结果样本如下：  

![](https://github.com/noahzhy/noahzhy.github.io/raw/master/img/head.gif)    

同样，这里是我们运动检测结果的完整视频：

<iframe width="854" height="480" src="
https://www.youtube.com/embed/36j238XtcIE" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

正如你看到的，我们的运动检测系统尽管非常简单，但表现还不错！我们可以正常检测到我进入客厅和离开房间。  
  
然而，现实来讲，结果还远远谈不上完美。尽管只有一个人在屋内走动，我们却得到了多个外框——这和理想状态相差甚远。而且我可以看到，微小的光线变化，比如阴影和墙面反射，都触发了假阳性的运动检测结果。  
  
为了解决这些问题，我们依靠 OpenCV 中更加强大的背景移除方法，这些方法对阴影和少量的反射进行了处理。（我将在未来的文章中谈到这些更为先进的背景移除/前景检测方法）  
  
**但是于此同时，请考虑一下我们的最终目标**  
  
这个系统，尽管是在我们的笔记本/台式机系统上开发的，却是为了要部署在树莓派上，树莓派的计算资源非常有限。因此，我们需要让我们的运动检测方法保持简单和快速。我们的运动检测系统并不完美，很不幸这是一个不利的方面，但是对于我们特定的项目，它仍然能够很好的完成工作。  
  
最后，如果你想要利用你的摄像头的原始视频流来进行运动检测，空着```--video```选项即可。
```sh
python motion_detector.py
```

## 小结
通过本文，我们已经认识到我的朋友 James 是一个偷酒贼。真是个混蛋啊！  
  
为了能抓他个人赃并获，我们决定使用 Python 和 OpenCV 建立一个运动检测和追踪系统。这个系统可以获取视频流并分析它们获取运动。考虑到我们所使用的方法，能够得到可以接受的监测结果。  
  
最终目标是要把本系统部署在树莓派上，因此我们没有依赖 OpenCV 中一些比较先进的背景移除方法。相反，我们依赖一个简单，但合理高效的假设——视频的第一帧仅仅包含我们想要建模的背景，而不包括其他任何东西。  
  
在这个假设下，我们可以实施背景移除，检测图片中的运动，在检测到运动的区域画出轮廓框。  
  
在这个关于运动检测系列文章的**第二部分**，我们会**更新代码使其在树莓派上运行**。  
  
我们同样会**集成 Dropbox API**，允许我们监控家用监控系统并且当我们的系统检测到运动时，获取实时更新数据。  
  
敬请期待！

## 全部代码
```python
# 导入必要的软件包
import argparse
import datetime
import imutils
import time
import cv2

# 创建参数解析器并解析参数
ap = argparse.ArgumentParser()
ap.add_argument("-v", "--video", help="path to the video file")
ap.add_argument("-a", "--min-area", type=int, default=500, help="minimum area size")
args = vars(ap.parse_args())

# 如果video参数为None，那么我们从摄像头读取数据
if args.get("video", None) is None:
    camera = cv2.VideoCapture(0)
    time.sleep(0.25)

# 否则我们读取一个视频文件
else:
    camera = cv2.VideoCapture(args["video"])

# 初始化视频流的第一帧
firstFrame = None

# 遍历视频的每一帧
while True:
    # 获取当前帧并初始化occupied/unoccupied文本
    (grabbed, frame) = camera.read()
    text = "Unoccupied"

    # 如果不能抓取到一帧，说明我们到了视频的结尾
    if not grabbed:
        break

    # 调整该帧的大小，转换为灰阶图像并且对其进行高斯模糊
    frame = imutils.resize(frame, width=500)
    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    gray = cv2.GaussianBlur(gray, (21, 21), 0)

    # 如果第一帧是None，对其进行初始化
    if firstFrame is None:
        firstFrame = gray
        continue

    # 计算当前帧和第一帧的不同
    frameDelta = cv2.absdiff(firstFrame, gray)
    thresh = cv2.threshold(frameDelta, 25, 255, cv2.THRESH_BINARY)[1]

    # 扩展阀值图像填充孔洞，然后找到阀值图像上的轮廓
    thresh = cv2.dilate(thresh, None, iterations=2)
    (cnts, _) = cv2.findContours(thresh.copy(), cv2.RETR_EXTERNAL,
        cv2.CHAIN_APPROX_SIMPLE)

    # 遍历轮廓
    for c in cnts:
        # 遍历轮廓
        if cv2.contourArea(c) < args["min_area"]:
            continue

        # 计算轮廓的边界框，在当前帧中画出该框，并且更新标签
        (x, y, w, h) = cv2.boundingRect(c)
        cv2.rectangle(frame, (x, y), (x + w, y + h), (0, 255, 0), 2)
        text = "Occupied"

    # 在当前帧上写文字以及时间戳
    cv2.putText(frame, "Room Status: {}".format(text), (10, 20),
        cv2.FONT_HERSHEY_SIMPLEX, 0.5, (0, 0, 255), 2)
    cv2.putText(frame, datetime.datetime.now().strftime("%A %d %B %Y %I:%M:%S%p"),
        (10, frame.shape[0] - 10), cv2.FONT_HERSHEY_SIMPLEX, 0.35, (0, 0, 255), 1)

    # 显示当前帧并记录用户是否按下按键
    cv2.imshow("Security Feed", frame)
    cv2.imshow("Thresh", thresh)
    cv2.imshow("Frame Delta", frameDelta)
    key = cv2.waitKey(1) & 0xFF

    # 如果q键被按下，跳出循环
    if key == ord("q"):
        break

# 清理摄像机资源并关闭打开的窗口
camera.release()
cv2.destroyAllWindows()
```

#### 原文：<http://www.pyimagesearch.com/2015/05/25/basic-motion-detection-and-tracking-with-python-and-opencv/>
