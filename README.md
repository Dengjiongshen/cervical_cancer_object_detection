## Cervical Cancer Object Detection
This code is for the competition of ['Digitized Human Body' Visual Challenge - Intelligent Diagnosis of Cervical Cancer Risk](https://tianchi.aliyun.com/competition/entrance/231757/introduction). The purpose of the competition is to provide large-scale thin-layer cell data of cervical cancer labeled by professional doctors. The competitors can propose and comprehensively use methods such as object detection and deep learning to locate abnormal squamous epithelial cells (i.e., ASC) of cervical cancer cytology and classify cervical cancer cells through images, which improve the speed and accuracy of model detection, and assist doctors in real diagnosis.  
![image](https://tianchi-public.oss-cn-hangzhou.aliyuncs.com/public/files/forum/156976273635179161569762735242.jpeg)
Note: Data and kfbreader is not allowed to be published.  

The object detection steps are shown as below:    
### 1. Enviroment Preparation:  
(1) **kfbreader**:  
Since the kfb data need to be loaded by specified SDK (i.e., kfbreader), we have to setup kfbreader provided by the match orgnaisers. The specific tutorial can be visited by the [link](https://tianchi.aliyun.com/forum/postDetail?spm=5176.12586969.1002.3.76de2a3c3k6DZf&postId=83286).  
Remember to add kfbreader to the below paths:
```
export PYTHONPATH=/home/admin/jupyter/kfbreader-linux:$PYTHONPATH
export LD_LIBRARY=/opt/conda/lib:/home/admin/jupyter/kfbreader-linux:$LD_LIBRARY_PATH
```
(2) **detectron2**:
More information, please visit [detectron2 install tutorial](https://github.com/AlvinAi96/cervical_cancer_object_detection/blob/master/detectron2/INSTALL.md)  
Before downloading [fvcore](https://github.com/facebookresearch/fvcore) [cocoapi](https://github.com/cocodataset/cocoapi.git), make sure you have python >= 3.6 and pytorch 1.3.   
Setup detecron2 by running the following commands. (Note: <ROOT> is the root path of detectron2 file path)  
```
# setup fvcore
cd <ROOT>/fvcore-master
python setup.py --user

# setup cocoapi
cd <ROOT>/cocoapi-master/Pythonapi
python setup.py --user

# setup detectron2
cd <ROOT>/detectron2
python setup.py build develop --user
```
	
### 2. Data Preparation: 
(1) put train/test dataset into <ROOT>/Data/Train and <ROOT>/Data/test respectively.  
```
cd <ROOT>
mkdir /Data/Train
mkdir /Data/test	
```
(2) Generate training dataset for model.  
```
cd <ROOT>/Data
python roi_based_data_generation.py

# or
python pos_based_data_generation.py
```
(3) Generate extra rotated images with large bboxes and their labels.  
```
python big_rotate_object.py
```
(4) Transfer prepared datasets from <ROOT>/Data to <ROOT>/detectron2/VOC2007.  
<ROOT>/detectron2/VOC2007 file structure follows the structure of Pascal VOC2007 data file：  
```
VOC2007/
  Annotations/
  	patch0.json
	pathc1.json
	...
  ImageSets/
	Main/
	  train.txt
	  val.txt
    	  trainval.txt
  JPEGImages/
	patch0.jpg
	patch1.jpg
	...	
```
In order to gain the above structure format, run the following commands:  
```
cd <ROOT>/detectron2
mkdir / datasets
mkdir /VOC2007
mkdir /VOC2007/ImageSets
mkdir /VOC2007/ImageSets/Main
mkdir /VOC2007/Annotations
mkdir /VOc2007/JPEGImages
```
(2) 划分数据集并形成train/val/trainval.txt文件至/detectron2/datasets/VOC2007/ImageSets/Main/。  

```
cd <ROOT>
python Data/split_dataset_produce_txt.py
```

(3) 将本地数据集格式转换为PASCAL VOC格式要求，修改后文件为pascal_voc.py，替换原来/detectron2/detectron2/data/datasets/下的文件。  
(4) 将数据复制至VOC2007文件夹里：  

```
cd <ROOT>
find Data/Train/train/ -name "*.jpg" | xargs -i cp {} FSF/detectron2/datasets/VOC2007/JPEGImages/
find Data/Train/label/ -name "*.json" | xargs -i cp {} FSF/detectron2/datasets/VOC2007/Annotations/
# 顺便复制test数据到模型文件专门存放数据的文件夹里
mkdir /FSF/detectron2/datasets/VOC2007/test
find Data/test/ -name "*.kfb" | xargs -i cp {} FSF/detectron2/datasets/VOC2007/test
find Data/test/ -name "*.json" | xargs -i cp {} FSF/detectron2/datasets/VOC2007/test
```

注：阿里云使用软链接的文件无法打开，而且需复制文件太多，所以得如上复制，不能直接用cp。
### 步骤3：训练模型
(1) 在[Model_Zoo](https://github.com/facebookresearch/detectron2/blob/master/MODEL_ZOO.md)里下载ImageNet预训练模型的权重参数文件R-50.pkl，然后在/detectron2/configs/下创建ImageNetPretrained/MSRA/文件路径，并上传至该路径下。  
(2) 修改配置文件后为faster_rcnn_R_50_C4_exp1.yaml，并传至 /detectron2/config/PascalVOC-Detection/下。  

```
cd FSF/detectron2
python tools/train_net.py --num-gpus 2 --config-file configs/PascalVOC-Detection/faster_rcnn_R_50_C4_exp1.yaml
```
注：由于GPU数量的更改，``SOLVER.IMS_PER_BATCH``和``SOLVER.BASE_LR``也需要作出相应修改，具体修改方式请参考[实验记录2](https://github.com/AlvinAi96/WSI_Detection/blob/master/Experiment%20Record%202.md)于2019年12月7日记录内容。除了命令行修改，也可以在yaml配置文件内修改。

### 步骤4：预测结果
```
cd FSF/detectron2
python tools/prediction.py --final_nms_thresh 0.1 --stride_proportion 0.5
# 2GPU 多线程预测
python tools/prediction_multiprocess.py --final_nms_thresh 0.1 --stride_proportion 0.25 --model_weights_pth model_0079999.pth
```
注：关于调整参数还有很多，具体内容如下：  
(1)``--output_config``: 指定使用的config.yaml配置信息文件，默认"./output/config.yaml"。  
(2)``--model_weights_pth``: 指定output内得到的最后模型参数文件，默认“final_model.pth”。  
(3)``--conf_thresh``: 即cfg.MODEL.ROI_HEADS.SCORE_THRESH_TEST，检测某ROI内框时只保留score>conf_thresh的检测结果，默认0.05。  
(4)``--final_nms_switch``: 在kfb全图检测所有给定ROI后是否需要做非极大值抑制冗余检测框，默认True。  
(5)``--final_nms_thresh``: 只留下IOU<final_nms_thresh的检测框, 默认0.1。  
(6)``--img_size``: 输入图像的长/高，默认1000。  
(7)``--stride_proportion``: 在整图滑动预测的窗口移动步长占图像尺寸大小。默认0.5,即1000*1000的窗口以500（1000/2=500）的步长移动与原图之上。  
(8)``--save_path``: 预测结果保存地址，默认'./output/submit_result/'。  
(9)``--test_kfb_path``: 待预测Test文件存放地址，默认'./datasets/VOC2007/test/'。  

### 步骤5： 提交结果文件
```
cd FSF/detectron2/output
zip -r submit_result.zip submit_result/*
# 右键压缩文件提交
```
