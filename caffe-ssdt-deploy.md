## 
## MindStudio部署
通过ModelArts的训练和转换，我们得到了om模型，下面通过Mind Studio部署的过程使模型在Atlas200DK上完成推理，并展示结果。
### 第一步：新建工程
在mindstudio上新建一个object-detection的工程
1、打开mindstudio，选择File->New->New Project

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
