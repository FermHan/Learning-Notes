> 注：md文件，Typora书写，有些格式可能不太支持。各个平台修改起来比较麻烦，所以github会上传md文档，可下载用markdown文件查看。
> 发表在CSDN：https://blog.csdn.net/hancoder/article/details/87917174查看。
> 发表在知乎专栏：https://zhuanlan.zhihu.com/c_1088438808227254272

## 学习总结

本文依次讲解了目标检测的必读论文：R-CNN，SPP，Fast R-CNN,Faster R-CNN。
网上有很多“一文读懂...”，其实这类文章只是带入门，知道某个名词是干什么的，真正想要看懂这些文章还是需要先大概看一遍论文。而此文针对的是论文原文与百度结合的读者。因为此文写作时间有点跨度，零散地看到些论文中的重点理解就记录下来，所以此文排版不是非常整齐，但内容还是非常丰富的。

### 前言与知识预热

- 目标检测由来：1966年，Marvin Minsky让他的学生Gerald Jay Sussman花费一个暑假把相机连接到电脑上以使电脑能描述看到的东西。

- 实例分割（instance Segmentation）：标记实例和语义, 不仅要分割出`人`这个类, 而且要分割出`这个人是谁`, 也就是具体的实例。即不仅要分类，而且同类间需要区别每个个体。semantic segmentation - 只标记语义, 也就是说只分割出`人`这个类来


- 定位框的表示：只需（x_0,y_0,width,height）四个参数，即可表示一个框
- 目标检测竞赛：
  - Pascal VOC：20 classes停办
  - COCO：200 classes，2018年COCO竞赛中国团队包揽全部六项任务的冠军，其中旷视取得4项冠军
  - ImageNet ILSVRC停办
 - 目标检测数据集：Caltech2005，Pascal VOC2005,，LabelMe2007，ImageNet2009，Caltech-USCD-birds-200 2010，FGVC-Aircraft2013
 - pre-train和fine-tune：pre-train指的是在ImageNet上对CNN模型进行预训练。预训练的数据和真实训练的数据集不必一样。fine-tune指的是使用真正的数据集进行训练，调整。
- 评价准则：召回率

  - 如图所示，召回率指的是真实标签中有多少被检测出来了

  - IOU（Intersection-over-Union）指的是真实检测框与计算出来的检测框之间的交并比

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190202124252.png)

 - 从一张图中提取候选框Region Proposal方法：EdgeBoxes和Selective Search(SS)。

   - **选择性搜索SS**于层次聚类算法，主要通过框与框之间的颜色，纹理，大小，吻合等相似度排序，合并相邻的相似区域来减少候选框。即不断合并相似的框，直到相似度达到一定程度或者框的数目减小到阈值（~2K个）。合并策略：S = a\*S(color) + b\* S(texture) + c\*S(size) + d\*S(fit); 
- 非极大值抑制
  - 抑制不是最大值的元素，把与指定框（分数高的）overlap大于一定阈值的其他框移除
- 边界框回归：调整边界框

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190202133443.png)

Bounding Box Regression Coefficients (also referred to as “regression coefficients” and “regression targets”)**边框回归系数也叫回归系数，回归对象**: One of the goals of R-CNN is to produce good bounding boxes that closely fit object boundaries. R-CNN produces these bounding boxes by taking a given bounding box (defined by the coordinates of the top left corner, width and height) and tweaking its top left corner, width and height by applying a set of “regression coefficients”. **R-CNN通过左上边坐标和高宽来定义框。**These coefficients are computed as follows (Appendix C of (Anon. 2014)](http://www.telesens.co/2018/03/11/object-detection-and-classification-using-r-cnns/#ITEM-1455-4)[*](https://arxiv.org/pdf/1311.2524.pdf). Let the x, y coordinates of the top left corner of the target and original bounding box be denoted by$T_x,T_y,O_x,O_y$ respectively **分别代表预测和真实的左上角坐标xy**。and the width/height of the target and original bounding box by  $T_w,T_h,O_w,O_h$respectively**分别代表预测和真实的宽高.** Then, the regression targets (coefficients of the function that transform the original bounding box to the target box) are given as:

- $t_x={T_x-O_x\over{O_w}},t_y={T_y-O_y\over{O_h}},t_w=log({T_w \over O_w}),t_h=log({T_h \over O_h})$. 

- This function is readily invertible, i.e., given the regression coefficients and coordinates of the top left corner and the width and height of the original bounding box, the top left corner and width and height of the target box can be easily calculated. **Note the regression coefficients are invariant to an affine transformation with no shear.** This is an important point as while calculating the classification loss, the target regression coefficients are calculated in the original aspect ratio while the classification network output regression coefficients are calculated after the ROI pooling step on square feature maps (1:1 aspect ratio). This will become clearer when we discuss classification loss below.

  ![这11](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190316142242.png)

  

  上面这个图很好地解释了为什么R-CNN里的系数要这么设置，原因在于图缩放后，这些系数保持不变

两阶段

### object detection技术的演进：

-    RCNN（Selective Search + CNN + SVM）
-    ->SPP-net：Spatial Pyramid Pooling（空间金字塔池化）（ROI Pooling）
-    ->Fast R-CNN（Selective Search + CNN + ROI）
-    ->Faster R-CNN（RPN + VGG + ROI）
-    ->Mask R-CNN(resnet +FPN)

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190202123335.png)

## 一. RCNN

2014年，作者RBG

R-CNN解决的是，“为什么不用CNN做classification呢？”

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190203004726.png)

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190202133355.png)

**R-CNN步骤：**

一：用SS提取候选区域
二：用CNN提取区域特征
三：对区域进行SVM分类+边框校准

**R-CNN详解：**



一：用SS提取候选区域

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190202135519.png)

哪一层的特征好用？



二：用CNN提取区域特征

1. 将不同大小的候选区域（原图）缩放到相同大小

2. 将所有区域送入AlexNet提取特征：5个卷积层，2个全连接层

3. 以最后一个全连接网络的输出作为区域的特征表示：

   ![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190221195945.png)

- 有监督预训练pretraining
  - 图像分类任务：使用ImageNet，1000类，仅有图像标签，没有物体边框标准
  - 数据量120张图像，多。此时得到的是分类的网络，而不是检测网络
  - ![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190202224523.png)


- 针对目标任务进行微调 Fine-tuning
  - 	目标检测任务：Pascal VOC，20类，有物体边框标注
  - 	数据量：仅有数千或上万张图像，少。
  - 	已经经过了大量CNN的预训练，只需要少量图片就可以在检测上做的比较好
  - 	![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190202135000.png)
  - 	

三：对区域进行分类+边框校准

- 线性SVM分类器（不使用之前的softmax了）
  - 针对每个类别分类训练，每个类别对应一个SVM分类器
  - ![1550750429616](C:\Users\55373\AppData\Roaming\Typora\typora-user-images\1550750429616.png)
  - 两类分类：one-vs-all
  - 对于摩托车SVM，摩托车是正类，其余是父类。对于自行车SVM，自行车是正类，其余是父类。
  - ![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190202135923.png)

- softmax分类
  - 和整个CNN一起端到端训练
  - 所有类别一起训练
  - 多类分类

- 边框校准（包含更少的背景）

  ![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190221200327.png)

  线性回归模型（x,y,w,h）

$G_x=P_w d_x(P)+P_x$

$G_y=P_y d_y(P)+P_y​$

$G_w=P_wexp(d_w(P))$

$G_h=P_hexp(d_h(P))​$

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190221200614.png)

非极大值抑制算法：得分从大到小排序，最大的分别于后面的IoU比较，与拥有最大值的图片重合度大于0.5的，则丢掉小的。这样丢掉了一部分图片。然后拿出来最大值的图片，第二大的再和其余更小的去比较。

非极大值抑制后可能图片数大大减少了，然后再进行边框回归。

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190221201616.png)

**缺点：**

- 需要变形特征到固定大小，crop/warp会丢失了长宽比信息，影响检测精度
- 低效的流水线，SS耗时，每个候选区域输入到卷积也耗时
- 训练时间很长（48小时）=FineTune（18）+特征提取（63）+SVM/Bbox训练（3）
-  测试阶段很慢：VGG一张图片47s
- ![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190202140555.png)
- 所谓端到端是指在一个神经网络里整体去实现

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/R-CNNsudu.png?token=AkTVJXVR72f4JDP9Qp1rBL3DrVmZB4suks5cSrycwA%3D%3D)

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190202140143.png)

pool5底层的特征，万金油，但是准确率不高

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190202140428.png)

|     | 候选区域目标（RP）   | 特征提取        | 分类 |
| ---------- | ------------ | ------------ | ---- |
| RCNN     | selective search   | CNN          | SVM  |
| 传统的算法 | objectness,constrainedparametric min-cuts,sliding window，edge boxes,.... | HOG , SIFT,LBP, BoW,DPM,... | SVM  |

## 二. SPP - net：Spatial Pyramid Pooling

作者何凯明

R-CNN的低效原因是因为全连接层的输入长度需要固定（而卷积层的输入大小任意），**SPP的贡献是1：将不同大小的特征图归一化到相同大小**。此外：**2“SPP还对整张图计算卷积特征，去除了各个区域的重复计算。在整张图片上只进行一躺卷积，然后SPP-net从feature map上抽取特征。**

SPP-net与R-CNN的对比

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190203191640.png)

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190221202638.png)

区别：RCNN输入原图像的proposal VS SPP输入特征的proposal

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190203011613.png)

SPP：3个level和21个Bin：1\*1，2\*2，4\*4。在bin内使用maxpooling。如图所示，把特征图分别16,4,1等分，从每个等分块中取一个像素，从而构成相同长度的向量。也可以只进行16等分，从而得到长度16的向量。（为了可以把卷积层与全连接层连接）

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190203190738.png)

窗口大小 $win =  ⌈a/n⌉​$，步长 $stride = ⌊a/n⌋​$

解决的是，因为全连接层的输入需要固定尺寸，既然卷积层可以适应任何尺寸，那么只需要在卷积层的最后加入某种结构，使得后面全连接层得到的输入为固定程度就可以了。所以**将金字塔思想加入CNN，提取固定长度的特征向量，实现了数据的多尺度输入**。Spatial Pyramid Pooling添加在卷积后，全连接前。SPP还有一个优化点是：如果对ss提供的2000多个候选区域都逐一进行卷积处理，很耗时。解决思路是先将整张图卷积得到特征图，然后再将ss算法提供的2000多个候选区域的位置记录下来，通过比例映射到整张图的feature map上提取出候选区域的特征图B（映射关系与补偿有关），然后将B送入到金字塔池化层中，进行权重计算。

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190221202921.png)

**SPP-net的缺点：**步骤还是很繁琐

- R-CNN和SPP-net的训练都包含多个单独的步骤

  - 1）多网络进行微调
    - R-CNN对整个CNN进行微调
    - SPP-net只对SPP之后的（全连接）层进行微调
  - 2）训练SVM，
  - 3）训练边框回归模型
- 检测速度慢，尤其是R-CNN

  - RCNN+VGG16：检测一张图需要47s

  - 和RCNN一样，训练过程仍然是隔离的，多阶段。提取候选框 | 计算CNN特征| SVM分类 | Bounding Box回归独立训练，大量的中间结果需要转存，无法整体训练参数；
   2 ）SPP-Net在无法同时Tuning在SPP-Layer两边的卷积层和全连接层，很大程度上限制了深度CNN的效果；
  3）在整个过程中，Proposal Region仍然很耗时。
- 新问题：SPP之前的所有卷积层参数不能finetune

## 三. Fast R-CNN
![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190125081049.png)

**之前方法缺点：**

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190205225521.png)

解决的是，“为什么不一起输出bounding box和label呢？”。把SVM和边框回归去掉，由CNN直接得到类别和边框。

**框架**

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190204160716.png)

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190221211340.png)

1. 输入一张图和一堆bounding boxes（由SS产生）
2. 产生卷积特征图
3. 对于每一个box，通过ROI Pooling层得到一个定长特征向量
4. 输入：（1）K+1类别概率 ，（2）回归框位置

**贡献：**

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190205225455.png)

- 提出了**ROI Pooling**，可以看做是单层SPP-net(single-level SPP)

  ![RoI Pooling animation](https://github.com/deepsense-ai/roi-pooling/raw/master/roi_pooling_animation.gif)

  >简单地说就是把一张图差不多等分成很多份，从每一份中取一个像素，拼凑起来就得到相同长度的向量。窗口大小 $win =  ⌈a/n⌉$，步长 $stride = ⌊a/n⌋$

  ![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190221211621.png)

  >RoI pooling的梯度回传。RoI有重叠，会有一些像素重叠。非重叠的区域是maxpooling类似的梯度回传，如果是重叠的：多个区域的偏导之和。假设有2个区域r0和r1。r0经过roi pooling后假设是2\*2的。pooling之后是2×2的输出。重合了1个像素。对于r0来说是贡献到了右下角，r1贡献到了左上角。梯度回传的时候还有更深的层回传梯度。知道了2×2的梯度往回传的时候，算大图的梯度的时候，是两个小图梯度的相加。
  >
  >

- 引入**多任务学习**，将多个步骤整合到一个模型中。

  ![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190221213531.png)

  - SVM+Regressor -> Multitask Loss(SoftMax + Regressor)
  - $L(p,u,t^u,v)=L_{cls}(p,u)+\lambda[u>1]L_loc(t^u,v)$，背景的时候，u是0
  - 其中分类损失$L_{cls}=-logp_u$

- 边框回归：Lloc![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190206005906.png)

  即对于x,y,w,h，边框回归的损失都是smoothL1损失，t是预测的，v是真实的。u指示的是，背景的边框回归忽略掉不计算。$t^k=(t_x,t_y,t_w,t_h)$

- 全连接层加速：Truncated SVD

  ​	W≈U

  将一个大的全连接层分解层两个小的全连接层

  时间复杂度：O(uv)→O(t(u+v))

- 多任务学习的优势

- 抛弃SVM vs 使用Softmax

  - 为什么不使用SVM了：训练使用难样本挖掘以使网络获得高判别力，从而精准定位目标

- 结果比R-CNN好一些，不需要磁盘存储

缺点：

- Fast R-CNN让然需要专门的候选窗口生成模块
- 候选框提取方法仍是SS：CPU,2s/图，速度慢
- 其他方法：EdgeBox,GPU,0.2S/图

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190221214209.png)

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190221214334.png)

![finetune层多点好](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190221214519.png)

 Fast R-CNN的RegionProposal是在feature map之后做的，这样可以不用对所有的区域进行单独的CNN Forward步骤。

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190204161818.png)

- 分类损失：交叉熵损失
- 回归损失：预测值和真实值差异小于1的话，让损失小一些。如果差异比较大，让损失大一些





## 四. Faster R-CNN = Fast R-CNN + RPN

- 首先，这个链接里讲得很不错，https://zhuanlan.zhihu.com/p/31426458，建议直接转去看这个链接，本文会有参考。先放几张图简单看看Faster 
1：R-CNN的框架![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190204162240.png)

有张比较重要的图太长，放文中不太合适，我放文尾了，也可以点[这个链接](https://img-blog.csdn.net/20170503151750361?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTUxsZWFybmVyVEo=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)（对理解很重要）

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190316143920.png)

![img](https://img-blog.csdn.net/20180304220940218)

下面这张图的目的是为了显示训练是分阶段的，即像之前的方法一样，先产生建议框，然后拿建议框去分类，只不过这里建议框的生成方式换成了RPN网络。

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190222164644.png)

- 之前的Fast R-CNN存在的问题：存在瓶颈：选择性搜索，找出所有的候选框，这个也非常耗时。
     解决方法：加入一个提取边缘的神经网络，也就说找到候选框的工作也交给神经网络来做了，即Region Proposal Network(RPN)。
- Faster R-CNN解决的是，“为什么还要用selective search呢？”----将选择性搜索候选框的方法换成Region Proposal Network(RPN)。

**==2：RPN==**：

- 用CNN来生成候选窗口，量少质优（~300）

- RPN用于直接产生区域候选，不需要额外算法产生区域候选

- 让生成候选窗口的CNN和分类的CNN**共享卷积层**

   **其实RPN最终就是在原图尺度上，设置了密密麻麻的候选Anchor。然后用cnn去判断哪些Anchor是里面有目标的foreground anchor，哪些是没目标的backgroud。所以，仅仅是个二分类而已！**

- 具体做法：

   ![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190221220510.png)

2.1名词介绍：锚点anchors: The k proposals are parameterized relative to k reference boxes,which we call anchors

首先可以看下感受野的内容https://zhuanlan.zhihu.com/p/44106492，“目标检测task中设置anchor要严格对应感受野，anchor太大或偏离感受野都会严重影响检测性能”

即特征图上的最小单位点，比如原始图像的大小是256x256，特征提取网络中含有4个pool层，然后最终获得的特征图的大小为 256/16 x 256/16，即获得一个16x16的特征图，该图中的最小单位即是锚点，由于特征图和原始图像之间存在比例关系，在特征图上面密集的点对应到原始图像上面是有16个像素的间隔，如下图所示。所以anchor是在原图上还是在conv5上？答案是conv5_3上每个点对应9个anchors，但anchors的框是在原图上的

![img](https://img-blog.csdn.net/20180304212850265)

2. 2锚点与原图的对应

   在conv5_3特征图上用3\*3的窗口滑动（stride=1，padding 0填充，即输入输出大小不变，改变的是通道数），每个3\*3窗口可能对应原图多种（k=9种）形状（3种大小，三种形状）。相当于conv5_3上的每一个3\*3图对应原图的某一位置的框。我们让3\*3卷积后得到的下一层特征图rpn/output通道数与上一个特征图conv5_3通道数相等，这样下一层特征图rpn/output的每一个点对应上conv5_3上的一个3\*3的anchor，而conv5_3上的3\*3anchor对应于原图的一块区域。

   此时我们得到的rpn/output特征图相当于**每个点**都对应于**原图某个位置的框**。那我们针对rpn/output特征图上每个点进行处理或者算loss就相当于对原图上的框做处理或者调整。所以我们的思路是用1\*1的卷积核在rpn/output特征图上做滑动，这样滑动得到的特征图还是每个点对应于原图上某个位置的框。这步实际操作的时候是用两组1\*1的卷积两路并行（1\*1卷积特征图宽高不变），一路用于计算边框位置，一路用于计算框出物体的分类，只不过RPN这里我们不进行分类，仅仅区分是前/背景就可以了。这两路的具体形式后面再讲。

   假设每个点对应k个anchors（9个），则边界支路reg层输出通道数是4\*k，分类支路cls层输出通道数是2\*k（评估是/否物体的概率）。然后把这两个支路整合一下$Loss=Loss_{loc/reg}+Loss_{cls}​$，这样我们就利用RPN得到了一些含object的框。

   - 在train阶段，会输出约2000个proposal，但只会抽取其中256个proposal（128positive anchor + 128 negative anchors）来训练RPN的cls+reg结构；到了reference阶段，则直接输出最高score的300个proposal。inference时由于没有了监督信息，所有RPN**并不知道这些proposal是否为前景**，整个过程只是惯性地推送一波无tag的proposal给后面的Fast R-CNN。

   - 滑动窗口的位置提供了在原图像上定位的参考信息，边界框回归根据滑动窗口的参考信息提供了更精准的定位信息。回归器为每个锚点框计算偏移，分类器评测每个锚点框（回归后的）是物体的可能性

   - 3\*3的中心与原图anchor的中心对应

   - RPN/output上每个点对应什么样的框？
     针对每一个点，然后根据不同的尺度（128、256、512pixel）和不同的长宽比（1:1、0.5:1、1:0.5）产生9个BoundingBox，如下图所示，对于16x16的特征图，最终产生16x16x9个候选的ROI，最后原图上的框图下图所示。（可以理解为锚点在conv5_3 [其实是RPN/output] 上，框在原图上）

   - ![img](https://img-blog.csdn.net/20180304220001755)

   - ```python
     anchors，实际上就是一组由rpn/generate_anchors.py生成的矩形。直接运行作者demo中的generate_anchors.py可以得到以下输出：
     
     [[ -84.  -40.   99.   55.] #w=84+99+1=183+1,h=40+55+1=95+1
      [-176.  -88.  191.  103.] #w=367+1,h=191+1
      [-360. -184.  375.  199.] #735+1,383+1
      [ -56.  -56.   71.   71.] #127+1,127+1
      [-120. -120.  135.  135.] #255+1,255+1
      [-248. -248.  263.  263.] #511+1,511+1
      [ -36.  -80.   51.   95.] #87+1,175+1
      [ -80. -168.   95.  183.] #175+1,351+1
      [-168. -344.  183.  359.]]#351+1,703+1
     4个数对应的是左上角和右下角的坐标，第1+第3=第2+第4=15，我们可知这个是左上角坐标为（7,7）那个点的anchors，anchors中长宽1:2中最大为352x704，长宽2:1中最大736x384，基本是cover了800x600的各个尺度和形状。
     - 遍历Conv layers计算获得的feature maps，为每一个点都配备这9种anchors作为初始的检测框。这样做获得检测框很不准确，不用担心，后面还有2次bounding box regression可以修正检测框位置。
     ```

     


  2.3RPN的两个并行支路

首先看之前给过的图的RPN部分

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190310190400.png)

这两个支路中的上面支路用来分类前背景，下面支路用来回归框的位置(xyw宽h高定义了一个框)。

- 分类支路的18=2\*9，2是前背景的one-hot，9是9个anchor。接下来我引用[机器学习随笔](https://zhuanlan.zhihu.com/p/31426458)中的一些内容

> 那么为何要在softmax前后都接一个reshape layer？其实只是为了便于softmax分类，至于具体原因这就要从caffe的实现形式说起了。在caffe基本数据结构blob中以如下形式保存数据：
>
> ```text
> blob=[batch_size, channel，height，width]
> ```
>
> 对应至上面的保存bg/fg anchors的矩阵，其在caffe blob中的存储形式为[1, 2x9, H, W]。而在softmax分类时需要进行fg/bg二分类，所以reshape layer会将其变为[1, 2, 9xH, W]大小，**即单独“腾空”出来一个维度以便softmax分类，之后再reshape回复原状**。贴一段caffe softmax_loss_layer.cpp的reshape函数的解释，非常精辟：
>
> ```cpp
> "Number of labels must match number of predictions; "
> "e.g., if softmax axis == 1 and prediction shape is (N, C, H, W), "
> "label count (number of labels) must be N*H*W, "
> "with integer values in {0, 1, ..., C-1}.";
> ```

- 边框回归支路是靠下的支路。36=4\*9。4代表(x,y,w,h)。

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190310193746.png)

如图，G代表真实框，A代表anchor给出的框，我们的目的是让A更接近G。所以想让A变换后变成G'，这样A更接近G。xy代表的是框的中心，而x,y移动的距离是与w,h有关的，w,h越大，我们就让xy移动得多一些，反之则移动得小一些。为后文方便，我们在此文中定义A是初步预测框（原始anchors），G'是调整后的预测框（foreground anchors），G是真实框GT。

2.4 RPN的**Loss Function**

RPN的Loss由边框位置loss和分类loss组成

因为此时我们分类仅仅区分前景背景，需要定义一个正标签是什么

> 两种anchors被标记为正类label=1，p\*=1：**（1）与真实box有最高的IoU重合度（也许不到0.7），（2）与任意any真实box的IoU重叠度大于0.7。**
> **与所有真实box的IoU重叠度小于0.3的anchor被标记为负类**label=0,p\*=0。**剩下不是正类也不是负类的anchor对训练没有影响label=-1。**
>
> 这样一个真实box可能被赋给多个anchor正标签。通常第二种情况用来分辨正类已经足够了，但是仍然采用第一种的原因是如果全部框的IoU重合度都不大于0.7，那么我们就没框了，序号（1）是为了至少有个框。
>
> 通过IoU会把anchor分为正、负、不参与训练的标签，此外，全部anchors拿去训练太多了，训练程序会在合适的anchors中**随机**选取128个postive anchors+128个negative anchors进行训练

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190207222247.png)

其中t\*代表预测框A与真实框G的距离（平移量），t代表预测框A与调整后的预测框G'的距离。算Loss时算的是算出的平移量与实际平移量之间的误差$L_{reg}(t_i,t_i^*)=R(t_i-t_i^*)$。此外观察一下框回归函数我们会发现只有正标签前景才会计算回归loss，因为loss里乘了p\*。

由于在实际过程中，![N_\text{cls}](https://www.zhihu.com/equation?tex=N_%5Ctext%7Bcls%7D)和![N_\text{reg}](https://www.zhihu.com/equation?tex=N_%5Ctext%7Breg%7D)差距过大，用参数λ平衡二者（如![N_\text{cls}=256](https://www.zhihu.com/equation?tex=N_%5Ctext%7Bcls%7D%3D256)，![N_\text{reg}=2400](https://www.zhihu.com/equation?tex=N_%5Ctext%7Breg%7D%3D2400)时设置 ![λ=\frac{N_\text{reg}}{N_\text{cls}}\approx10](https://www.zhihu.com/equation?tex=%CE%BB%3D%5Cfrac%7BN_%5Ctext%7Breg%7D%7D%7BN_%5Ctext%7Bcls%7D%7D%5Capprox10) ），使总的网络Loss计算过程中能够均匀考虑2种Loss。

R代表的是smooth L1函数，这个函数我们在fast R-CNN里说过了。![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190310201335.png)

至于loss更多的细节还需要去论文或者代码里去读，我们在此简单地认为xywh需要作变换即可。

**==监督信号是Anchor与GT的差距 ![(t_x, t_y, t_w, t_h)](https://www.zhihu.com/equation?tex=%28t_x%2C+t_y%2C+t_w%2C+t_h%29)，即训练目标是：输入 Φ的情况下使网络输出与监督信号尽可能接近。==**

   ![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190221221038.png)

 

2.5 Loss训练的结果

源码中，会生成60\*40\*9(~21k)个anchor box，IoU后约有2k个anchor喂入loss，然后累加上训练好的△x, △y, △w, △h,从而得到了相较于之前更加准确的预测框region proposal，进一步对预测框进行越界剔除和使用NMS非最大值抑制，剔除掉重叠的框，得到越300个框

anchor产生的平均建议框尺寸如下图所示

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190312135805.png)

这样RPN检测框的部分就结束了，后面的步骤是识别的部分了。

3：RPN后的识别

因为RPN之后的网络有全连接层，所以需要进行RoI操作。

- proposal是对应M×N尺度的，所以首先使用spatial_scale参数将其映射回(M/16)×(N/16)大小的feature map尺度
- 再将每个proposal对应的feature map区域水平分为 pooled_w×pooled_h的网格；
- 对网格的每一份都进行max pooling处理。

这样处理后，即使大小不同的proposal输出结果都是 pooled_w×pooled_h 固定大小，实现了固定长度输出。接下来进行分类

跟RPN部分同理，此时利用bounding box regression获得每个proposal的位置偏移量bbox_pred，用于回归更加精确的目标检测框。

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190310223921.png)

> 如果你直接翻到这里，可能不懂RoI Pooling，RoIpooling是为了连接后面的FC层，细节可看fast R-CNN的部分（可能需要从SPP部分才能深刻理解），这部分也可以百度，还算简单。pooling这里在maskR-CNN中被改进为AlignPooling

然后我们利用后面的网络进行真正的分类，此时我们不是仅仅区分前背景了，还要区分阿猫阿狗。

但我们此时可能有疑问，那此不是有两个地方有Loss？RPN的Loss和最后识别的Loss。怎么反向回归？文章利用的是交替训练的方法。

4：模型学习：交替式4步法训练

* 4.1 基于预训练模型只训练RPN

* 4.2 基于预训练模型，以及上一步得到的RPN，训练Fast R-CNN

  >具体而言，RPN输出一个region proposal，通过region proposal映射到原图像，将映射后截取的图像通过几次conv-pool，然后再通过roi-pooling和fc再输出两条支路，一条是目标分类softmax，另一条是bbox回归。**此时，两个网络并没有共享参数**，只是分开训练；


* 4.3 固定共享的卷积层，再次训练RPN

* 4.4 固定共享的卷积层，基于上一步得到的RPN，只训练Fast R-CNN独有部分的参数

  ![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190222160414.png)

Detail:

> RPN与检测网络共享卷积部分，节省计算。但没能实现实时检测
>
> 分类网络的梯度不向RPN回传
>
> 只有在train时，cls+reg才能得到强监督信息(来源于ground truth)。即ground truth会告诉cls+reg结构，哪些才是真的前景，从而引导cls+reg结构学得正确区分前后景的能力；在reference阶段，就要靠cls+reg自力更生了。
>
> RPN的运用使得region proposal的额外开销就只有一个两层网络。
>
> 在train阶段，会输出约2000个proposal。到了reference阶段，则直接输出最高score的300个proposal。inference时由于没有了监督信息，所有RPN**并不知道这些proposal是否为前景**，整个过程只是惯性地推送一波无tag的proposal给后面的Fast R-CNN。
>
> ![这里写图片描述](https://img-blog.csdn.net/20180606125912501?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0pOaW5nV2Vp/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

5：Faster R-CNN总结：

- Fater R-CNN效果![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190222160943.png)

- RoI pooling存在的问题：（mask RCNN解决）
  由于预选ROI的位置通常是有模型回归得到的，一般来说是浮点数，而赤化后的特征图要求尺度固定，因此ROI Pooling这个操作存在两次数据量化的过程。1）将候选框边界量化为整数点坐标值；2）将量化后的边界区域平均分割成kxk个单元，对每个单元的边界进行量化。事实上，经过上面的两次量化操作，此时的ROI已经和最开始的ROI之间存在一定的偏差，这个偏差会影响检测的精确度。

![img](https://img-blog.csdn.net/20180304222739106)



![这里写图片描述](https://img-blog.csdn.net/20180118210431417?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSk5pbmdXZWk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![这里写图片描述](https://img-blog.csdn.net/20170503151750361?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTUxsZWFybmVyVEo=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



下面的图是RPN的部分

![这里写图片描述](https://img-blog.csdn.net/20170503151542992?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTUxsZWFybmVyVEo=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



- **Anchor Generation Layer:** This layer ==generates a fixed number of “anchors”== (bounding boxes) by first generating 9 anchors of different scales and aspect ratios and then replicating these anchors by translating them across uniformly spaced grid points spanning the input image.

- **Proposal Layer:** Transform the anchors according to the bounding box regression coefficients to generate transformed anchors.  ==Then prune the number of anchors by applying non-maximum suppression== (see Appendix) using the probability of an anchor being a foreground region

- **Anchor Target Layer**: The goal of the anchor target layer is to produce a set of “good” anchors and the corresponding foreground/background labels and target regression coefficients to train the Region Proposal Network. ==The output of this layer is only used to train the RPN network and is not used by the classification layer.== ==Given a set of anchors (produced by the anchor generation layer, the anchor target layer identifies promising foreground and background anchors.== Promising foreground anchors are those whose overlap with some ground truth box is higher than a threshold. Background boxes are those whose overlap with any ground truth box is lower than a threshold. The anchor target layer also outputs a set of bounding box regressors i.e., a measure of how far each anchor target is from the closest bounding box. These regressors only make sense for the foreground boxes as there is no notion of “closest bounding box” for a background box.

- RPN Loss: 

  The RPN loss function is the metric that is minimized during optimization to train the RPN network. The loss function is a combination of:

  - The proportion of bounding boxes produced by RPN that are correctly classified as foreground/background
  - Some distance measure between the predicted and target regression coefficients.

- **Proposal Target Layer:** The goal of the proposal target layer is to prune the list of anchors produced by the proposal layer and produce *class specific* bounding box regression targets that can be used to train the classification layer to produce good class labels and regression targets

- **ROI Pooling Layer:** Implements a spatial transformation network that samples the input feature map given the bounding box coordinates of the region proposals produced by the proposal target layer. These coordinates will generally not lie on integer boundaries, thus interpolation based sampling is required.

- **Classification Layer:** The classification layer takes the output feature maps produced by the ROI Pooling Layer and passes them through a series of convolutional layers. The output is fed through two fully connected layers. The first layer produces the class probability distribution for each region proposal and the second layer produces a set of class specific bounding box regressors.

- Classification Loss: 

  Similar to RPN loss, classification loss is the metric that is minimized during optimization to train the classification network. During back propagation, the error gradients flow to the RPN network as well, so training the classification layer modifies the weights of the RPN network as well. We’ll have more to say about this point later. The classification loss is a combination of:

  - The proportion of bounding boxes produced by RPN that are correctly classified (as the correct object class)
  - Some distance measure between the predicted and target regression coefficients.

后续FPN等欢迎去本人博客查看：https://blog.csdn.net/hancoder/


参考：

B站视频：python tensorflow图像处理

https://zhuanlan.zhihu.com/p/31427164 #解读非常好，点进去看专栏

www.zhuanzhi.ai #专知-深度学习：算法到实践

https://cloud.tencent.com/developer/article/1015122

https://blog.csdn.net/wakojosin/article/details/79363224 #RPN

https://blog.csdn.net/mllearnertj/article/details/53709766 #RPN

https://blog.csdn.net/WZZ18191171661/article/details/79439212

https://github.com/rbgirshick/py-faster-rcnn/blob/master/models/pascal_voc/ZF/faster_rcnn_end2end/train.prototxt  #RPN

http://www.cnblogs.com/zf-blog/p/7286405.html #rpn代码理解

https://www.learnopencv.com/selective-search-for-object-detection-cpp-python/

https://blog.csdn.net/v1_vivian/article/details/73275259

https://blog.csdn.net/xiamentingtao/article/details/78598027

https://github.com/deepsense-ai/roi-pooling

http://blog.leanote.com/post/afanti.deng@gmail.com/b5f4f526490b

