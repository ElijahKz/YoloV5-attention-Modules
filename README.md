# YoloV5-attention-Modules
# Deep Learning : YoloV5 object Detector + GradCam + AttentionModules


In this projects we are going to see different methods to use attention modules on modern
Deep learning  object detectors like yoloV5. Let's see a little bit about it. We'll have
the original YoloV5 on pytorch and we'll take the next topics


⭐ YoloV5 standar Architecture

⭐ YoloV5 + SE attention module

⭐ YoloV5 + CBAM attention module

⭐ We're going to compare our result with differents metrics: Accuracy Curve, Loss Curve, Confusion Matrix y GradCam.

⭐  Then let's look our model by visual attention using EigenCam


# Using GradCAM from Deep Learning with Python

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

For more  information visit https://jacobgil.github.io/pytorch-gradcam-book/introduction.html
----------


# Using squeeze and excitation block on pytorch

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

# Then we modify our .yml config file for adding our block SENet

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