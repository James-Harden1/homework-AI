# 点云高效表征学习



### 是什么

点云是一种数据，它是数据表示在三维空间的形式。它通过大量离散的三维坐标点（*x，y，z*）部分点还包含附加信息（如颜色RGB、反射强度、法向量等）集合来描述物体或场景的表面形态。 



### 获取方式

它有传感器采集和算法重建两种生成与获取方式，前者典型代表是激光雷达，是我们后面要讲的自动驾驶中点云数据的收集方式。后者一般是通过扫描得到图像的点云数据集。



### ·处理点云数据的现实意义

点云数据一般比较无序，稀疏，处理起来比较复杂，而点云高效表征学习，就是用例如DGCNN和可变形卷积等新方法处理点云数据，提取有效特征，让之后训练模型与实际应用的效果更好。而处理好点云数据，对自动驾驶和工业零件的检测有重大意义和应用价值。



## 1. 局部几何特征聚合

而实现点云高效表征学习所需的策略是局部几何特征聚合。如何理解呢：想象你要描述一个复杂的物体（比如一辆自行车），但不需要描述整个自行车，而是先观察它的局部零件（车轮、车架、脚踏板），然后把这些零件的特征组合起来，最终得到整辆车的完整描述。这就是"局部几何特征聚合"的核心思想。

![image-20250406222020857](C:\Users\20786\AppData\Roaming\Typora\typora-user-images\image-20250406222020857.png)





> 我们先把目标物体表面扫描出点云，再把整个点云图划分为小区域（用立体图形划分），每个区域几十到几百个点，然后我们把局部几何特征如（法线方向、曲率、密度）用PointNet网络提取出一个抽象特征，最终我们把局部特征整合成整体特征。
>
> 我们为什么要做聚合？1. 局部信息不稳定单个点的特征容易受噪声干扰（比如3D扫描中的误差） 2. 全局信息太粗糙直接看整体会忽略细节（比如分辨不出车轮和车架的区别）3. 通过聚合达到平衡（既保留细节，又消除噪声），从而得到一个更全面、更稳定的整体特征。



### 而把多个局部特征合并成整体特征有以下三个常见方法：

1. **最大池化（Max Pooling）**：取每个特征维度的最大值（保留最显著特征，但有可能丢失细节）

2. **平均池化（Avg Pooling）**：取特征维度的平均值（平滑特征，弱化突出特征）【注意池化也可以混合起来使用，平衡提取综合特征（找一个平衡点，看需求，就是说可以优化）】

3. **注意力（attention）聚合**：给不同局部区域分配不同权重（智能选择重要特征，但计算量大）

   类似于词向量的注意力机制，把每个区域与其他区域做矩阵点积计算。这样计算后，可以得到相对于整体而言（自注意力），更具关联度（重要性）的区域分配较高的概率，再把带有权重的区域拼接起来，更新整体的特征。



### 总而言之，局部几何特征聚合处理点云有以下特性：

局部性：聚焦小范围，避免被全局噪声干扰

层次性：像拼图一样，先拼局部再组合整体

鲁棒性：即使某些局部损坏（如遮挡），其他局部仍能提供信息



## 2. DGCNN

DGCNN（动态图卷积神经网络）是一种基于动态图结构提出的深度学习模型。它擅长从无序的三维点云数据（如激光雷达扫描的环境点云）中发现物体的局部特征和整体结构。

DGCNN有多层网络，每层网络中会先根据当前点的特征（坐标和语义信息）：浅层中会关注多一点几何距离，而深层会着重关注语义关联度。根据这些特征，构建新的关系网，小区域，从而实现动态图的构建，让局部几何特征划分区域变得更合理。

![image-20250406222053934](C:\Users\20786\AppData\Roaming\Typora\typora-user-images\image-20250406222053934.png)

而如何让提取特征也变得更准确？这就需要用到EdgeConv边卷积。它先对每个点及其邻居点计算“边特征”，比如坐标差、颜色差异等，然后通过多层感知机（MLP）提取信息。使用最大池化（类似“投票”）选出邻居中最显著的特征，作为当前点的新特征。注意这里边卷积也有很多层，逐步提取小区域里的特征。

![image-20250406222124738](C:\Users\20786\AppData\Roaming\Typora\typora-user-images\image-20250406222124738.png)

### 应用于自动驾驶

#### 点云分割任务

目标：将点云中的每个点分类到特定类别（如车辆、行人、道路）。  
挑战：点云无序、稀疏、噪声多，需同时捕捉局部细节和全局语义。

自动驾驶中要实时做点云分割：时刻更新动态图，区分清楚道路上的不同物体（路障，行人）



#### DGCNN在自动驾驶这个领域的优势：

可以准确高效分割出图像的边缘划分

1. 动态局部感知 通过动态图捕捉不同层次的局部结构（如车轮的圆形边缘、车身的平面）

2. 多层特征融合 将浅层几何特征与深层语义特征拼接，提升分割精度

3. 鲁棒性 对点云密度变化不敏感，适合复杂场景，如高速移动场景，点云较稀疏，颜色较复杂。



## 3. 可形变卷积

· 为什么要改进：传统卷积的局限性

固定感受 3×3卷积核只能覆盖固定的9个位置（像一个僵硬的格子）。

不适应形变 无法处理物体拉伸、旋转或局部变形（如弯曲的裂纹、不规则的零件缺陷）。

可变形卷积的改进

- **动态偏移量**：为每个卷积核位置学习一个偏移量（Δx, Δy），让采样点自由移动。

- **自适应形状**：根据图像内容调整采样点位置（例如弯曲到裂纹边缘）。



可形变卷积的**关键**就是学习一个偏移量让卷积核可以自适应目标的几何形变（如边缘区域）为什么要改变传统的卷积（CNN）？原来的卷积核是定死的，无法拟合不规则形状。而可形变卷积有一个小卷积层，负责生成偏移量，而卷积核这时根据偏移量不规则划过图像，增强模型对目标几何形变（如旋转、缩放、遮挡）的建模能力。

![image-20250406222152345](C:\Users\20786\AppData\Roaming\Typora\typora-user-images\image-20250406222152345.png)



这个小卷积层类似于一个独立的模型（奖励模型--损失函数优化），它决定偏移量，让模型识别出完整的划痕以及边缘关键信息。先用大量带裂纹的工业零件训练它，让他学习。让他有通过小样本数据预训练临时调整偏移量，以用于检测目标批次工业零件。【这个过程是动态的，由数据驱动】



· 值得注意的是DGCNN也可以用于检测工业零件缺陷。总的来说，DGCNN是为应对点云动态图（三维空间）而对卷积的结构上优化，以达成更准确分类，收集数据，特征的目的。而可变形卷积就是简单的添加了一个小卷积层，通过让卷积核多一个不定的偏移量（X，y）对功能上的优化。本质上二者都是卷积的变体，而卷积本身就是对空间的细化，对数据细节的放大，再把特征聚合——就是“局部几何特征聚合”的思想。

### 数据对于深度学习模型的重要性

高效学习点云数据，可以帮助我们准确抓取真实三维世界里的图像特征，有效地传达给机器模型，让机器对真实世界的理解更加准确。而数据的更精细准确，可以提高预训练，训练效果，提升模型预测以及完成生成性任务的能力。