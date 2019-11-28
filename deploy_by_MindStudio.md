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



### 第二步、转换模型
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
### 第三步、搭建流程
通过双击新创建的后缀名为.mind的文件（例如object-detection.mind）即可打开Engine流程编排窗口，如图所示。
     ![mind](https://github.com/shanchenqi/atlas200DK/blob/master/picture/mind.png)

object-detection网络主要包含如下节点：一个数据集、一个模型、一个数据预处理、一个执行引擎以及一个图片后处理节点。流程编排具体操作步骤如下：
1. 添加模型。

在右侧工具栏中选择“ Model > My Models”选择刚刚转换的物体检测模型

详细的模型转化配置参数请参见新增自定义模型组件章节。

2. 添加数据集节点。
本示例使用Mind Studio内置的数据集“Datasets > BuiltIn Datasets > Pascal100”。

如果用户想使用自定义数据集，请在右侧工具栏中选择“Datasets > My Datasets”右侧的“[+]”，在弹出的Import Dataset窗口供填写Dataset Name，选择Data Type及Data Source，导入用户自定义数据集。

3. 将所需节点放置到对应位置。
在右侧Tool工具栏中，点击“Datasets”项，展开数据集列表，并展开其子项“My Datasets”。
选择Pascal100控件，在该控件上长按鼠标左键，将其拖动进入左侧绘制区域，然后松开鼠标左键完成放置。
按同样的方式放置其他节点：

a. Model > My Models下的object-detection节点。

b. reprocess下的ImagePreProcess节点。

c. Deep-Learning Execution Engine下的MindInferenceEngine节点。

d.Postprocess下的SSDPostProcess节点。

    ![mind](https://github.com/shanchenqi/atlas200DK/blob/master/picture/create-project3.png)
4. 配置节点属性
a. 点击“ImagePreProcess”节点。
b. 在右侧“property”中打开Resize开关。
c. 将“resize_width”与“resize_height”都设置为300（默认即为开启，宽与高为224\*224），Resize的大小设置应与模型的输入要求保持一致，模型要求大小可以通过模型prototxt文件的“input_param”参数获得，对于本实验ssd模型，输入数据格式要求为300\*300
5. 建立节点连线关系
在完成所需节点的放置与属性设置后，需要建立其相应的连接关系。
 ![mind](https://github.com/shanchenqi/atlas200DK/blob/master/picture/creat-project3.png)

橘黄色的圆形端点为输出端点，可以从该点引出连线，绿色端点为输入端点，可以放置连线。
### 第四步、编译运行
1. 在完成网络结构的编辑之后，点击左下方的“Generate”来生成对应的源码与执行脚本
2. 单击画布下方的“Run”，配置硬件平台的IP地址。若使用USB连接Atlas200DK的IP地址为192.168.1.2
### 第五步、结果查看
鼠标右键点击画布中的SSDPostProcess节点，选择Image Result
![mind](https://github.com/shanchenqi/atlas200DK/blob/master/picture/result.png)

