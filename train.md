### 1. 配置ModelArts访问秘钥
登录[ModelArts](https://console.huaweicloud.com/modelarts/?region=cn-north-1#/manage/dashboard)管理控制台，在“全局配置”界面添加访问秘钥，如下图（如已添加密钥，可跳过此步）：

<img src="https://github.com/huaweicloud/ModelArts-Lab/blob/master/train_inference/image_recognition/images/%E6%B7%BB%E5%8A%A0%E8%AE%BF%E9%97%AE%E7%A7%98%E9%92%A5.PNG" width="800px" />

### 2.修改训练代码
1. 请将obs://3377/中的文件拷贝至自己的桶中,命名为“bucketname”;
2. 将obs://bucketname/
### 3.创建训练作业
在ModelArts“训练作业”界面，单击“创建”按钮，进入创建训练作业页面。按照如下指导填写字段：

![train](https://github.com/shanchenqi/atlas200DK/blob/master/picture/train.png)

名称：自定义

数据来源：选择数据存储位置

数据存储位置：训练数据集的路径，选择OBS路径/bucketname/train-res/data/

算法来源：常用框架

AI引擎：Caffe，Caffe-1.0.0-python2.7

代码目录：训练脚本所在的目录，选择OBS路径/bucketname/scripts/

启动文件：训练脚本，选择OBS路径/bucketname/scripts/train-caffe-ssd.py

运行参数：添加max_epochs=20。运行参数中设置的变量会传入到训练脚本中，经过解析，可以使用。此字段用于设置算法中的超参。

训练输出位置：选择OBS路径。训练输出位置用来保存训练输得到的模型和TensorBoard日志。

资源池：机器的规格，选择“计算型GPU(P100)实例”，表示这台机器包含一张GPU卡。

计算节点个数：选择1，表示我们运行一个单机训练任务。（注：本训练脚本不支持分布式训练）

所有字段填写好之后，确认参数无误，点击下一步，然后点击立即创建，开始训练。训练时长预计8到15分钟左右，ModelArts使用高峰期可能会有时间延迟。

### 3.查看训练作业结果

此时点开日志，可以通过日志完成查看模型的性能。在训练作业完成后，模型存在obs中的输出位置。输出一共四个文件：
1.deploy.prototxt：训练生成的模型文件2.VGG_VOC0712_SSD_300x300_iter_50.caffemodel：训练生成的参数文件
3.aipp.config文件：模型转换的配置文件，会调度ascend芯片的最佳性能
4.opmap文件，在模型转换是存在一个不支持的算子，才此文件中将其进行替代，使模型转换成功。

