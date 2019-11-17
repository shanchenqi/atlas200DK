## 
## MindStudio部署
通过ModelArts的训练和转换，我们得到了om模型，下面通过Mind Studio部署的过程使模型在Atlas200DK上完成推理，并展示结果。
### 第一步：新建工程
在mindstudio上新建一个object-detection的工程
1. 打开mindstudio，选择File->New->New Project

![new_project](https://github.com/shanchenqi/atlas200DK/blob/master/picture/new_project.png)

2. 输入工程名，修改Tatget为Atlas DK。并点击Create

![creat_project](https://github.com/shanchenqi/atlas200DK/blob/master/picture/create.png)

3. 单击“Create”新建一个Mind工程，DEFAULT模式下新建的Mind工程下会自动生成跟工程名相同的mind文件，如图所示，该文件不可复制、删除、重命名。

     ![creat_project](https://github.com/shanchenqi/atlas200DK/blob/master/picture/creat-project2.png)



第二步、转换模型
A.使用ModelArts的模型转换功能，将om模型从obs下载到本地后，可以直接加载离线模型。
B.将modelarts上训练得到的caffe模型转换为mindstudio支持的om模型
1. 选择Tool->Convert Model，进入模型转换界面

     ![convert_model](https://github.com/shanchenqi/atlas200DK/blob/master/picture/convert_model.png)

2. 模型转换

     Model Type选择 Caffe;
     Model File选择训练生成的模型文件，例如：deploy.txt
     Weight File选择训练生成的权重文件，例如：VGG_VOC0712_SSD_300x300_iter_50.caffemodel
    ![convert_model](https://github.com/shanchenqi/atlas200DK/blob/master/picture/convert_model2.png)

3. 单击OK开始模型转换。
   模型在转换的时候，会有报错。报错信息如下所示。
    
     ![convert_model](https://github.com/shanchenqi/atlas200DK/blob/master/picture/convert_model3.png)

   此时在DetectionOutput层的Suggestion中选择SSDDetectionOutput，并点击Retry，转换成功后如下图所示。

     ![convert_model](https://github.com/shanchenqi/atlas200DK/blob/master/picture/convert_model4.png)
   模型转换成功后，后缀为.om的Davinci模型存放地址为$HOME/tools/che/model-zoo/my-model/xxx。
第三步、搭建流程
通过双击新创建的后缀名为.mind的文件（例如object-detection.mind）即可打开Engine流程编排窗口，如图所示。
     ![mind](https://github.com/shanchenqi/atlas200DK/blob/master/picture/mind.png)

object-detection网络主要包含如下节点：一个数据集、一个模型、一个数据预处理、一个执行引擎以及一个图片后处理节点。流程编排具体操作步骤如下：
1. 添加模型。

在右侧工具栏中选择“ Model > My Models”选择刚刚转换的物体检测模型

详细的模型转化配置参数请参见新增自定义模型组件章节。

添加数据集节点。
本示例使用Mind Studio内置的数据集“Datasets > BuiltIn Datasets > ImageNet100”。

如果用户想使用自定义数据集，请在右侧工具栏中选择“Datasets > My Datasets”右侧的“[+]”，在弹出的Import Dataset窗口供填写Dataset Name，选择Data Type及Data Source，导入用户自定义数据集。

详细的添加数据集参数请参见数据集导入章节。

将所需节点放置到对应位置。
在右侧Tool工具栏中，点击“Datasets”项，展开数据集列表，并展开其子项“My Datasets”。
选择ImageNet控件，在该控件上长按鼠标左键，将其拖动进入左侧绘制区域，然后松开鼠标左键完成放置，如图3所示。
图3 放置节点操作


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
