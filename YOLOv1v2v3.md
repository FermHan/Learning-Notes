本文依次讲解YOLOv1,v2,v3。博客地址https://blog.csdn.net/hancoder/article/details/87994678

[TOC]

# YOLOv1

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190916120531.png)

- Over-Feat Sermanet等人
- YOLO ，Redmon等人
- SSD 2016

## 1.1 Introduction

名字解释：You Only Look Once: Unified, Real-Time Object Detection

You Only Look Once说的是只需要一次CNN运算，Unified指的是这是一个统一的框架，提供end-to-end的预测，而Real-Time体现是Yolo算法速度快。

## 1.2 Unified Detection

我们先大概看一下YOLO的模型，我们刚开始只需知道输出是7\*7\*30就好。然后我们一步步分析为什么输出是7\*7\*30，这输出代表什么？

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190301170551.png)

YOLO可以同时预测一张图片中的所有类别和框，先简单看一下下图效果。在这里我们看看输出是3个框效果还不错就好。



![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190228093116.png)

结合着上图，我们先熟悉一些内容：我们会把图分成7\*7的小图，而【每个小图】会关联着2个形状不规则的【bounding box】框。既然有两个box，那么就得由两组坐标来表示，所以【每个box】有5个参数（=4+1）：中心点的xywh+【每个box】还有1个置信度confidence，它的作用我们可以暂且简单理解为画的框与真实框groundtruth框的交并比IoU；【每个小图】有一组条件类别概率（个数等于训练类别数目20个）。

不知道有没有细心的同学有这样的疑惑：上面不应该是【每个box】都有一组类别概率吗？怎么是【每个小图】呢？其实这也是YOLOv1的缺点，每个小图是不允许有两种类别的。这可能出于速度、准确度等考虑。

我们把上面的话变得专业一点：把小图称作网格单元grid cell，每个小图grid cell 的 边界框bounding box数目是B。置信度confidence，类别为class，这两个单词开头都是C，大小c用法不是很统一。

即可表述为：

YOLO它**将图像分成$S \times S$（S=7）的网格grid cell，并且每个网格单元负责预测$B$（B=2）个边界框box。加上每个边界框box的置信度confidence，每个网格的$C$个条件类别概率，grid cell内有物体时，每个网格即可最后被编码长度输出为$S \times S \times (B*5 + C)$的张量**。

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190301195521.png)

**上图即目标检测的输出维度。**

==每个网格并不是只要有物体就要为之画框的。而是只有当物体的【中心】落到那个网格grid cell里，那个网格grid cell才负责检测那个物体==。在形式上，我们将**==置信度confidence定义为==**$\Pr(\textrm{Object}) * \textrm{IOU}_{\textrm{pred}}^{\textrm{truth}}$。**如果该网格中不存在目标的中心，则置信度分数应是零**(因为Pr(Object)=0)。否则，我们希望==置信度分数等于预测框与真实值之间联合部分的交集（IOU）==（即Pr(Object)应趋向1，因而Confidence=IoU）。

> 注意：**上面2个相乘的是置信度，下面还有类似3个相乘的称为类别置信度**，注意区分。

每个边界框包含5个预测：$x$，$y$，$w$，$h$和置信度c，即自身bounding box位置+confidence。$(x，y)$坐标表示边界框的中心点相对于这个grid cell左上角的偏移（因为xy代表的是框的中心，所以我们经常用$x_c,y_c$表示框的坐标）。xywh都会被归一化（0,1）区间的。wh归一化比较简单：w=框的宽度/图片总宽度。因为xy只是中心点的相对于grid cell左上角的偏移。具体而言，xy是如下图

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190326160323.png)

$x=x_c · {{S}\over {width_{image}}}-col={{x_c}\over{width_{grid}}}-col$

$y=y_c · {{S}\over {height_{image}}}-row={{y_c}\over{height_{grid}}}-row$

> 解读 ：$x=x_c / {{width_{image}}\over {S}}-col$代表先把图分成7份，${{width_{image}}\over {S}}$是每个grid宽度，$x_c / {{width_{image}}\over {S}}$代表$x_c$包含几点几个小grid，再$-col$代表$x_c$在单个grid里的相对位置
>
> 通常情况下，YOLO不预测边界框中心的绝对坐标。它预测的是偏移量，预测的结果通过一个sigmoid函数，迫使输出的值在0和1之间。例如，考虑上图中狗的情况。如果对中心xy的预测是（0.4,0.7），那么这意味着中心位于13×13特征地图上的（2.4,5.7）。 （因为红细胞的左上角坐标是（2,5））。
> 但如果预测的x，y坐标大于1，会发生什么情况，比如（1.2,0.7）。这意味着中心位于（3.2,5.7）。注意现在中心位于我们的红色区域或第2排的第5个单元格的右侧。这打破了YOLO背后的理论，因为如果我们假设红色区域负责预测狗，狗的中心必须位于红色区域中，而不是位于红色区域旁边的其他网格里。
> 因此，为了解决这个问题，输出是通过一个sigmoid函数传递的，该函数在0到1的范围内压扁输出，有效地将中心保持在预测的网格中。下面的这个公式详细展示了预测结果如何转化为最终box的预测的。
>
> 即$b_x=\sigma(t_x)+c_x$,$b_y=\sigma(t_y)+c_y$,$b_w=p_we^{t_w}$,$b_h=p_he^{t_h}$
>
> 虽然每个格子可以预测B个bounding box，但是最终只选择只选择IOU最高的bounding box作为物体检测输出，即每个格子最多只预测出一个物体。当物体占画面比例较小，如图像中包含畜群或鸟群时，每个格子包含多个物体，但却只能检测出其中一个。这是YOLO方法的一个缺陷。

## 1.3 网络框架

受GoogLeNet启发，YOLOv1有24个卷积层+2个全连接层（fastYOLO版本中只适应9个conv）。没有用Inception模块，而是用1×1卷积+3×3卷积。1×1卷积用来降维，3×3卷积用来恢复到正常通道数。

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190601184648.png)

YOLOv1在ImageNet1000类上预训练，预训练的时候只使用网络的前20个卷积层+一个全连接。

预训练后用到检测网络上，去掉全连接，加了4个卷积层和两个全连接。

## 1.4 Loss

知道了模式之后我们面对更难的问题是面对30维如此多的参数，我们如何定义与ground truth的距离。

**Loss = λcoord ×坐标预测误差 + （含object的box confidence预测误差 + λnoobj ×不含object的box confidence预测误差） + 类别预测误差**

展开就是
$$
\begin{multline} 
Loss=\lambda_\textbf{coord}\sum_{i = 0}^{S^2}\sum_{j = 0}^{B} \mathbb{1}_{ij}^{\text{obj}}\left[\left( x_i - \hat{x}_i \right)^2 +\left( y_i - \hat{y}_i \right)^2\right] 
\\+ \lambda_\textbf{coord}\sum_{i = 0}^{S^2}\sum_{j = 0}^{B} \mathbb{1}_{ij}^{\text{obj}}\left[\left( \sqrt{w_i} - \sqrt{\hat{w}_i} \right)^2 +\left( \sqrt{h_i} - \sqrt{\hat{h}_i} \right)^2\right]  
\\+ \sum_{i = 0}^{S^2}\sum_{j = 0}^{B} \mathbb{1}_{ij}^{\text{obj}}\left( C_i - \hat{C}_i \right)^2  
\\+ \lambda_\textrm{noobj}\sum_{i = 0}^{S^2}\sum_{j = 0}^{B}\mathbb{1}_{ij}^{\text{noobj}}\left( C_i - \hat{C}_i \right)^2  
\\+ \sum_{i = 0}^{S^2} \mathbb{1}_i^{\text{obj}}\sum_{c \in \textrm{classes}}\left(  p_i(c) - \hat{p}_i(c)  \right)^2
\end{multline}
$$
对比图：![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190301201831.png)

其中：C为confidence，$\mathbb{1}_i^{\text{obj}}$表示目标的中心是否出现在网格单元$i$中，$\mathbb{1}_{ij}^{\text{obj}}$表示网格单元$i$中的第$j$个边界框预测器“负责”该预测。每个网格的30维的对应关系如图所示。（这个1可以在后面理解）

每个网格单元还预测$C$个条件类别概率$\Pr(\textrm{Class}_i | \textrm{Object})$。这些概率以包含目标的网格单元为条件。每个网格单元grid cell我们只预测的一组类别概率，而不管边界框的的数量$B$是多少。（这里的一组指的是只保留一组（20个）分类，所以最后每个网格的维度是5(xywhc)+5(xywhc)+20(class)，省略了另外一个20）。

### 解读LOSS：

Loss=回归检测框 + 预测置信度（物体性）+ 预测是背景的分数+预测物体类别

我们**使用平方和衡量坐标误差，因为它很容易进行优化**，但是它并不完全符合我们最大化平均精度的目标。

分类误差与定位误差的权重是一样的，这可能并不理想。

**另外，在每张图像中，许多网格单元不包含任何对象。这将导致这些单元格的“置信度”分数趋向零，通常压倒了包含目标的单元格的梯度。这可能导致模型不稳定，从而导致训练早期发散。**



**权重的理解**：简言之就是大多数网格与box并不包含object，正负样本不均衡，比如输出7\*7\*30的张量中可能大多数无物体confidence是代入![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190301210410.png)这项里面计算的，只有少量几个的confidence是代入![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190301210346.png)这里面计算的。我们需要让背景的confidence权重减少一点。我们使用两个参数$\lambda_\textrm{coord}$和$\lambda_\textrm{noobj}$来完成这个工作。我们设置  $ \lambda_\textrm{coord} = 5$和 $ \lambda_\textrm{noobj} = 0.5$ ，相当于背景的权重是前景物体框权重的一半。而前景的框位置的权重是置信度的5倍，背景不计算框位置loss。



**框与gt的匹配**：虽然YOLO每个网格预测多个框，但是在训练的时候，==对于每个物体，我们只想让一个框去负责预测这个物体。我们就让与这个gt框有最高IOU值的框负责预测这个物体==。这将让框预测更specialization。每个预测更好地预测固定的大小，比率或者物体类表，提供了整体召回率。



**$1_{ij}^{obj}$与$1_{ij}^{noobj}$的理解**：$1_{ij}^{obj}$表示由第i个网格中第j个边界框预测器负责预测，结合分配gt的理解方式即$1_{ij}^{obj}$的个数等于gt框的个数。此外需要注意的是如果box所在的框不是物体的中心，只会代入loss中的![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190301210410.png)这一项，不带入其余4项。第5项为分类误差，仅在该网格内有物体时才计算。前两项为位置误差，仅在框与gt有最大IOU时候计算。在这个地方我们可以看出每个cell的2个box只取其中一个，至少有一个当做了背景。$1_i^{obj}$表示物体是否出现在了网格i中，第i个网格内有物体才会代入分类误差中

>如果某个单元格中没有目标,则不对分类误差进行反向传播;B个bbox中与GT具有最高IoU的一个进行坐标误差的反向传播,其余不进行.



**大小框误差规模的解决**：对于大box来说与groundtruth差1毫米可能无所谓，但对于小box来说偏差很谨慎。small deviations in large boxes matter less than in small boxes. 为了部分解决这个问题，我们直接**预测边界框==宽度和高度的平方根==，而不是宽度和高度**。（联想根号x图像：小box 的横轴值较小，发生偏移时，反应到y轴上相比大 box 要大，较小的坐标误差比较大的边界框更重要，更敏感。）

```python
# class_loss, 计算类别的损失,p
class_delta = response * (predict_classes - classes)#response查看哪个cell负责标记object
class_loss = tf.reduce_mean(   #平方差损失函数
    tf.reduce_sum(tf.square(class_delta), axis=[1, 2, 3]),
    name='class_loss') * self.class_scale   # self.class_scale为损失函数前面的系数

# 有目标的时候，置信度损失函数
object_delta = object_mask * (predict_scales - iou_predict_truth)  
#用iou_predict_truth替代真实的置信度，真的妙，佩服的5体投递
object_loss = tf.reduce_mean(  #平方差损失函数
    tf.reduce_sum(tf.square(object_delta), axis=[1, 2, 3]),
    name='object_loss') * self.object_scale

# 没有目标的时候，置信度的损失函数
noobject_delta = noobject_mask * predict_scales
noobject_loss = tf.reduce_mean(       #平方差损失函数
    tf.reduce_sum(tf.square(noobject_delta), axis=[1, 2, 3]),
    name='noobject_loss') * self.noobject_scale

# 框坐标的损失，只计算有目标的cell中iou最大的那个框的损失，即用这个iou最大的框来负责预测这个框，其它不管，乘以0
coord_mask = tf.expand_dims(object_mask, 4)  # object_mask其维度为：[batch_size, 7, 7, 2]， 扩展维度之后变成[batch_size, 7, 7, 2, 1]
boxes_delta = coord_mask * (predict_boxes - boxes_tran)     #predict_boxes维度为： [batch_size, 7, 7, 2, 4]，这些框的坐标都是偏移值
coord_loss = tf.reduce_mean(  #平方差损失函数
    tf.reduce_sum(tf.square(boxes_delta), axis=[1, 2, 3, 4]),
    name='coord_loss') * self.coord_scale
```

使用了0.5的dropout。数据增强

至此，我们就训练好了网络。而当测试的时候，我们还要做一些改变。

Pr(Object)是不是就是上面的1ij ?仍需看代码怎么实现的

## 1.5 test

定义：类别置信度 = 条件类别概率 \* 置信度，$$\Pr(\textrm{Class}_i | \textrm{Object}) * \Pr(\textrm{Object}) * \textrm{IOU}_{\textrm{pred}}^{\textrm{truth}} = \Pr(\textrm{Class}_i)*\textrm{IOU}_{\textrm{pred}}^{\textrm{truth}}$$ 

即每个网格预测的class信息乘以每个bounding box预测的confidence信息。这得到了 每个bounding box特定类别的置信度分数（暂且叫做：类别置信度）。类别置信度包含了类别在框中出现的概率以及预测框与物体的拟合程度。

左式第一项为每个网格预测的类别信息，第二、三项为每个bounding box预测的confidence。

它为我们提供了==<u>每个bounding box特定类别</u>的置信度分数（暂且叫做：类别置信度）==。

得到每个 box 的 class-specific confidence score 以后，设置阈值，滤掉得分低的 boxes，对保留的 boxes 进行 NMS 处理，就得到最终的检测结果。

类别置信度在输出的7\*7\*30张量并没有直观表示的，而是需要我们计算得出。如下图，对于每个网格grid cell。

- ![同一grid cell的第一个box](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190301213306.png)
- ![同一grid cell的第2个box](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190301213349.png)

如上俩图所示，同一个grid cell有2个box，所以得到2组类别置信度。每组类别置信度是用每个box的confidence乘以同一网格的类别概率得到的。这里还想说明的是既然后面乘的是同一组概率，那么两组类别置信度还是与置信度乘正比的，因为乘的概率一定。因为概率的最大值对应的类别是确定的，所以两组类别置信度算出来的box只能被网络认为是同一分类，不可能出现同一grid cell不同分类的情况的。

虽然每个格子可以预测 B 个 bounding box，但是最终只选择 IOU 最高的 bounding box 作为物体检测输出，即每个格子最多只预测出一个物体。当物体占画面比例较小，如图像中包含畜群或鸟群时，每个格子包含多个物体，但却只能检测出其中一个。这是 YOLO 方法的一个缺陷。

接着每个网格都计算出了2个置信度向量，如下面3张流程图，注意找不同。

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190301214048.png)

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190301214113.png)

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190301214131.png)

共得到7\*7\*2=98个类别置信度向量（竖条黄色线）。

接下来我们从每类的视角分析类别置信度，比如对于狗类来说，狗的类别置信度都是在类别置信度向量的第一个位置。（狗是横条红虚线）

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190301214455.png)

第一步我们先将每一横条中少于一定阈值thresh的数值置为0，比如小于0.2，这步称为set zero。然后以竖条为单位，将狗的类别置信度（红线）从大到小排列，这样大值在前，最后是0，这步称为sort descending。然后对狗进行非极大值抑制NMS操作，以防止同一物体被几个框同时圈到，很多余。

得到每个 box 的 类别置信度class-specific confidence score 以后，首先设置阈值，滤掉得分低的 boxes，然后对保留的 boxes 进行 NMS 处理，就得到最终的检测结果。（设置阈值代表20种class中只会留很少的class，NMS非极大值抑制的主要作用是某个物体被几个框选择到了，我们把多余的框过滤只要最准确的那个框）。NMS是通过类别置信度来筛选的。

> 注：
>
> **NMS原理：** 首先从所有的预测框中找到置信度最大的那个bbox（已经排好序放到最前面了），然后挨个计算其与剩余bbox的IOU，如果IoU值大于一定阈值（重合度过高，如0.5），那么就将该类别置信度值置为0，把该bbox剔除；然后对剩余的预测框重复上述过程，直到处理完所有的检测框。
>
> *虽然每个格子可以预测 B 个 bounding box，但是最终只选择只选择 IOU 最高的 bounding box 作为物体检测输出，即==每个格子最多只预测出一个物体==。**当物体占画面比例较小，如图像中包含畜群或鸟群时，每个格子包含多个物体，但却只能检测出其中一个。这是 YOLO 方法的一个缺陷**。

### 附：NMS示例:

狗类别的置信度效果如图所示
![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190301221316.png)

然后我们以此计算其他类别的，直到算到最后一个类别后
![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190301221354.png)

对于整张图是这样的效果
![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190301221433.png)

NMS进行后，很多类别置信度被置为0了。（实际上我此时也不清楚到底内存空间里里有没有存放类别置信度，我觉得很可能没用，这里“类别置信度置为0”只是把之前的置信度confidence置为0了，同样的效果）。然后我们就可以如上图算NMS后的，每个box的类别置信度向量，其中每一条的最大值即为这个box的预测类别。（很多box都没0称为背景了）。

然后我们依次计算每个box的最大值，每1竖条的最大值即该box的预测类别。下图表示第一个box是物体
![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190301221739.png)

下图表示第二个box不是物体，因为最大值还是0,
![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190301221821.png)

然后我们就得到了比如3个框。
![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190301221907.png)





## 1.7 YOLOv1结语

YOLO的特点：

YOLO 对相互靠的很近的物体，还有很小的群体检测效果不好，这是因为一个网格中只预测了两个框，并且只属于一类。

同一类物体出现的新的不常见的长宽比和其他情况时，泛化能力偏弱。

由于损失函数的问题，定位误差是影响检测效果的主要原因。尤其是小物体的处理上，还有待加强。

速度特别快，泛化能力强，提供速度，降低了精度

但小物体，重叠物体无法检测



v1对于整个yolo系列的价值，即v2/v3还保留的特性，可以总结为3点： 
1. leaky ReLU，相比普通ReLU，leaky并不会让负数直接为0，而是乘以一个很小的系数(恒定)，保留负数输出，但衰减负数输出；公式如下:
y=x,x>0；y=0.1x,otherwise

2. 分而治之，用网格来划分图片区域，每块区域独立检测目标； 
3. 端到端训练。损失函数的反向传播可以贯穿整个网络，这也是one-stage检测算法的优势。

## 待解决问题

最后为什么要用全连接？ 直接从7×7×1024卷积到7×7×30不就可以了？为什么中间要有个4096全连接？

后续：YOLOv2(9000)，YOLOv3

YOLOv1参考：

翻译：https://www.jianshu.com/p/a2a22b0c4742（md格式）

解读：https://blog.csdn.net/guleileo/article/details/80581858

代码：https://github.com/gliese581gg/YOLO_tensorflow

视频：https://www.bilibili.com/video/av23354360

PPT：https://drive.google.com/file/d/164mVbMBhoMzY5pkaEOdK3IIcIwTOj2B-/view

解读：https://blog.csdn.net/baidu_27643275/article/details/82789212

解读：https://zhuanlan.zhihu.com/p/31427164

解读：https://blog.csdn.net/leviopku/article/details/82660381

# YOLOv2

YOLOv2快过Faster R-CNN，SSD

> 这篇文章一共介绍了YOLO v2和YOLO9000两个模型，二者略有不同。前者主要是YOLOv1的升级版，后者的主要检测网络也是YOLO v2，同时对数据集做了融合，使得模型可以检测9000多类物体。而提出YOLO9000的原因主要是目前检测的数据集数据量较小，因此利用数量较大的分类数据集来帮助训练检测模型。

Better，Faster部分讲YOLOv2；Stronger讲YOLO9000


## 2.1 Better更好

### 2.1.1 Batch Normalization:

使用 Batch Normalization 对网络进行优化。对网络的每一层的输入都做了归一化，这样网络就不需要每层都去学数据的分布，让网络提高了收敛性，更快收敛，同时还消除了对其他形式的正则化（regularization）的依赖。

使用 Batch Normalization 可以从模型中去掉 Dropout，而不会产生过拟合。

通过对 YOLO 的每一个卷积层增加 Batch Normalization，最终使得 mAP 提高了 2%，同时还使模型正则化。YOLO3代均采用BN。

### 2.1.2 High resolution classifier

目前业界标准的检测方法，都要先把分类器（即卷积分类网络）放在ImageNet上进行预训练，这样卷积网络对物体更敏感。在预训练的基础上对网络进行改进与进一步训练。从 Alexnet 开始，大多数的分类器都运行在小于 256×256 的图片上，因为一般是预训练的ImageNet。而现在 YOLO 从 224×224 增加到了 448×448，这就意味着网络需要适应新的输入分辨率。

为了适应新的分辨率，YOLO v2 的分类网络以 448×448 的分辨率先在 ImageNet上进行微调，微调 10 个 epochs，**让网络有时间调整滤波器（filters），好让其能更好的运行在新分辨率上**，还需要调优用于检测的 Resulting Network。最终通过使用高分辨率，mAP 提升了 4%。

### 2.1.3 Convolution with anchor boxes

YOLOv1包含有全连接层，从而能直接预测 Bounding Boxes 的坐标值。  Faster R-CNN 只用卷积层与 Region Proposal Network 来预测 Anchor Box 偏移值offset与置信度（前景得分），而不是直接预测坐标值。作者发现通过预测偏移量offset而不是坐标值能够简化问题，让神经网络学习起来更容易。

所以 YOLOv2 去掉了全连接层，使用 Anchor Boxes 来预测 Bounding Boxes。作者去掉了网络中一个全连接和一个池化层，这让卷积层的输出能有更高的分辨率。

用 416×416 大小的输入代替原来448×448。由于图片中的物体都倾向于出现在图片的中心位置，特别是那种比较大的物体，所以有一个单独位于物体中心的位置用于预测这些物体。YOLO 的卷积层采用 32 这个值来下采样图片，所以通过选择 416×416 用作输入尺寸最终能输出一个 13×13 的特征图。（416/2^5=13）

 使用 Anchor Box 会让精确度稍微下降，但用了它能让 YOLOv2 能预测出大于一千个框，同时 recall 达到88%，mAP 达到 69.2%。

| without anchor  | 69.5 mAP              | 81% recall              |
| --------------- | --------------------- | ----------------------- |
| **with anchor** | **69.2 mAP**：降低0.3 | **88% recall**：提升7.0 |

### 2.1.4 Dimension clusters

之前 Anchor Box 的尺寸是手动选择的（一般指faster rcnn中的anchor），所以尺寸还有优化的余地。 为了优化，在训练集的 Bounding Boxes 上跑一下 k-means聚类，来为anchor找到一个比较好的初始值。（anchor基本等同于prior）

如果我们用标准的欧式距离的 k-means，尺寸大的框比小框产生更多的错误。因为我们的目的是提高 IOU 分数，我们对好坏的标准是IOU值，而与 Box 的大小无关，所以距离度量的使用下面公式更好： 

$d(box,centroid)=1-IOU(box,centroid)$

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190604212812.png)

通过分析实验结果（Figure 2），左图：在模型复杂性与 high recall 之间权衡之后，选择聚类分类数 K=5。

结果是矮宽框较少，而高瘦框多。

Table1 表面用k聚类和原始的anchor box时的平均IOU。（这个平均IOU是与gt框最接近的prior的IOU的平均数）可以看出使用5个prior的平均IOU（61%）就好过了原来使用8个anchor的平均准确率（60.9%）。当选用9个prior时，效果更明显，平均准确率达到了67.2%，可见用k聚类很有效。

| Box Generation | #    | Avg IOU |
| -------------- | ---- | ------- |
| Cluster SSE    | 5    | 58.7    |
| Cluster IOU    | 5    | 61      |
| Anchor Boxes   | 9    | 60.9    |
| Cluster IOU    | 9    | 67.2    |

> cfg文件中的内容
>
> COCO: (0.57273, 0.677385), (1.87446, 2.06253), (3.33843, 5.47434), (7.88282, 3.52778), (9.77052, 9.16828)
> VOC: (1.3221, 1.73145), (3.19275, 4.00944), (5.05587, 8.09892), (9.47112, 4.84053), (11.2364, 10.0071)
>
> 应该是很对于卷积图13×13

### 2.1.5 Direct location prediction

用 Anchor Box 的方法，会让 model 变得不稳定，尤其是在最开始的几次迭代的时候。大多数不稳定因素产生自预测 Box 的（x,y）位置的时候。

因为像在RPN网络中使用的offset策略：$x=(t_x*w_a)-x_a$,$y=(t_y*h_a)-y_a$。可想而知，只要t=-1和1这个框就可能偏移了整张图像，这使训练话费较长时间趋于稳定。

与RPN不同，YOLOv2还是像YOLOv1一样，预测的t是与网格相关的。假设一个网格的大小是1，只要在t输出之前加一个sigmoid即可让比例t在0-1之间，从而让框的中心点限制在一个网格中。

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190604220631.png)

每个网格负责预测5个priors，即预测框。每个框有5个坐标 tx，ty，tw，th，t0。令cx，cy为网格左上角的坐标，pw,ph为prior的宽高，to是置信度（置信度：含有物体的概率 × 与gt的IOU）。即可得到预测框的计算公式：


$$
b_x=σ(t_x)+c_x,	
b_y = σ(t_y) + c_y ,
b_w = p_we^{t_w},
b_h = p_he^{t_h}，
$$

$$
Pr(object) ∗ IOU(b, object) = σ(t_o)
$$

因为使用了限制让数值变得参数化，也让网络更容易学习、更稳定。使用聚类Dimension clusters和直接位置预测Direct location prediction的方法，使 YOLO 比其他使用 Anchor Box 的版本提高了近5％。

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190604221336.png)

>  别处说：从第四行可以看出，**anchor机制只是试验性在yolo_v2上铺设，一旦有了dimension priors就把anchor抛弃了**。最后达到78.6mAP的成熟模型上也没用anchor boxes。
>
>  没有anchor哪来的pw？

### 2.1.6 Fine-Grained Features

经YOLOv1修改后的YOLOv2，在13×13的特征图上进行预测（YOLOv1是7×7）。虽然这对大物体检测来说用不着这么细粒度的特征图，但他更多的是使用细粒度特征对定位小物体有好处。

Faster-RCNN、SSD 都使用了多尺寸的特征图来进行预测，以获得不同的分辨率。而YOLOv2仅仅在26×26的特征层上添加一个==passthrough层==。这个层像之前ResNet中的identity mappings一样，堆叠concat（联系）高分辨率特征与低分辨率特征，stacking adjacent features into different channels instead of spatial locations。这使 26×26×512 特征层变成 13×13×2048，然后就可以堆叠到后一层特征图上了。

> passthrough层与ResNet网络的shortcut类似，以前面更高分辨率的特征图为输入，然后将其连接到后面的低分辨率特征图上。前面的特征图维度是后面的特征图的2倍，passthrough层抽取前面层的每个 $2\times2$的局部区域，然后将其转化为channel维度，对于 $26\times26\times512$的特征图，经passthrough层处理之后就变成了 $13\times13\times2048$的新特征图（特征图大小降低4倍，而channles增加4倍），这样就可以与后面的$13\times13\times1024$（最后一层）  特征图连接在一起形成 $13\times13\times3072$ 大小的特征图，然后在此特征图基础上卷积做预测。在YOLO的C源码中，passthrough层称为[reorg layer](https://github.com/pjreddie/darknet/blob/master/src/reorg_layer.c)。在TensorFlow中，可以使用[tf.extract_image_patches](www.tensorflow.org/api_docs/python/tf/extract_image_patches)或者[tf.space_to_depth](www.tensorflow.org/api_docs/python/tf/space_to_depth)来实现passthrough层：
>
> ```python
> out = tf.extract_image_patches(in, [1, stride, stride, 1], [1, stride, stride, 1], [1,1,1,1], padding="VALID")
> // or use tf.space_to_depth
> out = tf.space_to_depth(in, 2)
> ```

![img](https://pic3.zhimg.com/80/v2-c94c787a81c1216d8963f7c173c6f086_hd.jpg)

> 另外，作者在后期的实现中借鉴了ResNet网络，不是直接对高分辨特征图处理，而是增加了一个中间卷积层，先采用64个 $1\times1$卷积核进行卷积，然后再进行passthrough处理，这样  $26\times26\times512$的特征图得到 $$13\times13\times1024$$ 的特征图。这算是实现上的一个小细节。使用Fine-Grained Features之后YOLOv2的性能有1%的提升。

### 2.1.7 Multi-Scale Training

作者**希望 YOLOv2 能健壮地运行于不同尺寸的图片之上**，所以把这一想法用于训练模型中。 简单地说就是输入大小的图像不一样来训练。检测数据集fine tune时候才这么做，预训练不改变。


区别于之前固定图片尺寸的做法，YOLOv2 每迭代几次都会改变网络，**修改最后检测层的处理**。每 10 个 Batch，网络会随机地选择一个新的图片尺寸。由于使用了下采样参数是  32，所以不同的尺寸大小也选择为 32 的倍数 {320，352…..608}（分别是10，11...19倍），最小 320×320，最大 608×608，网络会自动改变尺寸，并继续训练的过程。


这一策略让网络在不同的输入尺寸上都能达到一个很好的预测效果，同一网络能在不同分辨率上进行检测。当输入图片尺寸比较小的时候跑的比较快，输入图片尺寸比较大的时候精度高，所以你可以在 YOLOv2 的速度和精度上进行权衡。

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190602230939.png)

输入分辨率高的时候准确率高

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190602231747.png)



## 2.2 Faster

一张224×224的图片通过VGG16要经过306.9亿浮点运算，为了更快，网络没必要像VGG16这么复杂。

YOLOv2用基于GoogleNet的自定义的网络，一次前向传播只要85.2亿次浮点运算，虽然准确率较VGG16有所下降，在ImageNet上从90%下降到了88%。

### 2.2.1 Darknet-19

也是用3×3的卷积，池化后的通道数也变为2倍。像Network-in-Network NIN一样，采用全局平均池化，在3×3卷积直接用1×1的过滤器压缩特征表达。用了BN加速收敛，稳定训练，正则化模型。

因为有19个卷积层和5个最大池化，所以叫Darknet-19。表6即为模型。处理一张图片只需要55.8亿次运算，在Imagenet上72.9%的top-1准确率，91.2%的top-5.

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190602232958.png)

这个网络包含19个卷积层和5个max pooling层，而在YOLO v1中采用的GooleNet，包含24个卷积层和2个全连接层，因此Darknet-19整体上卷积卷积操作比YOLO v1中用的GoogleNet要少，这是计算量减少的关键。最后用average pooling层代替全连接层进行预测。这个网络在ImageNet上取得了top-5的91.2%的准确率。

### 2.2.2 Training for classification预训练

 training for classification都是在ImageNet上进行预训练，主要分两步：

从头开始在ImageNet1000上预训练160epoch，**此时的输入图像大小是224×224**，初始学习率0.1，权重衰减0.0005，动量0.9。数据增强方式包括随机裁剪，旋转以及色度，亮度的调整等。

在224×224上预训练后，**再拿448×448的输入训练**，训练参数与上相同，不同处在于10个epoch，初始学习率为0.001。

结果表明fine-tuning后的top-1准确率为76.5%，top-5准确率为93.3%。

而原来的训练方式，Darknet-19的top-1准确率是72.9%，top-5准确率为91.2%。因此可以看出第1,2两步分别从网络结构和训练方式两方面入手提高了主网络的分类准确率。


### 2.2.3 Training for detection微调训练

之前training for classification的操作相当于是预训练，只是简单地进行分类，没有检测的过程。所以迁移到检测的网络上时需要对网络做一些修改，而保持主题框架的权重不变，这使网络对物体已经有了一定的敏感程度。

所以此处**去掉最后一个卷积，池化层和softmax层**，

![img](https://pic4.zhimg.com/80/v2-b23fdd08f65266f7af640c1d3d00c05f_hd.jpg)

新增3个3×3×1024的卷积，再加一个1×1卷积。输入的维度为anchors×（5+classes）。还加了一个passthrouth层，起始点为3×3×512层的最后一层到倒数第二层。

对于VOC数据，由于每个grid cell我们需要预测5个box，每个box有5个坐标值和20个类别值，所以每个grid cell有125个filter。有5个预测框anchor，每个预测框有5个坐标t和20类，所以一共有5×（5+20）=125个通道。

T=(batchsize,13,13,125)reshape成(batchsize,13,13,5,25)。所以T[...,0:4]为边界框的位置和大小，T...,4]为对应的置信度。T[...,5：]为该框对应的预测值。

匹配原则，对于某个ground truth，首先要确定其中心点要落在哪个cell上，然后计算这个cell的5个先验框与ground truth的IOU值（YOLOv2中bias_match=1），计算IOU值时不考虑坐标，只考虑形状（即只考虑wh），所以先将先验框与ground truth的中心点都偏移到同一位置（原点），然后计算出对应的IOU值，IOU值最大的那个先验框与ground truth匹配，对应的预测框用来预测这个ground truth，其他4个不与该gt匹配。

> YOLOv1中采用的是平方根以降低boxes的大小对误差的影响，而YOLOv2是直接计算，但是根据ground truth的大小对权重系数进行修正：l.coord_scale * (2 - truth.w*truth.h)（这里w和h都归一化到(0,1))，这样对于尺度较小的boxes其权重系数会更大一些，可以放大误差，起到和YOLOv1计算平方根相似的效果（参考[YOLO v2 损失函数源码分析](https://link.zhihu.com/?target=http%3A//www.cnblogs.com/YiXiaoZhou/p/7429481.html)）。
>
> YOLOv2的官方训练权重文件转换了TensorFlow的checkpoint文件（[下载链接](https://link.zhihu.com/?target=https%3A//pan.baidu.com/s/1ZeT5HerjQxyUZ_L9d3X52w)）
>

这里训练的参数如下：160个epoch，学习率0.001，并且在第60和90epoch的时候将学习率除以10，weight decay采用0.0005，动量0.9。也有数据增强的内容：crop，color shifting等


## 2.3 Stronger

这里只是为了可以预测9000个类，简略看看即可。总结一句话就是这么多类其实是一个树，根节点是物体的概率。树下结点的概率是每个类的概率，但是这个概率是在上一层数概率的基础上，即条件概率。因为是条件概率，所以任一节点下的同层子节点可以用softmax函数归一化。如果要算某个结点类的概率，即该结点到根节点所有概率值相乘，这也是和条件概率公式符合的。

检测数据集只有普通的标签，如“猫”，“狗”。但是分类数据集有更具体的标签，如把狗细分为”哈士奇“，”牛头梗“，”金毛狗“等。如果想在两个数据集上训练，需要一个一致性的方法融合这些标签。

众多分类方法采用了softmax进行计算概率分布。用了softmax是假设了各个类别是独立的。你不想盲目地混合数据集，因为把ImageNet中的金毛狗和COCO里的狗当做不同类别是不合适的。

我们采用多标签模型取组合这些数据集，并且假设类别不是独立的，假定一张图片可以由多个分类。

### 2.3.1 Hierarchical classification

ImageNet的标签名字是来自于WordNet，这是一个语言库。

如 “Norfolk terrier” 和“Yorkshire terrier” 都是 “terrier” 活泼的狗的“下义词”，而他是猎犬的一种，而猎犬是狗dog的一种，dog是犬科canine的一种。

WordNet 的结构是一种直接图表directed graph，而不是树的一种，因为语言是很复杂的。比如狗同时是犬科和家畜的子集。但没有使用全图，而是建立了等级树去简化问题。

为了建立这个等级树（分层树），首先检查 ImagenNet 中出现的名词，再在 WordNet 中找到这些名词，再找到这些名词到达他们根节点的路径（在这里设所有的根节点为实体对象（physical object）。在 WordNet 中，大多数同义词只有一个路径，所以首先把这写路径中的词全部都加到分层树中。接着迭代地检查剩下的名词，把他们添加到分层树上，但要让树成长尽可能少。比如由两条路径到底根节点，那么一条路径添加了3条边，而另一条仅添加了1条边，添加的原则是取最短路径加入到树中。

最终结果是一个WordTree。在每个结点上预测了每个下义词的条件概率，比如下"terrier"这个结点上我们预测：

Pr(Norfolk terrierjterrier)
Pr(Yorkshire terrierjterrier)
Pr(Bedlington terrierjterrier) 

....

如果我们想要计算一个特定结点的绝对概率，我们仅需要简单地沿着该结点要根结点的路径，相乘，即可得到绝对该。所以，比如我们想要知道Norfolk terrier  的概率时，我们可以如下计算

Pr(Norfolk terrier) = Pr(Norfolk terrierjterrier)
	∗Pr(terrierjhunting dog)
	∗ ... ∗
	∗Pr(mammaljP r(animal)
	∗Pr(animaljphysical object) 

对于分类，我们假定了图像包含一个物体，Pr(physical object) = 1

为了沿着这个方法，我们用ImageNet-1000在Darknet-19模型上训练。为了建立 WordtTree 1K，把所有中间词汇加入到 WordTree 上，把标签空间从 1000 扩大到了 1369。在训练过程中，我们传播gt类标，如果有一个图片的标签是“Norfolk terrier”，那么这个图片还会获得”狗“（dog）以及“哺乳动物”（mammal）等标签。总之现在一张图片是多标记的，标记之间不需要相互独立。

为了计算条件概率，我们模型预测一个1369个值的向量。之前的 ImageNet 分类是使用一个大 softmax 进行分类。而现在，WordTree 只需要对同一概念下的同义词进行 softmax 分类。 

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190603204531.png)

这种方法的好处：在对未知或者新的物体进行分类时，性能降低的很优雅（gracefully）。比如看到一个狗的照片，但不知道是哪种种类的狗，那么就高置信度（confidence）预测是”狗“，而其他狗的种类的同义词如”哈士奇“”牛头梗“”金毛“等这些则低置信度。

### 2.3.2 Datasets combination with wordtree

用 WordTree 把数据集合中的类别映射到分层树中的同义词上，例如上图 Figure 6，WordTree 混合 ImageNet 与 COCO。

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190603211104.png)

### 2.3.3 Joint classification and detection

作者的目的是：训练一个 Extremely Large Scale 检测器。所以训练的时候使用 WordTree 混合了 COCO 检测数据集与 ImageNet 中的 Top9000 类，混合后的数据集对应的 WordTree 有 9418 个类。另一方面，由于 ImageNet 数据集太大了，作者为了平衡一下两个数据集之间的数据量，通过过采样（oversampling） COCO 数据集中的数据，使 COCO 数据集与 ImageNet 数据集之间的数据量比例达到 1：4。

YOLO9000 的训练基于 YOLO v2 的构架，但是使用 3 priors 而不是 5 来限制输出的大小。当网络遇到检测数据集中的图片时则正常地反方向传播，当遇到分类数据集图片的时候，只使用分类的 loss 功能进行反向传播。同时作者假设 IOU 最少为 0.3。最后根据这些假设进行反向传播。

使用联合训练法，YOLO9000 使用 COCO 检测数据集学习检测图片中的物体的位置，使用 ImageNet 分类数据集学习如何对大量的类别中进行分类。 


为了评估这一方法，使用 ImageNet Detection Task 对训练结果进行评估。 

评估结果： 

- YOLO9000 取得 19.7 mAP。 在未学习过的 156 个分类数据上进行测试， mAP 达到 16.0。
- YOLO9000 的 mAP 比 DPM 高，而且 YOLO 有更多先进的特征，YOLO9000 是用部分监督的方式在不同训练集上进行训练，同时还能检测 9000个物体类别，并保证实时运行。

# YOLOv3

## 3.1 YOLOv3与YOLOv2比较

### 3.1.1 与YOLOv2相同的地方

- 框的表示方式：聚类anchor，预测的是t
- 激活函数leakly relu
- 端到端训练，一个loss函数
- batch normalization+leakly relu接在每层卷积之后
- 多尺度训练

### 3.1.2 YOLOv3与YOLOv2显著区别

- darknet-19改成darknet-53。后者还提供了tiny darknet版本，想要速度快就改用tiny darknet

- 类FPN结构：输出3个尺度的feature map，多尺度预测，每个尺度3个prior。prior聚类结果

  `10,13,  16,30,  33,23;  30,61,  62,45,  59,119;  116,90,  156,198,  373,326`

- 残差结构，v2没有

## 3.2 分类网络 darknet-53

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190603220619.png)

步长为2的卷积也算入其中：53=1+1+ 1×2+1 + 2×2+1 + 8×2+1 + 8×2+1 + 4×2 + 1。最后的1可能是把FC算进去了，在检测网络中会去掉变成darknet-52，但仍叫darknet-53

**没有池化层**，池化层是通过步长为2的3×3卷积代替的。5次步长为2的卷积，与YOLOv2一样缩放了32倍，所以输入尺寸得是32的倍数。416/32=13

表一网络中有些内容没有体现，比如Batch Normalization和leakly relu，和YOLOv2一样每个卷积后面都有着两个步骤。所以我们把这3个内容记为一个基本单元：DBL=卷积conv+BN+激活函数Leakly relu。我们记做大写CONV，与conv相区别。CONV中的卷积都是3×3的，步长为1或2。conv是1×1的，步长为1。

框内的内容为类似resnet的内容。8×代表重复8次，有8次res连接，res连接的具体内容为先用CONV1×1卷积降低通道数，再用CONV3×3通道数恢复原来的通道数，以减少一些浮点数运算，再与这两个卷积之前的特征图对应元素相加。除了输入大小外，在每个stage（即每种大小的特征图）上都有res模块，所以我们把步长为2的类pooling的卷积+res模块连在一起，记为resn，n代表pooling后又几个res模块，即表1中的1.2.8.8.4。可参照下图理解

## 3.3 检测网络

检测网络就比原来分类网络darknet多了个类FPN结构。FPN解读：https://blog.csdn.net/hancoder/article/details/89048870

![NETRON](https://img-blog.csdn.net/2018100917221176?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xldmlvcGt1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

类FPN结构只用了3个stage的特征图。

y1：darknet-53输入后+（5个3×3CONV）+（1个3×3CONV+1×1conv）

y2：上采样y1过程中的（darknet-53输入后+5个3×3CONV），上采样填充的元素是通道中同一位置的元素，即上采样2倍后通道数会缩小2×2=4倍。然后与上个stage的最后一层输出concat，即特征图同样大小的情况下在通道方向上拼接，通道数增大。拼接后同y1的步骤一样，再通过（5个3×3CONV）+（1个3×3CONV+1×1conv）

y3：同理，再次说明上采样的地方是深层特征图的5个3×3CONV后。concat后同样再（5个3×3CONV）+（1个3×3CONV+1×1conv）

![](https://raw.githubusercontent.com/FermHan/tuchuang/master/20190606202757.png)

- concat类似于python中的tf.concat和torch.cat

- Total252=add23+BN72+leaklyrelu72+convat2+conv75+UpSampling2+ZeroPadding5

- 当输入为416×416时，3个输出的通道数都是255=3×（5+80），3个输出的特征图边长分别为13,26,52

- https://github.com/marvis/pytorch-yolo3 该界面有具体每层输入输出

## 3.4 实验效果

YOLOv3 在 Pascal Titan X 上处理 608x608 图像速度可以达到 20FPS，在 COCO test-dev 上 mAP@0.5 达到 57.9%，与RetinaNet（FocalLoss论文所提出的单阶段网络）的结果相近，并且速度快 4 倍.

> YOLOv3 不使用 Softmax 对每个框进行分类，主要考虑因素有：
>
> 1. Softmax 使得每个框分配一个类别（得分最高的一个），而对于 Open Images这种数据集，目标可能有重叠的类别标签，因此 Softmax不适用于多标签分类。
> 2. Softmax 可被独立的多个 logistic 分类器替代，且准确率不会下降。 
> 3. 分类损失采用 binary cross-entropy loss.
>
> 使用softmax强行让每个框有1个类，这通常是不符生活经验的

3.5参考

https://zhuanlan.zhihu.com/p/35325884

