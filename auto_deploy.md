请参考https://github.com/shanchenqi/object_detection/blob/master/README_cn.md
完成代码的部署，下面将和大家分享关键代码及其逻辑结构


## 代码讲解
1.objectdetectionapp中的graph.config文件：
在模型的部署过程中，包括三个engine，这里我们称之为引擎：摄像头获取图片的引擎Mind_camera_datasets，推理的前处理引擎object_detection_inference，推理的后处理引擎object_detection_post_process。在graph.config配置文件中，含有这三个引擎的信息:引擎的id，引擎的名称，引擎的配置信息等。graph.config配置文件中还包括将这三个引擎串联起来需要的两个链接信息（connects），含有输入输出节点的ID等信息。
    
    graphs {
      graph_id: 1875
      priority: 0
     #一共有三个engines
    engines {
        id: 958
        engine_name: "Mind_camera_datasets"
        side: HOST
        ai_config {
           ******some items
            }

    engines {
        id: 244
        engine_name: "object_detection_inference"
        side: DEVICE
        ai_config {
        items {
            name: "model_path"
            value: "../MyModel/object_detection.om"
        }
        }
    }

    engines {
        id: 601
        engine_name: "object_detection_post_process"
        side: HOST

        ai_config {
        items {
            name: "output_name"
            value: "poselayer"
        }

        ******some items
        }
    }
       #connects将engines连接起来
    connects {
        src_engine_id: 958
        src_port_id: 0
        target_engine_id: 244
        target_port_id: 0
    }

    connects {
        src_engine_id: 244
        src_port_id: 0
        target_engine_id: 601
        target_port_id: 0
    }
    }			


2.objectdetectionapp中的main.py:
在main.py文件中将上述图文件中的各个引擎串联并通过objectDetectGraph.StartupEngines(engineList)启动。可在原程序代码中查看注释：https://github.com/shanchenqi/object_detection/blob/master/objectdetectionapp/main.py




3.objectdetectionapp中的mind_camera_dataset.py:
通过mind_camera_dataset.py文件将摄像头输入的视频文件转化为YUV格式的数据，后续将其送入推理引擎进行推理。

4.objectdetectionapp中的object_detection_inference.py：
Process函数是程序的入口，
由于模型的输入是300×300,在这里首先对输入的图片数据进行resize的操作，使其符合模型的输入，在这里resize操作将图片处理的宽和高分别设为128和16的倍数，即384×304，使芯片发挥最佳性能。在模型的推理过程中，会通过aipp的操作直接处理resize后的图片，并不会造成图片的大小与模型的输入不相符的情况。

    def ResizeImageOfFrame(self, srcFrame):
        resizedImgList = []
        for i in range(0, srcFrame.batchInfo.batchSize):
            srcImgParam = srcFrame.imageList[i]
            if srcImgParam.imageData.format != IMAGEFORMAT.YUV420SP:
                print("Resize image: input image type does not yuv420sp, notify camera stop")
                SetExitFlag(True)
                return HIAI_APP_ERROR

            resizedImg = ImageData()
            #model resolution, not image resolution
            #通过resolution存储模型的输入图片的size
            modeResolution = Resolution(kModelWidth, kModelHeight)
            ret = ResizeImage(resizedImg, srcImgParam, modeResolution, self.isAligned)
            if ret != HIAI_APP_OK:
                print("Image resize error")
                continue
            resizedImgList.append(resizedImg)
        return resizedImgList
    
ResizeImage()将图片进行resize，方法的定义如下

    def ResizeImage(destImage, srcImageParam, modeResution,  isAligned = True):
        srcImageC = image.CreateImageDataC(srcImageParam.imageId, srcImageParam.imageData)
        modeResolutionC = image.CreateResolutionC(modeResution)
        destImageC = image.ImageDataC()
        ret = hiaiApp.ResizeImage(byref(destImageC), byref(modeResolutionC), byref(srcImageC), isAligned)
        if ret != HIAI_APP_OK:
            print("Resize image failed, return ", ret)
            return HIAI_APP_ERROR
        ##AlignUP()将图片的宽高转为合适的大小，宽为128的倍数，高为16的倍数。
        if isAligned:
            destImage.width = AlignUP(modeResolutionC.width, kVpcWidthAlign)
            destImage.height = AlignUP(modeResolutionC.height, kVpcHeightAlign)
        else:
            destImage.width = modeResolutionC.width
            destImage.height = modeResolutionC.height
        destImage.size = destImageC.size
        destImage.data = destImageC.data

        return HIAI_APP_OK


接下来，ConvImageOfFrameToJpeg方法将视频的YUV数据转化为jpeg格式，通过将EngineTrans类实例化，存储图片的信息发送给后处理引擎，将模型的推理结果展现在presenterserver中。

    def ConvertImageYuvToJpeg(destImageParam, srcImageParam):
            srcImageC = image.CreateImageDataC(srcImageParam.imageId, srcImageParam.imageData)
        destImageC = image.ImageDataC()
        #hiaiApp.ConvertImageToJpeg调用Atlas200DK上的hiaiApp模块，完成图片的转化
        
            ret = hiaiApp.ConvertImageToJpeg(byref(destImageC), byref(srcImageC))
            if ret != HIAI_APP_OK:
                print("Convert image failed!")
                return HIAI_APP_ERROR

            destImageParam.imageId = srcImageParam.imageId
            destImageParam.imageData.width = srcImageParam.imageData.width
            destImageParam.imageData.height = srcImageParam.imageData.height
            destImageParam.imageData.size = destImageC.size
            destImageParam.imageData.data = destImageC.data
最后调用ExcuteInference方法完成图片的推理，再推理的过程中，通过YUV2Array函数将原始的图片转化为一个narray，通过Atlas200DK的hiai模块，完成图片的推理，将推理结果和原始图片共同发送到最后的后处理引擎中。具体的代码如下：
     
     def ExcuteInference(self, images):
        result = []
        for i in range(0, len(images)):
        #将YUV格式的数据转化为narray
            nArray = Yuv2Array(images[i])
            ssd = {"name": "object_detection", "path": self.modelPath}
        #将narray转化为tensor作为模型的输入   
            nntensor = hiai.NNTensor(nArray)
            tensorList = hiai.NNTensorList(nntensor)
            #self.first==True，此时的推理开启aipp完成。
            if self.first == True:
                
                self.graph = hiai.Graph(hiai.GraphConfig(graph_id=2001))
                with self.graph.as_default():
                    self.engine_config = hiai.EngineConfig(engine_name="HIAIDvppInferenceEngine",
                                                  side=hiai.HiaiPythonSide.Device,
                                                  internal_so_name='/lib/libhiai_python_device2.7.so',
                                                  engine_id=2001)
                    self.engine = hiai.Engine(self.engine_config)
                    self.ai_model_desc = hiai.AIModelDescription(name=ssd['name'], path=ssd['path'])
                    self.ai_config = hiai.AIConfig(hiai.AIConfigItem("Inference", "item_value_2"))
                   # 在此处完成推理流程的编排
                    final_result = self.engine.inference(input_tensor_list=tensorList,
                                                ai_model=self.ai_model_desc,
                                                ai_config=self.ai_config)
                ret = copy.deepcopy(self.graph.create_graph())
                if ret != hiai.HiaiPythonStatust.HIAI_PYTHON_OK:
                    print("create graph failed, ret", ret)
                    d_ret = graph.destroy()
                    SetExitFlag(True)
                    return HIAI_APP_ERROR, None 
                self.first = False
            else:
                with self.graph.as_default():
                    final_result = self.engine.inference(input_tensor_list=tensorList,
                                                ai_model=self.ai_model_desc,
                                                ai_config=self.ai_config)
            在这里调用Graph.proc进行推理得到的结果。
            resTensorList = self.graph.proc(input_nntensorlist=tensorList)
            result.append(resTensorList)
        return HIAI_APP_OK, result

5.objectdetectionapp中的object_post_process.py:
在后处理引擎中,首先判断是否有推理结果，如果没有推理结果，则调用HandleOriginalImage方法直接将原始图片发送到服务中。如果有推理结果，则调用HandelResults方法处理推理结果。模型的推理结果是一个size为(200,1,1,7)的tensor，其中的200表示模型会将推理结果中的top200输出，7维数据中的result[1]表示推理结果的标签id，result[2]表示置信度，result[3-6]表示识别出物体的box的坐标，将这些结果处理好发送到presenter server中进行结果的展示。


    def HandleResults(self, inferenceData):
        jpegImageParam = inferenceData.imageParamList[0]
        inferenceResult = inferenceData.outputData[0][0]
        widthWithScale = round(inferenceData.widthScale * jpegImageParam.imageData.width, 3)
        #在这里由于显示器的显示宽（1024像素）与实际图片（1280像素）存在差异，所以此处应做相应修改。
        widthWithScale=widthWithScale/1280*1024
        heightWithScale = round(inferenceData.heightScale * jpegImageParam.imageData.height, 3)

        detectionResults = []
        for i in range(0, inferenceResult.shape[0]):
            for j in range(0, inferenceResult.shape[1]):
                for k in range(0, inferenceResult.shape[2]):
                    result = inferenceResult[i][j][k]
                    attr = result[kAttributeIndex]   #1
                    score = result[kScoreIndex] #object detection confidence  #2
                    
                    #Get the object position in the image
                    one_result = DetectionResult()
                    one_result.lt.x = result[kAnchorLeftTopAxisIndexX] * widthWithScale   #3
                    one_result.lt.y = result[kAnchorLeftTopAxisIndexY] * heightWithScale   #4
                    one_result.rb.x = result[kAnchorRightBottomAxisIndexX] * widthWithScale  #5
                    one_result.rb.y = result[kAnchorRightBottomAxisIndexY] * heightWithScale  #6
                    if self.IsInvalidResults(attr, score, one_result.lt, one_result.rb):
                        continue
                    print("score=%f, lt.x=%d, lt.y=%d, rb.x=%d rb.y=%d",
                          score, one_result.lt.x, one_result.lt.y,one_result.rb.x, one_result.rb.y)

                    score_percent = score * kScorePercent
                    #Construct the object detection confidence string
                    one_result.result_text = kobjectLabelTextPrefix + str(score_percent) + kobjectLabelTextSuffix
                    detectionResults.append(one_result)
        #Send the object position, confidence string and image to presenter server
        ret = SendImage(self.channel, jpegImageParam.imageId, jpegImageParam.imageData, detectionResults)