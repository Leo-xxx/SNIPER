# 物体检测之SNIPER

[![刘岩](https://pic2.zhimg.com/v2-1bd0072f46c5fd2f79653d2f2fae64f1_xs.jpg)](https://www.zhihu.com/people/yan-liu-43)

[刘岩](https://www.zhihu.com/people/yan-liu-43)

算法菜鸡

关注他

15 人赞同了该文章

图像金字塔是传统的提升物体检测精度的策略之一，其能提升精度的一个原因是尺寸多样性的引入。但是图像金字塔也有一个非常严重的缺点：即增加了模型的计算量，一个经过3个尺度放大（1x,2x,3x）的图像金子塔要多处理14倍的像素点。

SNIPER（Scale Normalization for Image Pyramid with Efficient Resampling）的提出动机便是解决图像金字塔计算量大的问题，图像金字塔存在一个固有的问题：对于一张放大之后的高分辨率的图片，其物体也随之放大，有时甚至待检测物体的尺寸超过了特征向量的感受野，这时候对大尺寸物体的检测便很难做到精确，因为这时候已经没有合适的特征向量了；对于一个缩小了若干倍的图像中的一个小尺寸物体，由于网络结构中降采样的存在，此时也很难找到合适的特征向量用于该物体的检测。因此作者在之前的SNIP[2]论文中便提出了高分辨率图像中的大尺寸物体和低分辨率图像中的小尺寸物体时应该忽略的。

为了阐述SNIPER的提出动机，作者用很大的篇幅来对比R-CNN和Fast R-CNN的异同，一个重要的观点就是R-CNN具有尺度不变性，而Fast R-CNN不具有该特征。R-CNN先通过Selective Search选取候选区域，然后无论候选区域的尺寸是多少都会将其归一化到 ![[公式]](https://www.zhihu.com/equation?tex=224%5Ctimes224) 的尺寸，该策略虽然备受诟病，但是它却保证了R-CNN具有尺度不变性的特征。Fast R-CNN将resize的部分移到了使用了卷积之后，也就是使用RoI池化产生长度固定的特征向量，也就是说Fast R-CNN是将原始图像作为输入，无论里面的尺寸大小，均使用相同的卷积核计算特征向量。但是这样做真的合理吗？不同尺寸的待检测物体使用相同的卷积核来训练真的好吗？答案当然是否定的。

SNIPER策略最重要的贡献是提出了尺寸固定（ ![[公式]](https://www.zhihu.com/equation?tex=512+%5Ctimes+512) ）的chips的东西，chips是图像金子塔中某个尺度图像的一个子图。chips有正负之分，其中正chip中包含了很多尺寸合适的标注样本，而负chip则不包含标注样本或者如第二段中所阐述的包含的样本不适合这个尺度的图像。

需要注意的是SNIPER只是一个采样策略，是对图像金字塔的替代。在论文中作者将SNIPER应用到了Faster R-CNN和Mask R-CNN中，同理，SNIPER也几乎可以应用到其它任何检测甚至识别算法中。

作者已经开源了基于MXNet的[源码](https://link.zhihu.com/?target=https%3A//github.com/mahyarnajibi/SNIPER)，我们在后面的分析中会结合源码进行讲解。

------

## 1. SNIPER 详解

作为一个采样策略，SNIPER的输入数据是原始的数据集，输出的是在图像上采样得到的子图（chips），如图1的虚线部分所示，而这些chips会直接作为s。当然，chips上的物体的Ground Truth也需要针对性的修改。那么这些chips是怎么计算的呢，下面我们详细分析之。

![img](https://pic1.zhimg.com/80/v2-3532bd9f88940a7f3a59b94c380ee728_hd.jpg)图1：SNIPER的chips示意图

## 1.1 Chips生成

按照图像金字塔的思想，一个原始输入图片会通过构建多个尺度 ![[公式]](https://www.zhihu.com/equation?tex=%5C%7Bs_1%2Cs_2%2C+...%2C+s_n%5C%7D) 的形式图像金字塔，源码中使用的是3个尺度(3.0, 1.667, 512.0/ms)，分别表示把图像放大3倍，1.667倍以及最大边长固定位512。

在图像金字塔的某个的图像上（假设大小为 ![[公式]](https://www.zhihu.com/equation?tex=W_i%5Ctimes+H_i) ）通过步长为32的滑窗的方法得到约 ![[公式]](https://www.zhihu.com/equation?tex=%5Cfrac%7BW_i%7D%7B32%7D+%5Ctimes+%5Cfrac%7BH_i%7D%7B32%7D) 个 ![[公式]](https://www.zhihu.com/equation?tex=512%5Ctimes512) 的chip。

那么整个图像金字塔的chips的总数约为：

![[公式]](https://www.zhihu.com/equation?tex=+%5Csum_i%5En+%5Cfrac%7BW_i%7D%7B32%7D%5Ctimes%5Cfrac%7BH_i%7D%7B32%7D+%5Ctag1)

## 1.2 正chips

对于1.1部分提到的 ![[公式]](https://www.zhihu.com/equation?tex=n) 个尺度，图像的Ground Truth也会进行对应的放大或者缩小。对于每个尺度，它都有更合适的检测框。论文中给出的判断标准是对于三个尺度，它们理想的检测区域的面积介于 ![[公式]](https://www.zhihu.com/equation?tex=%5Cmathcal%7BR%7D%5Ei+%3D+%5Br_%7Bmax%7D%5Ei%2C+r_%7Bmin%7D%5Ei%5D) 之间（用换算之前的Ground Truth计算面积），源码中给出三个尺度对应的检测面积的区域依次是 ![[公式]](https://www.zhihu.com/equation?tex=%280%2C+80%5E2%29%2C+%2832%5E2%2C+150%5E2%29%2C+%28120%5E2%2C+inf%29) 。

如果一个Ground Truth完全位于一个chip内，那么我们就说这个Ground Truth被这个chip包围（Covered）。在实际采样中，处于计算量和样本平衡的角度考虑，所有的chip不可能都被采样到。在论文中，作者使用了贪心的策略根据chip包围的Ground Truth的数量的多少，从每个尺度中抽取了前 ![[公式]](https://www.zhihu.com/equation?tex=K) 个chip作为正chip，记做 ![[公式]](https://www.zhihu.com/equation?tex=C_%7Bpos%7D%5Ei) 。

最后在放大或者缩小的图像中，将采样的chips裁剪出来，得到一系列大小固定（ ![[公式]](https://www.zhihu.com/equation?tex=512%5Ctimes512) ）的子图，这些子图将作为后续检测算法的训练样本。由于得到的chip的尺寸比较小，且大多数没用的背景都都被忽略了（思想类似于Attention），这对训练的速度的提升是非常有帮助的，同时每个chip只包含合适尺寸的检测物体，这使得模型更加容易收敛。

因为多尺度的chips之间是互相覆盖的，所以可以保证了一个Ground Truth至少被一个Chip采样得到，一个Ground Truth既可以被不同尺度的chips所共同包围，也可以被相同尺度的不同chips所共同包围。

图2中得到的chips便是从图1中的虚线部分裁剪出来的。如图2所示，绿色的Ground Truth代表的是和该chips匹配的待检测物体，而红色的Ground Truth由于面积不在范围内，因此不会被标注出来，在检测的时候等同的看做背景区域。

![img](https://pic4.zhimg.com/80/v2-c3bac2d8c537c5d353c49d4ddf220a3f_hd.jpg)图2：由图1得到的chips

需要注意一点，源码中生成chips的方式并不是之前所说的先生成图像金字塔，再从图像金字塔中裁剪出 ![[公式]](https://www.zhihu.com/equation?tex=512%5Ctimes512) 的chips。源码中采用的方式是从原图中采样出等比例的chips，再将其resize到 ![[公式]](https://www.zhihu.com/equation?tex=512%5Ctimes512) 。这两种策略是等价的，但是第二种无疑实现起来更为简单。

## 1.3 负chips

如果只用正chips作为样本进行训练，模型的假正利率往往很高（不包含任何物体的chip判断为包含包含物体的chip）。具体原因是因为参与模型训练的数据都是包围着Ground Truth的chips，而在测试的时候（第2节详细介绍SNIPER的测试部分），输入的是整张图像的图像金字塔，这时候必然包含不包围任何Ground Truth的背景区域，也就是说训练集和测试集的分布是不一样的。

为了弥补训练集合测试集之间的分布差距，SNIPER提出了从图像金字塔中采样出一批 ![[公式]](https://www.zhihu.com/equation?tex=512%5Ctimes512) 负chips，它们将作为训练数据共同参与模型的训练。所谓负chips，是指不包含Ground Truth或者包含的Ground Truth比较少的chip。

论文中给出的策略是首先只使用正chip训练一个只有几个Epoch的弱RPN。在这里我们对RPN的精度并没有特别高的要求，因为它只是我们用来选择chip的一个工具，对最终的结果影响十分微弱。尽管这个RPN检测能力很弱，但是其并不是随机初始化的一个模型，它得到的检测框还是有一定的置信度的。所以策略的第二步是根据弱RPN的检测结果选择那些“假正“的样本。详细的说，首先去掉 ![[公式]](https://www.zhihu.com/equation?tex=C_%7Bpos%7D%5Ei) 中的正chips，然后根据弱RPN的检测结果，从每个尺度选择至少包含 ![[公式]](https://www.zhihu.com/equation?tex=M) 个候选区域的chips组成负chips池，最后在训练的时候从中随机选择固定数量的组成训练的负样本，表示为 ![[公式]](https://www.zhihu.com/equation?tex=%5Ccup_%7Bi%3D1%7D%5En+C_%7Bneg%7D%5Ei) 。

如图3所示，上排绿色部分表示Ground Truth，下排红色区域表示弱RPN预测的候选区域，橙色区域表示根据弱RPN得到的负chips。

![img](https://pic2.zhimg.com/80/v2-585aedf3aec09c9b9102af30a161f5c1_hd.jpg)图3：SNIPER中的负chips

## 1.4 SNIPER的训练

通过上面的分析，我们可以根据Ground Truth和弱RPN得到一批正chips和一批负chips，这批chips将作为训练样本直接输入给检测算法。SNIPER采用了Faster R-CNN作为这些chips的检测算法框架，并且SNIPER后面模型训练的细节也基本和Faster R-CNN相同，除了一点，SNIPER接的Faster R-CNN的RPN和Fast R-CNN使用了不同的标签系统。

[Faster R-CNN](https://zhuanlan.zhihu.com/p/42741973)[3]系列论文中我们讲过，Faster R-CNN是由RPN和Fast R-CNN组成的多任务模型，在SNIPER中，两个任务会使用两个不同的标签，首先训练RPN时，chips中的待检测物体的Ground Truth并不会受范围 ![[公式]](https://www.zhihu.com/equation?tex=%5Cmathcal%7BR%7D) 的限制。但是再根据RPN提取的候选区域训练Fast R-CNN时，在范围 ![[公式]](https://www.zhihu.com/equation?tex=%5Cmathcal%7BR%7D) 之外的候选区域并不会参与Fast R-CNN的训练。这么做的原因作者没有指出，猜测是RPN需要检测的候选区域覆盖范围更广，因此需要更多的，范围更大的Ground Truth。而Fast R-CNN需要检测的更准确，因此把这个任务交给了能提取到更合适的特征向量的对应尺度。

## 2. SNIPER的测试过程

SNIPER的测试过程有一个致命的缺点，它必须要求输入的图像是图像金字塔，因为它参与训练的Fast R-CNN的chip在范围$$\mathcal{R}$$之外已经过滤掉了，因此模型并不擅长检测不在这个范围内的物体，也就是如果只使用单尺度的话，那些太大或者太小的物体都很难被检测到。

SNIPER使用的图像金字塔的尺度依次是 ![[公式]](https://www.zhihu.com/equation?tex=%28480%2C512%29%2C%28800%2C1280%29%2C%281400%2C2000%29) ，其中第一个值表示resize后短边的大小，但是当长边大于第二个值时，应该讲长边固定到第二个值，第一个值随意。

在图像金字塔提取完检测框之后，使用soft-NMS得到最终的候选区域。

## 3. 总结

小物体检测一直是困扰物体检测领域的一个重要难题，传统的图像金字塔式解决该问题的一个常见的传统策略，但是速度太慢，SNIPER的提出便是动机便是解决图像金字塔的速度问题。

需要注意SNIPER并不是一个检测算法，而是对输入图像的一个采样策略，其采样的结果（chips）将作为输入输入到物体检测算法中。

算法虽然使用了RPN，但是并不是离开了RPN就无法工作了，RPN提供了一个提取假正利率的功能，这个可以通过Selective Search或者Edge Box近似替代。

另外，SNIPER仅仅是对训练速度的提升，往往更重要的检测速度并没有提升，反而是模型必须依赖图像金字塔，这反而降低了模型的通用性。

最后，作者开源的源码和论文出入较大，读起来比较费劲，等之后有时间的话再详细学习这份源码。

## Reference

[1] Singh B, Najibi M, Davis L S. SNIPER: Efficient Multi-Scale Training[J]. arXiv preprint arXiv:1805.09300, 2018.

[2] B. Singh and L. S. Davis. An analysis of scale invariance in object detection-snip. CVPR, 2018.

[3] S. Ren, K. He, R. Girshick, and J. Sun. Faster r-cnn: Towards real-time object detection with region proposal networks. In Advances in neural information processing systems, pages 91–99, 2015.

发布于 2018-10-15