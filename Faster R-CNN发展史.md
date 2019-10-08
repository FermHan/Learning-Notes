注：md文件，Typora书写，md兼容程度github=CSDN>知乎，若有不兼容处麻烦移步其他平台，github文档供下载。

> 上传在github：https://github.com/FermHan/Learning-Notes 
>发表在CSDN：https://blog.csdn.net/hancoder/article/
> 发表在知乎专栏：https://zhuanlan.zhihu.com/c_1088438808227254272

[TOC]

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190202123335.png)



## 四. Faster R-CNN = Fast R-CNN + RPN

- 推荐链接，https://zhuanlan.zhihu.com/p/31426458 ，http://www.telesens.co/2018/03/11/object-detection-and-classification-using-r-cnns/  ，https://github.com/ruotianluo/pytorch-faster-rcnn 

Faster RCNN特点：

- 使用RPN代替SS

- RPN可以端到端地训练。
- 5 FPS

### 1：R-CNN的框架

有张比较重要的图太长，放文中不太合适，我放文尾了，也可以点[这个链接](https://img-blog.csdn.net/20170503151750361?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTUxsZWFybmVyVEo=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)（对理解很重要）

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190316143920.png)

![img](https://img-blog.csdn.net/20180304220940218)

下面这张图的目的是为了显示训练是分阶段的，即像之前的方法一样，先产生建议框，然后拿建议框去分类，只不过这里建议框的生成方式换成了RPN网络。

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190222164644.png)

- 之前的Fast R-CNN存在的问题：存在瓶颈：选择性搜索，找出所有的候选框，这个也非常耗时。
     解决方法：加入一个提取边缘的神经网络，也就说找到候选框的工作也交给神经网络来做了，即Region Proposal Network(RPN)。
- Faster R-CNN解决的是，“为什么还要用selective search呢？”----将选择性搜索候选框的方法换成Region Proposal Network(RPN)。

### 2：图像的预处理

![](http://www.telesens.co/wp-content/uploads/2018/03/img_5aa46e9e0bbd7-1024x421.png)

减的这个均值不是当前图像像素的平均值，而是与所有训练和测试图像有关。默认的参数值是600和1000。

图像的输入大小任意。

### 3：RPN

- RPN的最终结果是用CNN来生成候选窗口，通过得分排序等方式挑出量少质优的框（~300）

- 让生成候选窗口的CNN和分类的CNN**共享卷积层**

   **其实RPN最终就是在原图尺度上，设置了密密麻麻的候选Anchor。然后用cnn去判断哪些Anchor是里面有目标的foreground anchor，哪些是没目标的backgroud。所以，仅仅是个二分类而已！**

- 具体做法：

   ![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190221220510.png)

2.1名词介绍：**anchors**: The k proposals are parameterized relative to k reference boxes,which we call anchors

首先可以看下感受野的内容https://zhuanlan.zhihu.com/p/44106492，“目标检测task中设置anchor要严格对应感受野，anchor太大或偏离感受野都会严重影响检测性能”

比如原始输入图像的大小是256x256，特征提取网络中含有4个pool层，然后最终获得的特征图的大小为 256/16 x 256/16，即获得一个16x16的特征图，该图中的最小单位即是锚点，由于特征图和原始图像之间存在比例关系，在特征图上面密集的点对应到原始图像上面是有16个像素的间隔。

![img](https://img-blog.csdn.net/20180304212850265)

2. 2锚点与原图的对应

   可能你还是不太清楚anchor是什么。假设我们此时看的是特征图上正中心的一个点，那么以这个点为中心进行3×3卷积，对应的感受野仍是以原始输入图像为中心的，而大小变为：$3×2^4=48$的区域。特征图上3×3卷积可以看做是3×3大小的框，而因为感受野的存在，对应到原图上是在这48个像素内画的一组框，这个数量可能有些庞大，但原始图像上这个区域最多有2,3个物体，所有我们只要设置3种大小，3种长宽比的框就可以了，至于跟真实框的匹配程度，我们只需要通过这9个框慢慢回归到真实框就可以了。总之，conv5_3特征图上的3×3卷积是对应9个anchors预设框的。

   具体这9个anchor的宽高是多少呢？在程序中直接运行作者demo中的generate_anchors.py可以得到9个输出，这9个输出对于每个3×3是通用的，但这个anchors其实是超参。 

   ```PYTHON
   [[ -84.  -40.   99.   55.]
    [-176.  -88.  191.  103.]
    [-360. -184.  375.  199.]
    [ -56.  -56.   71.   71.]
    [-120. -120.  135.  135.]
    [-248. -248.  263.  263.]
    [ -36.  -80.   51.   95.]
    [ -80. -168.   95.  183.]
    [-168. -344.  183.  359.]]
   # anchor得出每个anchor的 4个数对应的是左上角和右下角的坐标，第1+第3=第2+第4=15，我们可知这个是左上角坐标为（7,7）那个点的anchors，anchors中长宽1:2中最大为352x704，长宽2:1中最大736x384，基本是cover了800x600的各个尺度和形状。
   ```

   

   这里卷积的细节是在conv5_3特征图上用3×3的窗口滑动（stride=1，padding 0填充，即输入输出大小不变，改变的是通道数），每个3×3窗口可能对应原图多种（k=9种）形状（3种大小，三种形状）。相当于conv5_3上的每一个3×3图对应原图的某一位置的框。我们让3×3卷积后得到的下一层特征图rpn/output通道数与上一个特征图conv5_3通道数相等，这样下一层特征图rpn/output的每一个点对应上conv5_3上的一个3×3的anchor，而conv5_3上的3×3anchor对应于原图的一块区域。

   此时我们得到的rpn/output特征图相当于**每个点**都对应于**原图某个位置的框**。那我们针对rpn/output特征图上每个点进行处理或者算loss就相当于对原图上的框做处理或者调整。所以我们的思路是用1×1的卷积核在rpn/output特征图上做滑动，这样滑动得到的特征图还是每个点对应于原图上某个位置的框。这步实际操作的时候是用两组1×1的卷积两路并行（1×1卷积特征图宽高不变），一路用于计算边框位置，一路用于计算框出物体的分类，只不过RPN这里我们不进行分类，仅仅区分是前/背景就可以了。这两路的具体形式后面再讲。

   假设每个点对应k个anchors（9个），则边界支路reg层输出通道数是4×k，分类支路cls层输出通道数是2×k（评估是/否物体的概率）。然后把这两个支路整合一下$Loss=Loss_{loc/reg}+Loss_{cls}$，这样我们就利用RPN得到了一些含object的框。

   - 在train阶段，会输出约2000个proposal，但只会抽取其中256个proposal（128positive anchor + 128 negative anchors）来训练RPN的cls+reg结构；到了reference阶段，则直接输出最高score的300个proposal。inference时由于没有了监督信息，所有RPN**并不知道这些proposal是否为前景**，整个过程只是惯性地推送一波无tag的proposal给后面的Fast R-CNN。

   - 滑动窗口的位置提供了在原图像上定位的参考信息，边界框回归根据滑动窗口的参考信息提供了更精准的定位信息。回归器为每个锚点框计算偏移，分类器评测每个锚点框（回归后的）是物体的可能性

   - 3×3的中心与原图anchor的中心对应

   - RPN/output上每个点对应什么样的框？
     针对每一个点，然后根据不同的尺度（128、256、512pixel）和不同的长宽比（1:1、0.5:1、1:0.5）产生9个BoundingBox，如下图所示，对于16x16的特征图，最终产生16x16x9个候选的ROI，最后原图上的框图下图所示。（可以理解为锚点在conv5_3 [其实是RPN/output] 上，框在原图上）

   - 接下来整张图解释了从特征图上一个3×3卷积的anchors对应到原图上的情况，而最后一个图解释了当所有anchor画出来后框是密密麻麻的。![img](https://img-blog.csdn.net/20180304220001755)

     



 2.3RPN的两个并行支路

首先看之前给过的图的RPN部分

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190310190400.png)

这两个支路中的上面支路用来分类前背景，下面支路用来回归框的位置(xyw宽h高定义了一个框)。


- 边框回归支路是靠下的支路。36=4×9。4代表(x,y,w,h)。

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190310193746.png)

如图，G代表真实框，A代表anchor给出的框，我们的目的是让A更接近G。所以想让A变换后变成G'，这样A更接近G。xy代表的是框的中心，而x,y移动的距离是与w,h有关的，w,h越大，我们就让xy移动得多一些，反之则移动得小一些。为后文方便，我们在此文中定义A是初步预测框（原始anchors），G'是调整后的预测框（foreground anchors），G是真实框GT。

#### 2.4 RPN的**Loss Function**

RPN的Loss由边框位置loss和分类loss组成

因为此时我们分类仅仅区分前景背景，需要定义一个正标签是什么

> 两种anchors被标记为正类label=1，p×=1：**（1）与真实box有最高的IoU重合度（也许不到0.7），（2）与任意any真实box的IoU重叠度大于0.7。**
> **与所有真实box的IoU重叠度小于0.3的anchor被标记为负类**label=0,p×=0。**剩下不是正类也不是负类的anchor对训练没有影响label=-1。**
>
> 这样一个真实box可能被赋给多个anchor正标签。通常第二种情况用来分辨正类已经足够了，但是仍然采用第一种的原因是如果全部框的IoU重合度都不大于0.7，那么我们就没框了，序号（1）是为了至少有个框。
>
> 通过IoU会把anchor分为正、负、不参与训练的标签，此外，全部anchors拿去训练太多了，训练程序会在合适的anchors中**随机**选取128个postive anchors+128个negative anchors进行训练

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190207222247.png)

其中t×代表预测框A与真实框G的距离（平移量），t代表预测框A与调整后的预测框G'的距离。算Loss时算的是算出的平移量与实际平移量之间的误差$L_{reg}(t_i,t_i^*)=R(t_i-t_i^*)$。此外观察一下框回归函数我们会发现只有正标签前景才会计算回归loss，因为loss里乘了p×。

由于在实际过程中，![N_\text{cls}](https://www.zhihu.com/equation?tex=N_%5Ctext%7Bcls%7D)和![N_\text{reg}](https://www.zhihu.com/equation?tex=N_%5Ctext%7Breg%7D)差距过大，用参数λ平衡二者（如![N_\text{cls}=256](https://www.zhihu.com/equation?tex=N_%5Ctext%7Bcls%7D%3D256)，![N_\text{reg}=2400](https://www.zhihu.com/equation?tex=N_%5Ctext%7Breg%7D%3D2400)时设置 ![λ=\frac{N_\text{reg}}{N_\text{cls}}\approx10](https://www.zhihu.com/equation?tex=%CE%BB%3D%5Cfrac%7BN_%5Ctext%7Breg%7D%7D%7BN_%5Ctext%7Bcls%7D%7D%5Capprox10) ），使总的网络Loss计算过程中能够均匀考虑2种Loss。

R代表的是smooth L1函数，这个函数我们在fast R-CNN里说过了。![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190310201335.png)

至于loss更多的细节还需要去论文或者代码里去读，我们在此简单地认为xywh需要作变换即可。

**==监督信号是Anchor与GT的差距 ![(t_x, t_y, t_w, t_h)](https://www.zhihu.com/equation?tex=%28t_x%2C+t_y%2C+t_w%2C+t_h%29)，即训练目标是：输入 Φ的情况下使网络输出与监督信号尽可能接近。==**

   ![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190221221038.png)

 

2.5 Loss训练的结果

源码中，会生成60×40×9(~21k)个anchor box，IoU后约有2k个anchor喂入loss，然后累加上训练好的△x, △y, △w, △h,从而得到了相较于之前更加准确的预测框region proposal，进一步对预测框进行越界剔除和使用NMS非最大值抑制，剔除掉重叠的框，得到越300个框

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
  由于预选ROI的位置通常是由模型回归得到的，一般来说是浮点数，而池化后的特征图要求尺度固定，因此ROI Pooling这个操作存在两次数据量化的过程。1）将候选框边界量化为整数点坐标值；2）将量化后的边界区域平均分割成kxk个单元，对每个单元的边界进行量化。事实上，经过上面的两次量化操作，此时的ROI已经和最开始的ROI之间存在一定的偏差，这个偏差会影响检测的精确度。

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

后续FPN，CornerNet等欢迎去本人博客查看：https://blog.csdn.net/hancoder/


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

