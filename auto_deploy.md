## 自动部署
首先，通过mind_camera_datsets模块从摄像头获取YUV420SP格式的数据，接下来通过object_detection_inference.py完成图像的预处理和推理，通过object_dectection_post_process.py处理推理结果，并将推理结果发送到presenter server。
* 获取源码
将https://gitee.com/Atlas200DK/sample-facedetection-python.git仓中的代码下载至所在Ubuntu服务器（UIHost）的任意目录，例如代码存放路径为：$HOME/ascend/sample- facedetection-python。

* 安装环境依赖
进入sample- facedetection-python/scrip目录，然后切换到root用户下，在终端执行

bash install.sh <开发板ip地址> <外网ip> <内网ip>

开发板ip地址： 开发板网口地址。使用usb连接时默认为192.168.1.2
外网ip：UIHost连接Internet的网口IP
内网ip：UIHost上和开发板连接的网口IP。在UIHost上使用ifconfig可以查看
install.sh脚本执行的操作有：

1.安装presenterserver依赖的python包；

2.配置开发板和Ubuntu服务器网络，使开发板可以连接internet。Ubuntu服务器和开发板网络配置都需要在root账户下执行，所以在Ubuntu服务器上要切换到root账户执行install.sh脚本。并且在开发板侧install.sh脚本也会切换到root账户执行配置命令，切换时会提示用户输入开发板root账户密码，默认密码为"Mind@123"；

3.升级和更新开发板的linux系统。为了安装依赖的python包，install.sh脚本会自动在开发板上执行命令apt-get update和apt-get upgrade。根据网络以及开发板是否已经执行过更新等状况，该步骤的执行时间可能会超过20分钟，并且期间安装询问交互，选Y即可；

4.安装模型推理python包hiai,以及依赖的python-dev、numpy、pip、esasy_install、enum34、funcsigs、future等python包。因为numpy采用编译安装的方式，编译时间较长，所以安装时间会超过10分钟。并且安装过程中会出现安装询问交互，输入Y即可。
注意：安装环境依赖只需要在 （1）第一次运行Object Detection样例 （2）采用全新制卡方式升级开发板后运行Object Detection样例 两种场景下执行。已经成功安装后不需要执行。
## 部署
## 代码讲解
1.objectdetectionapp中的graph.config文件：
在模型的部署过程中，包括三个engine，这里我们称之为引擎：摄像头获取图片的引擎Mind_camera_datasets，推理的前处理引擎object_detection_inference，推理的后处理引擎object_detection_post_process。在graph.config配置文件中，含有这三个引擎的信息:引擎的id，引擎的名称，引擎的配置信息等。graph.config配置文件中还包括将这三个引擎串联起来需要的两个链接信息（connects），含有输入输出节点的ID等信息。
![train](https://github.com/shanchenqi/atlas200DK/blob/master/picture/graphconfig.png)

2.objectdetectionapp中的object_detection_inference.py:
通过mind_camera_dataset.py文件将摄像头输入的视频文件转化为YUV格式的数据，后续将其送入推理引擎进行推理。

3.objectdetectionapp中的object_detection_inference.py：
Process函数是程序的入口，
由于模型的输入是300×300,在这里首先对输入的图片数据进行resize的操作，使其符合模型的输入，在这里resize操作将图片处理的宽和高分别设为128和16的倍数，即384×304，使芯片发挥最佳性能。在模型的推理过程中，会通过aipp的操作直接处理resize后的图片，并不会造成图片的大小与模型的输入不相符的情况。
接下来，ConvImageOfFrameToJpeg方法将视频的YUV数据转化为jpeg格式，通过将EngineTrans类实例化，存储图片的信息发送给后处理引擎，将模型的推理结果展现在presenterserver中。
最后调用ExcuteInference方法完成图片的推理，再推理的过程中，通过YUV2Array函数将原始的图片转化为一个narray，通过Atlas200DK的hiai模块，完成图片的推理，将推理结果和原始图片共同发送到最后的后处理引擎中。

4.objectdetectionapp中的object_post_process.py:
在后处理引擎中,首先判断是否有推理结果，如果没有推理结果，则调用HandleOoriginalImage方法直接将原始图片发送到服务中。如果有推理结果，则调用HandelResults方法处理推理结果。模型的推理结果是一个size为(200,1,1,7)的tensor，其中的200表示模型会将推理结果中的top200输出，7维数据中的result[1]表示推理结果的标签id，result[2]表示置信度，result[3-6]表示识别出物体的box的坐标，将这些结果处理好发送到presenter server中进行结果的展示。
