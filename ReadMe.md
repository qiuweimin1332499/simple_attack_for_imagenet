# 解题思路与代码说明

> 由比赛问题和评估标准，需要在给定的数据集上以有限扰动（无穷范数）完成目标攻击。针对这一问题，采用如下思路：
> 1. 训练一个对抗鲁棒模型  
> 2. 选取攻击策略 
> 3. 融合其他泛化（迁移）策略

## 解题思路

### 1. 鲁棒模型

在ImageNet(ILSVRC 2012)上使用l2度量下的PGD对抗训练ResNet-50模型，攻击参数为: $\epsilon=3, steps=7, \alpha=0.5$，训练参数为：$BatchSize=256, LR=0.1, drop by 10 at epochs \in [100]$

预训练模型: [下载](http://andrewilyas.com/ImageNet.pt) [百度云[提取码:y4ob]](https://pan.baidu.com/s/1TfB-IljEtvVSP8MQYQbgFg )

### **2. 攻击策略**

对抗样本攻击问题中，无目标攻击通常迁移能力更强，目标攻击迁移能力弱。为使攻击迁移性更好，在交叉熵损失函数基础上考虑两种方案：1. 目标类损失 2. 目标损失和非目标损失的加权和。

实验发现单纯目标损失效果已经足够好，因而采用该方案。设攻击时每次迭代对图片的梯度为$g$，则对抗样本为：
$$
\begin{aligned}
X_{i+1} = X_i - \alpha \cdot \frac{g}{mean(|g|)} \\
X_{i+1} = Clip_{X, \epsilon}{X_{i+1}}\\
\end{aligned}
$$

该方法为**团队原创**，与原始PGD方法相比，实验表明同样参数下迁移性更好。以下将两种方案简称为`mean`和`sign`。
$$
\begin{aligned}
X_{i+1} = X_i - \alpha \cdot sign(g) \\
X_{i+1} = Clip_{X, \epsilon}{X_{i+1}}\\
\end{aligned}
$$

### 3. 其他策略

同时采用了`InputDiversity`和`模型融合`方式提高攻击效果，由于时间有限只有一个对抗鲁棒模型，因而采取鲁棒模型与普通模型(VGG19)的交叉熵进行加权融合的方式。

## 实验效果

为说明方案各个部分的贡献，对不同组合的攻击方案采取`线上得分`和`线下评估`进行评估，说明方案的有效性。其中线下评估方式采用NIPS17防御比赛第二名的模型进行测试，[Inception_ResNet](https://github.com/cihangxie/NIPS2017_adv_challenge_defense)。结果如下表所示，其中成功率和错误率为线下模型上分类到指定目标的比例和分类正确的比例，VGG项值为模型融合时损失函数上的权值：

| 攻击方式 | $\epsilon$ | $\alpha$ |  n   | Input-Diverse | +VGG |        得分         |    成功率     |    正确率    |
| :--: | :--------: | :------: | :--: | :-----------: | :--: | :---------------: | :--------: | :-------: |
| mean |     32     |    1     |  60  |       -       |  -   |   2.3942/2.4087   |   70.15%   |   6.91%   |
| sign |     32     |    1     |  60  |       -       |  -   |      2.1996       |   53.27%   |  16.20%   |
| mean |     32     |    1     |  60  |       Y       |  -   |    **2.7374**     |   73.60%   |   9.7%    |
| mean |     32     |    1     |  60  |       Y       | 0.1  | **2.7453**/2.7037 |   76.81%   |   8.47%   |
| sign |     32     |    1     |  60  |       Y       | 0.1  |      2.2752       |   63.57%   |  13.82%   |
| mean |     32     |    2     |  30  |       Y       | 0.1  |   2.2703/2.2881   | **98.03%** | **0.16%** |
| sign |     32     |    2     |  30  |       Y       | 0.1  |      2.1127       |   59.13%   |  13.49%   |

对比各项数据，可知：  
1. 在其他参数相同时，**`mean`攻击方案比`sign`方案效果更好 (+0.2~0.45)，时间复杂度相同。**（使用sign方案最好得分2.2752，而使用mean方案为**2.7453**分，估计约85%的非目标成功率和35%的目标攻击成功率）
2. **`InputDiversity`有效,与`mean`方案同时使用效果提升更多(L1-L3, +0.34, L2-L5, +0.08)**。
3. 不太成功的尝试：模型融合（提升微小，时间有限缺乏足够对抗鲁棒模型）、增加攻击步长（仅线下表现好，有一定偶然性）

## 代码说明
代码组织目录如下：  
.  
├── data（输入图片目录）  
│   ├── dev.csv  
│   └── images  
├── data_attacks（生成的对抗图片目录）  
│   ├── aux_vgg1_div_32_mean  
│   └── div_32_mean  
├── models（预训练模型）  
│   └── ImageNet.pt  
├── requirements.txt  
├── src（源代码）  
│   ├── attack_target.py  
│   ├── auxiliary_attack.py  
│   ├── main.py  
│   └── utils.py  
├── 说明文档.md  
└── 团队介绍.md  
将预训练模型和输入图片放到相应目录后，运行`python main.py`即可得到两种攻击下的对抗样本，分别耗时五分钟、十分钟左右（GTX 1080Ti）

