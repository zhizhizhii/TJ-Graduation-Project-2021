# MVSNet: Depth Inference for Unstructured Multi-view Stereo 

## 关键词
多视点立体视觉；深度图；深度学习
Multi-view Stereo, Depth Map, Deep Learning 

## 摘要
端到端的深度学习网络对多视点图像进行深度图推断

【主要流程】
1. 用CNN提取深度视觉图像特征
2. 构建3D代价体cost volumn：通过可微单应投影变换 homography warping
3. 使用三维卷积进行正则化，生成原始深度图
4. 通过参考图像进行深度图优化，提升边界区域的准确性
任意使用N张图片（N个视角）进行输入，使用基于方差的成本度量，将多个特征映射到一个代价体中

【结果】
在大型室内数据集DTU上进行训练和测试，只需很少的后处理，不仅在完整度、准确度等方面较之前最好的方法更好，而且运行速度快了好几倍甚至是好几个数量级；也在复杂的室外数据集Tanks and Temples上进行泛化测试，效果同样出色

## 1. 背景介绍
从几张部分重叠的图像中估计场景中的深度是计算机视觉方向几十年来一直努力的方向

【传统方法】
使用手工标注的相似度指标和工程标准计算场景深度，进而重建三维点云（e.g. [12]中的方法），在理想的实验场景中有着比较好的结果，但是存在不少限制：低纹理、镜面和反射区域等导致不完全重建
~在精度上很好，但在完整度上还有很大的提升空间~

【CNN】
卷积神经网络的成功也被逐步应用到三维重建中，理论上，基于学习的方法可以引入更多的全局语义信息，从而得到更加鲁棒的深度图估计
多视点的立体视觉匹配非常适合使用基于CNN的方法，因为图像对预先经过矫正，因此问题可以很好地扩展到像素级的视差估计，已经在双目视觉的匹配中有过很好的尝试

【双目 -> 多目】
如果直接将双目的方法扩展为多目的方法，无法充分利用多视角信息，每次选定图像对生成的点云也很难融合成全局点云

【MVSNet】
一次计算一张深度图，而非每次对于整个3D场景。选取一张参考图像和几张原始图像作为输入，通过网络进行参考图像的深度估计，进而生成深度图
**亮点核心**
1. 可微单应投影变换 differentiable homography warping~，隐式编码图像的特征和相机的几何参数，以构建三维代价体
2. 为适应任意数量的输入图像（任意多视角），提出基于方差的度量 variance-based metric 
，将众多特征映射到单一的代价体中

**同以往算法的两大特性**：
1. 3D代价体是建立在相机截锥上，而不是欧氏空间中的
2. 解藕mvs重建问题成更小的每个视角的深度图估计，从而扩展了进行大规模的重建泛化可能性

## 2. 相关工作
【多视点重建】
根据输出结果的不同，多视点重建方法可分为：
1. 点云重建：通常使用传播策略使重建逐渐稠密；很难并行化，需要大量时间处理
2. 体素重建：将3D空间划分为网格，估计每个体素是否附着在表面；空间离散产生错误，高内存消耗
3. 深度图重建：最灵活的表示方式，将复杂的多视点问题解藕为每一个视角的深度图估计，每次只需要关注一个参考图像和几张原始图像即可；得到的深度图也可以方便的融合成点云和进行体素重建；在mvs基准榜单上，目前结果最好的方法都是采用深度图重建的方式
4. （网格重建）

【基于深度学习的立体匹配】
Han等人第一次提出使用深度网络匹配两个图像对
Zbontar和Luo等人使用基于学习的特征和半全局匹配特征（semi-global matching，SGM）进行立体匹配和后处理
SGMNet动态调整SGM中的参数
GCNet使用3D CNN回归代价体，并使用soft argmin回归损失差距

【基于深度学习的多视点立体匹配】
- SurfaceNet：融合所有像素值和相机参数，构建有色体元数据体（Colored Voxel Cubes，CVC）作为网络的输入；用分治的策略（devide-and-conquer），对于大规模的重建要消耗很长的时间
- LSM：直接利用可区分的投影变换进行训练；只处理低体积分辨率的合成对象
二者都需要消耗巨大的内存用以构建3D体，网络很难扩展

---
## 3. 方法原理
### 图像特征
[image:83AF3659-3844-49CD-B9E1-94742983ADA4-23459-0000FE732FEDCC97/5ABF6B75-20E4-4C12-9C03-B0520566D27E.png]
总体是八层的2D CNN网络
第三层和第六层步长为2，用以将特征金字塔分为三个尺度
每一个尺度有两个卷积层用于提取高层级特征表示，每个卷积层由batch-normalization (BN) layer and a rectified linear unit (ReLU) 组成，最后一层除外
同其他匹配任务一样，参数在特征金字塔全局共享，用以更加高效的学习

2D网络的输出是N维32通道特征图，与输入图像相比，每个维度缩小4
虽然经过特征提取之后图像帧进行了下采样，但剩下像素的原始近邻信息已经被编码进32通道的像素级描述符中，防止密集匹配丢失有用的上下文信息

与直接进行简单密集匹配相比，提取的特征图显著提高了重建的质量

### 代价体
用提取出的特征图和输入相机参数构建3D代价体
之前的方法用标准网格划分空间，我们使用深度图推断，从参考相机的截体中构建代价体

[image:4E1BDDA0-29DE-45C5-8981-3F4D4C547F4A-23459-0000FE7D96F32CFC/9507F463-E53C-410D-BB8C-2056837DCBDB.png]

【可微单应变换投影】
连接二维特征提取和三维正则化网络的核心步骤，投影操作以可微的方式实现，实现了深度图推理的端到端训练

所有特征图都被影射到参考相机的不同前平行平面，以组成N个特征体 {Vi}N
通过投影变换，将某一个深度图以深度d从 Vi(d) 映射到 Fi，Hi(d)为第i张特征图和参考特征图在深度d的投影
n1为参考相机的主轴
[image:806D820F-A4F4-495F-9941-4430B7E8A169-23459-0000FEE055933D0E/B6BBEC55-13F9-4B71-A669-5CAA499B9018.png]

为了不失一般性，对于参考特征图F1本身的单应变换时3*3的单位矩阵
投影过程与经典的平面扫描类似[5]，除了可微的双线性插值时用于从特征图{Fi}N中采样像素，而不是原始图像{Ii}N

【代价指标】
聚集多重深度图{Vi}N为一个代价体C
为了适应任意数量的输入图像，提出了基于方差的代价度量指标M，用以度量N视点的相似性
W，H，D，F分别代表输入图像的宽、高、深度样本数、特征图通道数
V=W/4 * H/4 *D *F 特征体的尺寸
[image:BA0CC927-AB48-4593-B97B-A8935AF819B0-23459-0000FF76C62722CD/111D30E6-9452-4BA6-B8B9-9A093F28C58A.png]
[image:212F9C94-8FFA-4369-BAF7-7C8BA3486AF3-23459-0000FF6B5F7B6920/651B805D-EC94-4CBF-B7AF-9E674C8DCFEF.png]

传统MVS方法通过启发式方法计算参考图像和源图像间的代价，我们认为参考[11]所有视图都应该平等地贡献匹配样本，并且不优先考虑参考图像。但[11]中直接对多个CNN层取平均值来得到多补丁相似度（multi-patch similarity），这里我们选用方差，因为平均操作本身没有提供关于特征差异的信息，并且它们的网络需要预处理和后处理CNN层帮助推断相似性，我们采用方差的低价指标明确地衡量来多视图特征的差异

【代价体正则化】
直接从图像特征得到的代价体可能有很多噪声，例如非朗伯曲面或物体遮挡的存在，因此应该结合平滑约束以推断深度图
正则化将C转换为概率体P
[image:62EA5C54-E358-4361-8E5B-0FEA26F4CFB0-23459-0001000A9924B335/5F7BF7FF-48A0-4E1C-AAB1-41A73F3BFE0A.png]
我们提出了多尺度3D CNN用于代价体的正则化

四尺度网络类似于3D版本的UNet[31]，它使用编码器-解码器结构，以相对较低的内存和计算成本从一个大的感受野聚集邻近信息
为了进一步降低计算需求，在第一层卷积层之后将32通道代价体降低为8通道，将每个尺度内的卷积从3层更改为2层，最后的卷积层输出单通道的代价体。
最后沿深度方向应用softmax运算进行概率归一化

概率体不仅可以用于进行像素级的深度估计，还可用于测量估计的置信度

### 深度图
【初始估计】
从概率体P中检索深度图D最简单的方式是像素级的赢家通吃方法[5]，例如argmax，然而它不能进行亚像素级的估计，由于其不可微性，也不能用于反向传播训练
[image:59EE8451-A843-4E27-BCAF-30375149064F-23459-00010082B02E9CC8/5521A8B4-1994-4633-A6EA-985DB0E71714.png]
我们沿着深度方向计算期望值expectation，例如所有假设概率的加权和
[image:2988611B-88AA-4AE9-9311-6A4A2EE5B8F3-23459-0001009E235A9169/1D9D5532-8614-45A1-A387-6AFD304A2BD2.png]
它的结果与argmax相似，但完全可微

【概率图】
沿深度方向的概率分布也反映了深度估计的质量
但即使对于3D CNN正则化后得到的概率分布也是分散的，不能集中到单峰
基于这个观察，我们定义深度估计的质量d_hat为 估计深度在真值附近的概率
[image:89FEA770-72F4-49EA-AE3E-AC8ABD4D2E99-23459-000100F77E97406F/EB6ABC09-A4AE-4F3E-A48F-E4400867F06E.png]
而且概率方法可以更好的控制离群点滤波时的参数阈值

【深度图优化】
重建的边缘可能由于正则化时采用过大的感受野而过于平滑
可以采用参考图像自身所携带的边缘信息指导深度图优化
在MVSNet之后我们使用了深度剩余学习网络

原始深度图和统一尺寸后的原始图像聚合成四通道输入，然后经过三个32通道的2D卷积层和一个单通道卷积层，学习深度剩余
然后结合原始的深度图生成优化后的深度图

### 损失
结合原始深度图和优化后的深度图
使用深度图的真值和深度图估计值的平均绝对误差作为训练误差
由于深度图的真值在整张图像上不一定始终准确，只考虑有确认标签的像素
[image:5EBC6940-4DA6-4BD6-B7F8-C224BE484041-23459-000101BFB3FA5E64/6E432C4C-47E1-47FE-8EB2-237BCF646410.png]

---
## 4. 方法实现
### 训练
【数据集准备】
DTU只提供了点云和表面格式的真值，所以我们首先使用了SPSR生成每个视角的深度图
选取了和SurfaceNet中完全相同的train validation evaluation数据集
每个扫描场景包含7种不同光照条件的49张图像，将每张图片作为参考，一共有27097训练样本

【视角选择】
N=3，一张参考，两张原始输入图像
> 根据两个视角计算了一个score（这里没看懂，感觉也用不上）
> [image:3F4DE362-78FF-4D06-B0ED-7B76F0D2146A-23459-0001027BE0374477/1BD76E29-EFE0-4DF9-A1D5-E0E92C1DCAD6.png]
由于图像在特征提取时会进行下采样，原始图像尺寸需要能被32整除，考虑到GPU性能，将图像尺寸调整到800*600，并且在中间截取W=640，H=512的区域作为训练输入
深度假设在425mm～935mm间以2mm均匀采样，最终D=256

### 后处理
【深度图滤波】
深度神经网络对每一像素进行深度估计，在转换为点云前需要进行离群值的滤波，尤其是背景区域
- 光度一致性：表征匹配质量
- 几何一致性：表征多视角中的深度一致性
[image:0520670F-9AFD-45C9-93AD-6B8043406FD0-23459-000102EFF4343DF4/F3DA5F31-508D-4F4D-A5E5-FED17FB402C4.png]
将p1通过深度d1投影到另一个点pi，再将pi根据它的深度di投影回来，如果满足如上两个条件，则认为这两个视角满足集合一致性

两步的滤波策略极大增强了算法的鲁棒性，并且滤除了多种类的离群值

【深度图融合】
将多视角深度图融合成点云表示
使用的是[26] 不同视角的遮挡被最小化
> 这一段fusion的介绍感觉也用不上这么详细[image:C8037BCF-B48B-4E69-9903-AA217E4B4E2A-23459-0001034F895BE010/95734DC1-C213-448C-9F25-7121C40042C1.png]


---
## 5. 实验和结果
### DTU数据集上
view number, image width, height and depth sample number 
N = 5, W = 1600, H = 1184 and D = 256 

- 距离度量[1]
- 百分比度量[18]

- 准确性 accuracy
- 完整性 completeness
- f-score：融合二者得到的综合表现
- overall：acc和comp的平均

在最难的纹理缺失和反光的部分，有着其他方法没有的完整重建

### 坦克数据集
N = 5, W = 1920, H = 1056 and D = 256 
相机位姿通过SfM开源软件OpenMVG得到
虽然MVSNet是在完全不同的DTU数据集上进行训练的，但有着极强的泛化性

### 5.3 消融分析
用 验证loss 评估重建质量

【视角数量N】
随着视角的增多，validation loss降低，即使用N=3训练，在N=5时也有很好的验证效果，对于不同视角数量非常灵活

【图像特征】
基于深度学习得到的图像特征可以显著提高重建质量
取代原来的2D特征提取网络，换成单层32-channel卷积层，filter=7*7，stride=4

【代价指标】
variance operation based cost metric有更快的收敛和更低的validation loss

【深度优化】
不影响validation loss很多
from 75.58 to 75.69 (< 1mm f-score) and from 79.98 to 80.25 (< 2mm f-score) 

### 5.4 讨论
【运行时间】
takes around 230 seconds to reconstruct one scan (4.7 seconds per view) 
run time speed ∼ 5× faster than Gipuma, ∼ 100× than COLMAP and ∼ 160× than SurfaceNet 

【GPU消耗】
取决于输入图像尺寸和深度特征数量
可以使用一张消费级的GTX 1080ti（11GB）完成训练和验证

【训练数据】
DTU提供了ground truth
但不可能100%完整和准确，因此，前景后的一些三角形被错误地渲染到深度图中作为有效的像素，这可能会恶化训练过程
如果某个像素在其他所有视角中均被遮挡，它将不会被用来训练，因此无法预测被遮挡点的深度

## 6. 结论
最核心的贡献是 编码相机参数作为可微单应投影并结合图像信息构建代价体，嫁接了2D特征提取和3D损失回归的深度学习神经网络
























