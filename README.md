# Deep Learning : YoloV5 object Detector + EigenCam + AttentionModules


In this projects we are going to see different methods to use attention modules on modern
Deep learning  object detectors like yoloV5. Let's see a little bit about it. We'll have
the original YoloV5 on pytorch and we'll take the next topics


⭐ YoloV5 standar Architecture

⭐ YoloV5 + SE attention module

⭐ YoloV5 + CBAM attention module

⭐ We're going to compare our result with differents metrics: Accuracy Curve, Loss Curve, Confusion Matrix and EigenCam.

⭐  Then let's look our model by visual attention using EigenCam


## Using EigenCam from Deep Learning with Python

This code will be useful at the end for visualizaiton porpose

```python
#Algortimo of EigenCam para visualizacion

import warnings
warnings.filterwarnings('ignore')
warnings.simplefilter('ignore')
##import torch    
import cv2
##import numpy as np
import requests
import torchvision.transforms as transforms
from pytorch_grad_cam import EigenCAM
from pytorch_grad_cam.utils.image import show_cam_on_image, scale_cam_image
#from PIL import Image

COLORS = np.random.uniform(0, 255, size=(80, 3))

def parse_detections(results):
    detections = results.pandas().xyxy[0]
    detections = detections.to_dict()
    boxes, colors, names = [], [], []

    for i in range(len(detections["xmin"])):
        confidence = detections["confidence"][i]
        if confidence < 0.2:
            continue
        xmin = int(detections["xmin"][i])
        ymin = int(detections["ymin"][i])
        xmax = int(detections["xmax"][i])
        ymax = int(detections["ymax"][i])
        name = detections["name"][i]
        category = int(detections["class"][i])
        color = COLORS[category]

        boxes.append((xmin, ymin, xmax, ymax))
        colors.append(color)
        names.append(name)
    return boxes, colors, names


def draw_detections(boxes, colors, names, img):
    for box, color, name in zip(boxes, colors, names):
        xmin, ymin, xmax, ymax = box
        cv2.rectangle(
            img,
            (xmin, ymin),
            (xmax, ymax),
            color, 
            2)

        cv2.putText(img, name, (xmin, ymin - 5),
                    cv2.FONT_HERSHEY_SIMPLEX, 0.8, color, 2,
                    lineType=cv2.LINE_AA)
    return img
```

----------

For more  information visit https://jacobgil.github.io/pytorch-gradcam-book/introduction.html

It's highly probably your yolo architecture looks like this. Next you'll se a little changes that
we're going to do.


```yml
# YOLOv5 🚀 by Ultralytics, GPL-3.0 license

# Parameters
nc: 1  # number of classes
depth_multiple: 1.0 #0.33  # model depth multiple
width_multiple: 1.0 #0.50  # layer channel multiple
anchors:
  - [10,13, 16,30, 33,23]  # P3/8
  - [30,61, 62,45, 59,119]  # P4/16
  - [116,90, 156,198, 373,326]  # P5/32

# YOLOv5 v6.0 backbone
backbone:
  # [from, number, module, args]
  [[-1, 1, Conv, [64, 6, 2, 2]],  # 0-P1/2
   [-1, 1, Conv, [128, 3, 2]],  # 1-P2/4
   [-1, 3, C3, [128]],
   [-1, 1, Conv, [256, 3, 2]],  # 3-P3/8
   [-1, 6, C3, [256]],
   [-1, 1, Conv, [512, 3, 2]],  # 5-P4/16
   [-1, 9, C3, [512]],
   [-1, 1, Conv, [1024, 3, 2]],  # 7-P5/32
   [-1, 3, C3, [1024]],   
   [-1, 1, SPPF, [1024, 5]],  # 9
  ]

# YOLOv5 v6.0 head
head:
  [[-1, 1, Conv, [512, 1, 1]],
   [-1, 1, nn.Upsample, [None, 2, 'nearest']],
   [[-1, 6], 1, Concat, [1]],  # cat backbone P4
   [-1, 3, C3, [512, False]],  # 13

   [-1, 1, Conv, [256, 1, 1]],
   [-1, 1, nn.Upsample, [None, 2, 'nearest']],
   [[-1, 4], 1, Concat, [1]],  # cat backbone P3
   [-1, 3, C3, [256, False]],  # 17 (P3/8-small)

   [-1, 1, Conv, [256, 3, 2]],
   [[-1, 14], 1, Concat, [1]],  # cat head P4
   [-1, 3, C3, [512, False]],  # 20 (P4/16-medium)

   [-1, 1, Conv, [512, 3, 2]],
   [[-1, 10], 1, Concat, [1]],  # cat head P5
   [-1, 3, C3, [1024, False]],  # 23 (P5/32-large)

   [[17, 20, 23], 1, Detect, [nc, anchors]],  # Detect(P3, P4, P5)
  ]

```
----------


## Using squeeze and excitation block on pytorch

From this point. It's important that you can manage your fileseffectively, add these python codes to 
the common.py and then copy the .yml architecture  standard file, and add the new content as is shown below
to perform a succes object detector training. After all you'll train with something like this.



```bash
python train.py --img 416 --cfg [ yml_architecture_file ].yaml --hyp hyp.scratch-low.yaml --batch 16 --epochs 20 --data [ yml_data_description ].yaml --weights yolov5s.pt --workers 24 --name yolo_folder_det
```
----------

More information about how to train this detector can be found here https://github.com/ultralytics/yolov5

```python
class SENet(nn.Module):
    def __init__(self, c1,  ratio=16):
        super(SENet, self).__init__()
        #c*1*1
        self.avgpool = nn.AdaptiveAvgPool2d(1)
        self.l1 = nn.Linear(c1, c1 // ratio, bias=False)
        self.relu = nn.ReLU(inplace=True)
        self.l2 = nn.Linear(c1 // ratio, c1, bias=False)
        self.sig = nn.Sigmoid()
    def forward(self, x):
        b, c, _, _ = x.size()
        y = self.avgpool(x).view(b, c)
        y = self.l1(y)
        y = self.relu(y)
        y = self.l2(y)
        y = self.sig(y)
        y = y.view(b, c, 1, 1)
        return x * y.expand_as(x)
```
----------

## Modifying .yml config file for adding our block SENet

Here our head secction will be the same as the original
```yml
# YOLOv5 v6.0 backbone
backbone:
  # [from, number, module, args]
  [[-1, 1, Conv, [64, 6, 2, 2]],  # 0-P1/2
   [-1, 1, Conv, [128, 3, 2]],  # 1-P2/4
   [-1, 3, C3, [128]],
   [-1, 1, Conv, [256, 3, 2]],  # 3-P3/8
   [-1, 6, C3, [256]],
   [-1, 1, Conv, [512, 3, 2]],  # 5-P4/16
   [-1, 9, C3, [512]],
   [-1, 1, Conv, [1024, 3, 2]],  # 7-P5/32
   [-1, 3, C3, [1024]],
   [-1, 1, SENet,[1024]], #SEAttention
   [-1, 1, SPPF, [1024, 5]],  # 9
  ]
```
----------

## Using CBAM block

```python

# We will create some functions that will help us understand how cbam works 
#Channel attention module


class ChannelAttention(nn.Module):
  def __init__(self,in_planes,ratio=16):
    super(ChannelAttention,self).__init__()
    self.avg_pool=nn.AdaptiveAvgPool2d(1)
    self.max_pool=nn.AdaptiveMaxPool2d(1)

    self.fc1=nn.Conv2d(in_planes,in_planes//ratio,1,bias=False)
    self.relu1=nn.ReLU()
    self.fc2=nn.Conv2d(in_planes//ratio,in_planes,1,bias=False)
    self.sigmoid=nn.Sigmoid()

  def forward(self,x):
    avg_out = self.fc2(self.relu1(self.fc1(self.avg_pool(x))))
    max_out = self.fc2(self.relu1(self.fc1(self.max_pool(x))))
    out = avg_out + max_out
    return self.sigmoid(out)

#Defining the spatial module atenttion
class SpatialAttention(nn.Module):
  def __init__(self,kernel_size=7):
    super(SpatialAttention,self).__init__()
    self.conv1=nn.Conv2d(2,1,kernel_size,padding=3,bias=False)
    #kernel size = 7 padding is 3: (n-7+1) +2p = n
    self.sigmoid=nn.Sigmoid()

  def forward(self,x):
    avg_out = torch.mean(x,dim=1,keepdim=True)
    max_out,_ = torch.max(x,dim=1,keepdim=True)
    x=torch.cat([avg_out,max_out],dim=1)
    x=self.conv1(x)
    return self.sigmoid(x)

class CBAM(nn.Module):
  def __init__(self,channelin):
    super(CBAM,self).__init__()
    self.channel_attention= ChannelAttention(channelin)
    self.spatial_attention = SpatialAttention()
  def forward(self,x):
    out = self.channel_attention(x)*x
    #print('outchannels:{}'.format(out.shape))
    out = self.spatial_attention(out)*out
    return out

        
```
----------

## Let's add our CBAM block to the .yml

Here our backbone will be the same as original file.

```yml
# YOLOv5 v6.0 head
head:
  [[-1, 1, Conv, [512, 1, 1]],
   [-1, 1, nn.Upsample, [None, 2, 'nearest']],
   [[-1, 6], 1, Concat, [1]],  # cat backbone P4
   [-1, 3, C3, [512, False]],  # 13

   [-1, 1, Conv, [256, 1, 1]],
   [-1, 1, nn.Upsample, [None, 2, 'nearest']],
   [[-1, 4], 1, Concat, [1]],  # cat backbone P3
   [-1, 3, C3, [256, False]],  # 17 (P3/8-small)
   [-1, 1 , CBAM, [256]],


   [-1, 1, Conv, [256, 3, 2]],
   [[-1, 14], 1, Concat, [1]],  # cat head P4
   [-1, 3, C3, [512, False]],  # 20 (P4/16-medium)
  [-1, 1 , CBAM, [512]],

   [-1, 1, Conv, [512, 3, 2]],
   [[-1, 10], 1, Concat, [1]],  # cat head P5
   [-1, 3, C3, [1024, False]],  # 23 (P5/32-large)
    [-1, 1 , CBAM, [1024]],

   [[17, 20, 23], 1, Detect, [nc, anchors]],  # Detect(P3, P4, P5)
```
----------

## Metrics and evaluating 
Let's see our curves  of F1_curve, P_curve, PR_curve and R_curve for each class
#### Curves :

| Model | F1_curve | P_curve | R_curve |
| ---------------------------------------------------------------|--------------------|-----------------------------------------------------------------------------|---------|
 YoloV5 | <img class="img-results" src="./curves/base_yolov5/F1_curve.png" > | <img class="img-results" src="./curves/base_yolov5/P_curve.png"> | <img src="./curves/base_yolov5/PR_curve.png"> | <img src="./curves/base_yolov5/R_curve.png"> |
 YoloV5 + SE | <img class="img-results" src="./curves/se_yolov5/F1_curve.png" > | <img class="img-results" src="./curves/se_yolov5/P_curve.png"> | <img src="./curves/se_yolov5/PR_curve.png"> | <img src="./curves/se_yolov5/R_curve.png"> |
  YoloV5 + CBAM | <img class="img-results" src="./curves/cbam_yolov5/F1_curve.png" > | <img class="img-results" src="./curves/cbam_yolov5/P_curve.png"> | <img src="./curves/cbam_yolov5/PR_curve.png"> | <img src="./curves/cbam_yolov5/R_curve.png"> |

----------
## Let's see our Batch predictions
----------


| Yolov5 |
| ---------------------------------------------------------------|
|<img class="img-results" src="./curves/base_yolov5/val_batch2_labels.jpg" > |

| Yolov5 + SE |
| ---------------------------------------------------------------|
|<img class="img-results" src="./curves/se_yolov5/val_batch2_labels.jpg" > |

| Yolov5 + CBAM |
| ---------------------------------------------------------------|
|<img class="img-results" src="./curves/cbam_yolov5/val_batch2_labels.jpg" > |

## Confussion Matrix
| YoloV5 | YoloV5 + SE | YoloV5 + CBAM |
| ---------------------------------------------------------------|--------------------|-----------------------------------------------------------------------------|
|<img class="img-results" src="./curves/base_yolov5/confusion_matrix.png" > |<img class="img-results" src="./curves/se_yolov5/confusion_matrix.png" >|<img class="img-results" src="./curves/cbam_yolov5/confusion_matrix.png" > |

----------


## Now let's see our detector per class


| Model | F1_curve | P_curve | R_curve |
| ---------------------------------------------------------------|--------------------|-----------------------------------------------------------------------------|---------|
 YoloV5 | <img class="img-results" src="./curves/base_yolov5/F1_curve.png" > | <img class="img-results" src="./curves/base_yolov5/P_curve.png"> | <img src="./curves/base_yolov5/PR_curve.png"> | <img src="./curves/base_yolov5/R_curve.png"> |
 YoloV5 + SE | <img class="img-results" src="./curves/se_yolov5/F1_curve.png" > | <img class="img-results" src="./curves/se_yolov5/P_curve.png"> | <img src="./curves/se_yolov5/PR_curve.png"> | <img src="./curves/se_yolov5/R_curve.png"> |
  YoloV5 + CBAM | <img class="img-results" src="./curves/cbam_yolov5/F1_curve.png" > | <img class="img-results" src="./curves/cbam_yolov5/P_curve.png"> | <img src="./curves/cbam_yolov5/PR_curve.png"> | <img src="./curves/cbam_yolov5/R_curve.png"> |

----------


## Next we're going to see how good is our architecture
Firstly we'll see the detector, then the grad cam method for class and finally, the gradcam on the interest
region.

----------
### Detection
----------


| Model | YoloV5 | YoloV5 + SE | YoloV5 + CBAM |
| ---------------------------------------------------------------|--------------------|-----------------------------------------------------------------------------|---------|
 bicicleta | <img class="img-results" src="./curves/base_yolov5/detector/bicicleta.png" > | <img class="img-results" src="./curves/se_yolov5/detector/bicicleta.png"> | <img src="./curves/cbam_yolov5/detector/bicicleta.png"> |
 Car | <img class="img-results" src="./curves/base_yolov5/detector/Car.png" > | <img class="img-results" src="./curves/se_yolov5/detector/Car.png"> | <img src="./curves/cbam_yolov5/detector/Car.png"> | 
Bote | <img class="img-results" src="./curves/base_yolov5/detector/Bote.png" > | <img class="img-results" src="./curves/se_yolov5/detector/Bote.png"> | <img src="./curves/cbam_yolov5/detector/Bote.png"> |
Airplane | <img class="img-results" src="./curves/base_yolov5/detector/Airplane.png" > | <img class="img-results" src="./curves/se_yolov5/detector/Airplane.png"> | <img src="./curves/cbam_yolov5/detector/Airplane.png"> |
Helicopter | <img class="img-results" src="./curves/base_yolov5/detector/Helicopter.png" > | <img class="img-results" src="./curves/se_yolov5/detector/Helicopter.png"> | <img src="./curves/cbam_yolov5/detector/Helicopter.png"> |

----------
### GradCam
----------

| Model | YoloV5 | YoloV5 + SE | YoloV5 + CBAM |
| ---------------------------------------------------------------|--------------------|-----------------------------------------------------------------------------|---------|
bicicleta | <img class="img-results" src="./curves/base_yolov5/gradcam/bicicleta.png" > | <img class="img-results" src="./curves/se_yolov5/gradcam/bicicleta.png"> | <img src="./curves/cbam_yolov5/gradcam/bicicleta.png"> |
Car | <img class="img-results" src="./curves/base_yolov5/gradcam/car.png" > | <img class="img-results" src="./curves/se_yolov5/gradcam/Car.png"> | <img src="./curves/cbam_yolov5/gradcam/Car.png"> | 
Bote | <img class="img-results" src="./curves/base_yolov5/gradcam/bote.png" > | <img class="img-results" src="./curves/se_yolov5/gradcam/Bote.png"> | <img src="./curves/cbam_yolov5/gradcam/Bote.png"> |
Airplane | <img class="img-results" src="./curves/base_yolov5/gradcam/Airplane.png" > | <img class="img-results" src="./curves/se_yolov5/gradcam/Airplane.png"> | <img src="./curves/cbam_yolov5/gradcam/Airplane.png"> |
Helicopter | <img class="img-results" src="./curves/base_yolov5/gradcam/Helicopter.png" > | <img class="img-results" src="./curves/se_yolov5/gradcam/Helicopter.png"> | <img src="./curves/cbam_yolov5/gradcam/Helicopter.png"> |


----------
### Detection + GradCam
----------

| Model | YoloV5 | YoloV5 + SE | YoloV5 + CBAM |
| ---------------------------------------------------------------|--------------------|-----------------------------------------------------------------------------|---------|
bicicleta | <img class="img-results" src="./curves/base_yolov5/detectCam/bicicleta.png" > | <img class="img-results" src="./curves/se_yolov5/detectCam/bicicleta.png"> | <img src="./curves/cbam_yolov5/detectCam/bicicleta.png"> |
Car | <img class="img-results" src="./curves/base_yolov5/detectCam/Car.png" > | <img class="img-results" src="./curves/se_yolov5/detectCam/Car.png"> | <img src="./curves/cbam_yolov5/detectCam/Car.png"> | 
Bote | <img class="img-results" src="./curves/base_yolov5/detectCam/Boat.png" > | <img class="img-results" src="./curves/se_yolov5/detectCam/Boat.png"> | <img src="./curves/cbam_yolov5/detectCam/Boat.png"> |
Airplane | <img class="img-results" src="./curves/base_yolov5/detectCam/Airplane.png" > | <img class="img-results" src="./curves/se_yolov5/detectCam/Airplane.png"> | <img src="./curves/cbam_yolov5/detectCam/Airplane.png"> |
Helicopter | <img class="img-results" src="./curves/base_yolov5/detectCam/Helicopter.png" > | <img class="img-results" src="./curves/se_yolov5/detectCam/Helicopter.png"> | <img src="./curves/cbam_yolov5/detectCam/Helicopter.png"> |
----------
## ¿How to improve it?
As you'll able to see. The three architecture are good at detections. However, the attentions modules, can give usa little more understanding about what is the network looging for. It's clear that practicly the attention modules are good option to increase the attention level of our architecture. whether you're using Channel or spatial attention, cbam or newest techniques of eigenCam. 

But now , what we have , it's a little issue specially on our dataset . You could say , that  we have a bad performance in some clases. However, we have a little  a issue a litte bit deeper; an issue about unbalanced on clases, We have five hundred of images per clases, except on the last two ones. And sadly, In that images, there's much background content in there than the object in self. We can test this images on interactive experiences but maybe a better balanced dataset and other techinques for segmentation content could be more effectly, in the case of the classes are surrounded of many similar enviroment background(sky and flat).

So let's try to fix that. Starting from the balanced classes, and maybe Other not-so-difficult background objects. Or segmentation strategies. I'll be posting another project like this, so i'll see you soon. If you want that i make a tutorial about how train this architectures just send me message to my networks!

See you soon!